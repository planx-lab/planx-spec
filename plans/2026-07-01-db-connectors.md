# DB Connectors — Plan 8: PostgreSQL + SQL Server Implementation

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development. Each task: strict TDD (failing test → impl → pass) + two-stage review (spec then code quality) looped to zero. The authoritative design is [`db-connectors-design.md`](../db-connectors-design.md) — read it; this plan references it for code and does not re-embed it.

**Goal:** Implement `planx-plugin-postgres` and `planx-plugin-sqlserver` — two self-describing binary connectors (Source + Sink each) — per the design doc. Unit-testable WITHOUT a real DB (via a driver seam); container e2e runs LATER once the user finishes pulling images.

**Architecture:** See design §1–§12. Load-bearing decision: the **`DBBatch` typed envelope** (`Columns []string` + `Rows []DBRow{Types []byte, Vals []string}` with Kind tags) — gob-safe, full type fidelity, symmetric across both connectors. Postgres uses pgx/v5 (`pgxpool` + `rows.Values()` + `CopyFrom`); SQL Server uses `database/sql` + `go-mssqldb` (prepared INSERT in tx). Each binary registers `dbbatch.DBBatch{}` + `dbbatch.DBRow{}` via `init(){sdk.RegisterType(...)}`.

**Tech Stack:** Go 1.25.3, `github.com/jackc/pgx/v5 v5.10.0`, `github.com/microsoft/go-mssqldb v1.10.0`, `planx-sdk-go/sdk`.

**Open questions (all defaulted per design §11 — proceeding with recommendations; user may override):** raw `query` Source config (not table+columns); copy-paste `dbbatch` per repo (not shared SDK package, v2); pg Sink `CopyFrom`-only (append-only); `pgtype.Numeric`→string; ssl `disable`/`false` default; `Validate` left nil; `_ context.Context` in Init; `batchRows` default 1000.

**Implementation order (design §12):** **Postgres first** (pgx better documented, sharpest type issues), then SQL Server mirrors the established pattern. The `dbbatch` file + Kind-tag table are defined on postgres and copied verbatim into sqlserver.

---

## File Structure (per connector — design §9)

```
planx-plugin-postgres/          planx-plugin-sqlserver/
├── cmd/plugin/main.go          ├── cmd/plugin/main.go
├── internal/source/{source.go, source_test.go, conn.go}   (querier seam)
├── internal/sink/{sink.go, sink_test.go, exec.go}         (executor seam)
├── internal/dbbatch/dbbatch.go  (BYTE-IDENTICAL between both repos)
├── go.mod  .gitignore  AI.md  repo.lock  README.md
```

---

## POSTGRES (establishes the pattern)

### Task P1: Repo bootstrap

**Files:** go.mod, AI.md, repo.lock, README.md, .gitignore (copy csv's, `repo.lock` ALLOWED = `cmd/plugin/ internal/{source,sink,dbbatch}/ go.mod`).

- [ ] Create `/Users/mw/workspace/repo/github.com/planx-lab/planx-plugin-postgres/`. `go mod init github.com/planx-lab/planx-plugin-postgres`, go 1.25.3, replace block (sdk-go + proto → ../), require `github.com/jackc/pgx/v5 v5.10.0`.
- [ ] AI.md / repo.lock / .gitignore per design §9 (mirror csv). README describes pg source (query/batchRows/sslmode) + sink (table/columns/CopyFrom).
- [ ] `git init` + baseline commit. **Do NOT push yet** (per .git-audit discipline: build green first; push at P5).
- [ ] **Verify:** `go build ./...` (empty packages OK — no source/sink yet; create empty dirs or defer). Confirmed git-init'd (audit step).

### Task P2: `internal/dbbatch/` — the envelope (TDD, gob round-trip FIRST)

**Files:** `internal/dbbatch/dbbatch.go`, `internal/dbbatch/dbbatch_test.go`.

This is the gob contract — get it green BEFORE source/sink. Design §3 has the exact types + Kind constants + encode/decode rules.

- [ ] **RED:** Write `dbbatch_test.go` — (1) compile-time: types exist; (2) **gob round-trip via the SDK codec**: `batch.NewCodec().Pack(DBBatch{...})` then `Unpack` → assert Columns/Rows/Types/Vals survive (the regression test for the gob footgun — fails loudly if RegisterType dropped); (3) `encodeRow`/`decodeValue` per Kind (int64/float64/string/bool/time.Time/[]byte/nil) — assert fidelity, esp. float64 precision (`strconv.FormatFloat 'f',-1,64`) + []byte base64 + nil→empty.
- [ ] **GREEN:** Write `dbbatch.go` per design §3: `DBBatch`, `DBRow`, Kind constants (1–7), `encodeRow(values []any) DBRow`, `decodeRowToArgs(row DBRow) []any`. Pure functions, no driver deps.
- [ ] **Verify:** `go test ./internal/dbbatch/ -v` all green. `go vet` clean.
- [ ] Commit. (Review: spec — gob round-trip genuinely exercised via SDK codec, not a fake; code quality — Kind coverage complete.)

### Task P3: Source (TDD, fake querier seam — no real DB)

**Files:** `internal/source/{source.go, source_test.go, conn.go}`. Design §4 (full ReadBatch code) + §8 (querier seam).

- [ ] **RED:** `source_test.go` per design §8 test list: (1) `var _ sdk.SourceSPI = New()`; (2) Init config-parse (valid/invalid/missing-required/defaults port+batchRows+sslmode); (3) ReadBatch via `fakeQuerier`: full batch / partial-then-EOF / empty-immediate-EOF / `rows.Err()` surfaced; (4) Close uninit + Close post-Init; (5) NULL → KindNil slot.
- [ ] **GREEN:** `conn.go` = the `querier` + `rowsIterator` seam interface (design §8). `source.go` = `Source` (cfg, querier, columns, done flag), `Init` (parse+validate+defaults+build DSN via net/url+connect+Ping+query→rows+capture columns), `ReadBatch` (two-phase EOF per design §4 code), `scanRow` (pgx `rows.Values()` → encodeRow), `Close` (rows.Close + pool.Close, idempotent). `init(){sdk.RegisterType(dbbatch.DBBatch{}); sdk.RegisterType(dbbatch.DBRow{})}`.
- [ ] **Verify:** `go test ./internal/source/ -v` green, NO real DB touched. The real pgx `querier` impl lives in `conn.go` (calls pgxpool) but tests use `fakeQuerier`.
- [ ] Commit. (Review: EOF two-phase correct; seam lets tests run DB-free; DSN builds password via url.UserPassword.)

### Task P4: Sink (TDD, fake executor seam — no real DB)

**Files:** `internal/sink/{sink.go, sink_test.go, exec.go}`. Design §5 (pg CopyFrom code) + §8.

- [ ] **RED:** `sink_test.go`: (1) SPI conformance; (2) Init config-parse + required; (3) WriteBatch via `fakeExecutor`: empty no-op / type-assertion error (`expected dbbatch.DBBatch, got %T`) / columns-override / NULL→nil arg; (4) Close uninit + post-Init.
- [ ] **GREEN:** `exec.go` = narrow `copyFromPool` seam (just `CopyFrom`). `sink.go` = `Sink`, `Init` (connect+Ping), `WriteBatch` (type-assert DBBatch → build `copySource` impl `pgx.CopyFromSource` → `pool.CopyFrom`, columns override, decode Vals per Kind), `Close`. `init(){RegisterType×2}`.
- [ ] **Verify:** `go test ./internal/sink/ -v` green, no real DB.
- [ ] Commit. (Review: CopyFrom path; decodeRowToArgs typed correctly; empty batch no-op.)

### Task P5: main.go + whole-module + push

**Files:** `cmd/plugin/main.go` (design §7 postgres-source + postgres-sink ConfigSchema).

- [ ] Write main.go: `sdk.Serve(sdk.Plugin{ID:"postgres", Components:[{ID:"source", Kind:KindSource, Source:source.New, ConfigSchema: pgSourceSchema}, {ID:"sink", Kind:KindSink, Sink:sink.New, ConfigSchema: pgSinkSchema}]})`. ConfigSchemas per design §7 (host/port/database/user/SECRET password/query or table/columns/batchRows/sslmode enum).
- [ ] `go mod tidy` + `go build -o plugin ./cmd/plugin`. **Handshake smoke** (`./plugin & sleep 1; kill` → JSON handshake line). `go test ./...` + `go vet ./...` green.
- [ ] **Audit:** `.gitignore` has `/plugin` + `planx.handshake` (cmd/plugin/main.go tracked — NOT ignored — the csv lesson). `git ls-files cmd/plugin/main.go` non-empty.
- [ ] `gh repo create planx-lab/planx-plugin-postgres --public --source=. --push`. (Per .git-audit discipline: git-init'd from P1, now create remote + push.)
- [ ] Commit. (Review: ConfigSchema matches design §7; SECRET on password; no tracked artifacts.)

---

## SQL SERVER (mirrors postgres; dbbatch copied verbatim)

### Task S1: Repo bootstrap (as P1; driver = `github.com/microsoft/go-mssqldb v1.10.0`; repo.lock ALLOWED includes `dbbatch`).

### Task S2: `internal/dbbatch/dbbatch.go` — **copy verbatim from postgres P2** (byte-identical; design §3 mandates this for cross-connector gob interop). Copy the test too; both green.

### Task S3: Source (TDD, fake querier seam). Design §4 + §8. Differs from pg: `database/sql` rows (`rows.ColumnTypes` → `reflect.New(ct.ScanType())` Scan into ptrs → deref → encodeRow), DSN `sqlserver://user:pass@host:port?database=&encrypt=` via `url.UserPassword`, `encrypt` enum (true/false) not sslmode.

### Task S4: Sink (TDD, fake executor seam). Design §5 mssql path: prepared INSERT in tx, `@p1,@p2,...` placeholders, `decodeRowToArgs` per Kind, batched per `batchRows` (config). NOT CopyFrom (mssql has no clean equivalent; design §5 rationale).

### Task S5: main.go + whole-module + push (as P5; ConfigSchemas per design §7 sqlserver-source/sink; defaults port 1433, encrypt false).

---

## Task E2E (LATER — container, after user pulls images)

Deferred. Design §8 container e2e matrix: round-trip across SQL types, NULL preservation, empty→SUCCEEDED, 10k-row throughput, CopyFrom(pg) vs INSERT(mssql). Harness: Apple containers (pg + mssql images), real engine DAG source→sink, assert via direct driver connection. Run after both connectors pass P1–P5 / S1–S5.

---

## Self-Review notes

- **Spec coverage:** design §3 (dbbatch) → P2/S2; §4 (source) → P3/S3; §5 (sink) → P4/S4; §7 (ConfigSchema) → P5/S5; §8 (testing seam) → P3/P4/S3/S4; §6 (SECRET) → P5/S5. §11 open Qs all defaulted (noted above).
- **Compile boundaries:** P2 (dbbatch, no driver) → P3 (source, needs dbbatch+seam) → P4 (sink, needs dbbatch+seam) → P5 (main.go wires both). Each task's package builds+tests alone.
- **The csv lessons applied:** (1) `init(){RegisterType}` for the envelope types — tested by the gob round-trip in P2; (2) `.gitignore` `/plugin` anchored so `cmd/plugin/main.go` is tracked (P5 audit step); (3) two-phase EOF (partial batch then EOF, never drop rows).
- **No real DB until E2E:** the querier/executor seams (design §8) make P3/P4/S3/S4 fully unit-testable now, while images pull.
