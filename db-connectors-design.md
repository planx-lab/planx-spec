# DB Connectors Design — PostgreSQL + SQL Server

**Status:** Draft for implementation
**Date:** 2026-07-01
**Source:** design Workflow wf_aa1cef64-967 (6 agents, 660k tokens)

---

## 1. Overview & Scope

Two new Connector repos, following the proven `planx-plugin-csv` multi-component pattern exactly:

| Connector | Components | Driver |
|-----------|-----------|--------|
| `planx-plugin-postgres` | `postgres-source` + `postgres-sink` (one binary) | pgx/v5 |
| `planx-plugin-sqlserver` | `sqlserver-source` + `sqlserver-sink` (one binary) | go-mssqldb |

Each is a **single self-describing binary** registered via `sdk.Serve` with two `ComponentSpec`s (ADR-008/009). No `manifest.yaml`. The binary describes itself via Discover.

**Out of scope (v1):** CDC/incremental watermarks, upsert/`ON CONFLICT`, transaction semantics beyond per-batch commit, SSL/TLS negotiation beyond a single `sslmode`/`encrypt` enum, schema migration. These are flagged in §11 as open questions for v2.

**Testing posture:** Unit tests run **without a real DB** (TDD via a `db` seam interface + fake). Container e2e against real Postgres and SQL Server happens **later**, after the user finishes pulling the Apple-container images. The unit-test seam must be designed first so implementation can proceed immediately.

---

## 2. Driver Decisions

| Connector | Import path | Version | Rationale |
|-----------|-------------|---------|-----------|
| Postgres | `github.com/jackc/pgx/v5` (+ `pgxpool`) | **v5.10.0** | Latest stable; v5.9.0 fixed a memory-safety CVE; sub-packages (`pgconn`, `pgtype`, `pgxpool`) are all in the same module. `pgxpool` is concurrency-safe (a single `*pgx.Conn` is not). |
| SQL Server | `github.com/microsoft/go-mssqldb` | **v1.10.0** | Official Microsoft driver; pure Go (no CGO); `database/sql`-compatible; registered under driver name `"sqlserver"` (the `"mssql"` alias is deprecated and has unfixed bugs). |

Both are **normal versioned `require` lines** in `go.mod` (NOT `replace`). Pin known-good versions compatible with `go 1.25.3`.

**Connection strategy differs by driver:**
- Postgres uses the **pgx native API** (`pgxpool.Pool`) — gives access to `CopyFrom` (binary COPY protocol) for fast bulk insert and `rows.Values()` for heterogeneous column scan. Not `database/sql` + lib/pq.
- SQL Server uses **`database/sql`** with `sql.Open("sqlserver", dsn)` — mssqldb is a database/sql driver; no native bulk API worth the complexity (see §5).

---

## 3. THE Batch Payload Decision (the load-bearing choice)

This is the single most important design choice and the one place the research disagreed. Every other decision flows from it.

### Constraint recap (verified against the live codebase)

The SDK batch codec is gob-through-interface:
```go
// planx-sdk-go/internal/batch/codec.go:34-44
type batchWrapper struct { Batch any }
func (c *gobCodec) Pack(b any) (PackedBatch, error) {
    var buf bytes.Buffer
    err := gob.NewEncoder(&buf).Encode(batchWrapper{Batch: b})
    return buf.Bytes(), err
}
```
Because `Batch` is `any`, **every concrete type that flows through it must be `gob.Register`-ed on BOTH the source process and the sink process** (they are separate OS processes with independent per-process gob registries). The SDK pre-registers only `map[string]string{}` (codec.go:9-10). `sdk.RegisterType` is a thin pass-through to `gob.Register`.

A naive `[]map[string]any` from `rows.Scan`/`rows.Values()` **fails the gob round-trip** because `time.Time`, `[]any`, `map[string]any`, etc. are not registered. SQL Server's `DECIMAL`/`UNIQUEIDENTIFIER` arrive as `string`; pgx's `time.Time` is unregistered; `pgtype.Numeric` is an unregistered struct. This is the highest-impact footgun.

### Rejected alternatives

| Option | Verdict | Why |
|--------|---------|-----|
| `[]map[string]any` (naive scan) | Rejected | Requires registering `map[string]any` + every value type on both processes; fragile cross-connector coupling; `pgtype.Numeric` breaks it silently. |
| `[]Row{ Values []any }` (typed struct, any leaves) | Rejected | Still has the leaf-registration problem; `[]any` must be registered plus every concrete element type. |
| `[][]string` (CSV-style, all coerced to string) | Rejected for DB | Loses type fidelity — Sink cannot distinguish int from string-int, cannot format `time.Time` correctly, cannot pass binary as bytes. Fine for CSV; not for DB replication. |
| `DBBatch` envelope in a shared SDK package | Rejected for v1 | Adds a new exported type to the SDK surface; cross-repo governance overhead. Defer to v2 (see §11). |

### Chosen payload: per-connector `DBBatch` typed envelope (copy-pasted identically in each repo)

Each connector repo defines **its own byte-identical copy** of the envelope below. gob matches by type name AND wire structure, so the two repos must keep field names, types, and order identical for cross-connector pipelines (e.g. `postgres-source → sqlserver-sink`) to work.

```go
// internal/dbbatch/dbbatch.go (one file, copied identically into both repos)

package dbbatch

// DBBatch is the gob-registered batch payload for DB row data. Carries the
// column schema once at the top plus per-row type-tagged string slots.
type DBBatch struct {
    Columns []string // ordered column names (schema for every row in this batch)
    Rows    []DBRow
}

// DBRow is one row. Types and Vals are parallel to DBBatch.Columns.
type DBRow struct {
    Types []byte   // per-column kind tag (see Kind* constants)
    Vals  []string // per-column string-encoded value
}

// Kind tags. Stable across repos. Single byte, parallel to Columns.
const (
    KindInt    byte = 1 // int64        -> Vals: strconv.FormatInt(v, 10)
    KindFloat  byte = 2 // float64      -> Vals: strconv.FormatFloat(v, 'f', -1, 64)
    KindString byte = 3 // string       -> Vals: as-is
    KindBool   byte = 4 // bool         -> Vals: strconv.FormatBool(v)
    KindTime   byte = 5 // time.Time    -> Vals: t.Format(time.RFC3339Nano)
    KindBytes  byte = 6 // []byte       -> Vals: base64.StdEncoding.EncodeToString(v)
    KindNil    byte = 7 // SQL NULL     -> Vals: "" (value ignored on decode)
)
```

### Registration (call in BOTH source and sink `init()`)

```go
// in internal/source/source.go and internal/sink/sink.go
func init() {
    sdk.RegisterType(dbbatch.DBBatch{})
    sdk.RegisterType(dbbatch.DBRow{})
}
```

Exactly **two `RegisterType` calls per binary**, identical on both ends. The top-level type is concrete (not `any`), so no per-element-type registration explosion.

### Rationale

1. **gob-safe** — top-level concrete struct; no `any` leaves in the wire payload.
2. **Full type fidelity** — Sink reads the kind tag and issues a correctly-typed INSERT param (`int64`, `time.Time`, `[]byte`, etc.) rather than guessing from a string.
3. **Symmetric across connectors** — `postgres-source` and `sqlserver-sink` produce/consume the identical `DBBatch` wire format, so heterogeneous pipelines work.
4. **NULL survives cleanly** as `KindNil` with empty `Vals` (no gob nil-edge-case fragility, no `empty-string-vs-NULL` ambiguity that plagues `[][]string`).
5. **Compact** — parallel string slices; payload ≈5KB/100rows×5cols (vs 11.5KB for `[]map[string]any`).
6. **Matches codebase density** — the envelope + encode/decode helpers are ~80 lines per connector.

**Precision note:** `float64` uses `strconv.FormatFloat(v, 'f', -1, 64)` (shortest round-trip) so no precision loss. SQL Server `DECIMAL`/`MONEY` arrive as `string` from the driver → encoded as `KindString`, round-trips with full precision. pgx `pgtype.Numeric` must be coerced to `string` at scan time (or registered with gob) — see §11 open question.

---

## 4. Source Component Design

Both connectors share the same Source shape; only driver calls and DSN construction differ.

### ConfigSchema fields (Source)

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `host` | STRING | yes | — | DB host |
| `port` | INTEGER | no | 5432 (pg) / 1433 (mssql) | DB port |
| `database` | STRING | yes | — | Database name |
| `user` | STRING | yes | — | DB user |
| `password` | SECRET | yes | — | DB password (masked, never logged) |
| `query` | STRING | yes | — | Full SELECT query (e.g. `SELECT * FROM users`) |
| `batchRows` | INTEGER | no | 1000 | Rows per batch |
| `sslmode` / `encrypt` | ENUM | no | `disable` (pg) / `false` (mssql) | Connection security |

**Query strategy (decisive recommendation): v1 uses a raw configurable `query` string, NOT `table+columns`.** Rationale: a raw query is simpler, more flexible (WHERE, JOIN, projection), matches how DBAs think, and avoids the schema-introspection machinery a `table+columns` mode would require. The user is responsible for writing a finite SELECT (no `FOR UPDATE`, no cursors). The Source executes it verbatim. **Flag for human confirmation** in §11 — a future `table+columns` mode is a clean v2 addition.

### Init

1. `json.Unmarshal(cfg, &s.cfg)` → validate required fields → apply defaults (`port`, `batchRows`, `sslmode`).
2. Build driver DSN (see §7 for per-driver construction).
3. Open connection (pgxpool for pg; `sql.Open` for mssql) → `Ping()` to validate connectivity → store handle.
4. Issue the SELECT query → obtain `rows` cursor → store.
5. Capture column names from `rows.FieldDescriptions()` (pg) / `rows.Columns()` (mssql) once.
6. Any error closes the connection and returns `<connector> source: <what>: %w`.

### ReadBatch — two-phase EOF (mandatory)

Follows the csv canonical pattern exactly:

```go
func (s *Source) ReadBatch() (sdk.Batch, error) {
    batch := make([]dbbatch.DBRow, 0, s.cfg.BatchRows)
    for len(batch) < s.cfg.BatchRows {
        if !s.rows.Next() {          // pgx: rows.Next(); mssql: rows.Next()
            s.done = true
            break
        }
        row, err := s.scanRow()      // driver-specific; returns DBRow with Types+Vals
        if err != nil {
            return nil, fmt.Errorf("<connector> source: scan: %w", err)
        }
        batch = append(batch, row)
    }
    if s.done && len(batch) == 0 {
        return nil, io.EOF           // clean stream end -> DAG reaches SUCCEEDED
    }
    return dbbatch.DBBatch{Columns: s.columns, Rows: batch}, nil
}
```

After the loop, **always check `rows.Err()`** (both drivers surface delayed errors there, not from `Next()`).

- **Partial trailing batch returned first (nil error), then `io.EOF` on the next call** — getting this wrong either hangs the DAG (`wg.Wait` never returns) or silently drops the final rows.
- Returning `io.EOF` while a non-empty batch is buffered drops data. Forbidden.

### scanRow (driver-specific encode into DBBatch)

**Postgres (pgx):**
```go
vals, err := s.rows.Values()  // []any, pgx decodes via its type registry
// then for each val, switch on concrete type -> Kind tag + string slot
```
**SQL Server (database/sql):**
```go
cols := s.colTypes                       // []sql.ColumnType from rows.ColumnTypes()
ptrs := make([]any, len(cols))
for i, ct := range cols {
    ptrs[i] = reflect.New(ct.ScanType()).Interface()
}
s.rows.Scan(ptrs...)
// then deref each pointer, switch on concrete type -> Kind tag + string slot
```

Encode rules (shared, identical in both repos):
| Go type | Kind | Vals encoding |
|---------|------|---------------|
| `int64` | `KindInt` | `strconv.FormatInt(v, 10)` |
| `float64` | `KindFloat` | `strconv.FormatFloat(v, 'f', -1, 64)` |
| `string` | `KindString` | as-is |
| `bool` | `KindBool` | `strconv.FormatBool(v)` |
| `time.Time` | `KindTime` | `v.Format(time.RFC3339Nano)` |
| `[]byte` | `KindBytes` | `base64.StdEncoding.EncodeToString(v)` |
| `nil` | `KindNil` | `""` |

**Special case — pgx `pgtype.Numeric`:** coerce to `string` (via its `Value()` method or `.Int`/`.Exp`) at scan time and emit `KindString`. Registering the struct with gob is rejected (cross-version fragility). **Flag for human confirmation** — see §11.

### Close

1. `rows.Close()` first.
2. Close the connection (`pool.Close()` for pgxpool; `db.Close()` for mssql).
3. Idempotent: `Close` on an uninit Source returns `nil` (matches csv).

---

## 5. Sink Component Design

### ConfigSchema fields (Sink)

| Field | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `host` | STRING | yes | — | DB host |
| `port` | INTEGER | no | 5432 / 1433 | DB port |
| `database` | STRING | yes | — | Database name |
| `user` | STRING | yes | — | DB user |
| `password` | SECRET | yes | — | DB password |
| `table` | STRING | yes | — | Target table (e.g. `users` or `schema.users`) |
| `columns` | STRING | no | — | Comma-separated column list for INSERT. If empty, Sink derives from `DBBatch.Columns`. |
| `batchRows` (mssql only) | INTEGER | no | 1000 | Rows per INSERT transaction (pg uses CopyFrom) |
| `sslmode` / `encrypt` | ENUM | no | driver default | Connection security |

### Init

1. Parse + validate config (same shape as Source).
2. Build DSN, open connection, `Ping()`.
3. Store the connection handle. No prepare step for pg (CopyFrom is per-batch). mssql optionally prepares the INSERT statement on first `WriteBatch`.

### WriteBatch — insert strategy differs by driver (decisive recommendation)

**Postgres Sink: `CopyFrom` (binary COPY protocol) as the default.**

Rationale (from Agent 3): CopyFrom is documented as "faster than an insert with as few as 5 rows" and is the idiomatic bulk-load path for pgx. The `DBBatch.Rows [][]string` must be converted to `[][]any` for `pgx.CopyFromRows`, decoding each `Vals` slot back to its typed Go value per the `Types` tag.

```go
func (s *Sink) WriteBatch(batch sdk.Batch) error {
    dbb, ok := batch.(dbbatch.DBBatch)
    if !ok {
        return fmt.Errorf("postgres sink: expected dbbatch.DBBatch, got %T", batch)
    }
    if len(dbb.Rows) == 0 { return nil }
    cols := dbb.Columns
    if len(s.cfg.Columns) > 0 { cols = parseColumns(s.cfg.Columns) }
    src := &copySource{batch: dbb, cols: cols}  // implements pgx.CopyFromSource
    _, err := s.pool.CopyFrom(ctx, pgx.Identifier{s.cfg.Table}, cols, src)
    if err != nil { return fmt.Errorf("postgres sink: copy: %w", err) }
    return nil
}
```

**Why CopyFrom over batched INSERT for pg:** purpose-built for bulk insert, benchmarks consistently favor it even at small row counts, and `pgx.CopyFromRows` accepts the typed `[][]any` directly. **Upsert/`ON CONFLICT` is NOT supported by CopyFrom** — v1 is append-only. The staging-table pattern for idempotency is a documented v2 path (§11). **Flag for human confirmation.**

**SQL Server Sink: prepared INSERT inside a transaction, batched.**

Rationale (from Agent 4): `mssql.CopyIn`/BulkCopy forces importing the `mssql` subpackage, requires a fixed known column list, and breaks Always Encrypted. The prepared-INSERT-in-tx path is pure `database/sql`, flexible, and adequate for plugin throughput.

```go
func (s *Sink) WriteBatch(batch sdk.Batch) error {
    dbb, ok := batch.(dbbatch.DBBatch)
    if !ok {
        return fmt.Errorf("sqlserver sink: expected dbbatch.DBBatch, got %T", batch)
    }
    if len(dbb.Rows) == 0 { return nil }
    cols := dbb.Columns
    if len(s.cfg.Columns) > 0 { cols = parseColumns(s.cfg.Columns) }
    placeholders := mssqlPlaceholders(len(cols))  // @p1,@p2,...
    stmt := fmt.Sprintf("INSERT INTO %s (%s) VALUES (%s)",
        s.cfg.Table, strings.Join(cols, ","), placeholders)
    tx, err := s.db.BeginTx(ctx, nil)
    if err != nil { return fmt.Errorf("sqlserver sink: begin: %w", err) }
    prepared, err := tx.PrepareContext(ctx, stmt)
    if err != nil { /* rollback */ return ... }
    for _, row := range dbb.Rows {
        args := decodeRowToArgs(row, cols)  // []any typed per Kind tag
        if _, err := prepared.ExecContext(ctx, args...); err != nil { /* rollback */ return ... }
    }
    prepared.Close()
    return tx.Commit()
}
```

**`decodeRowToArgs`** reads the `Types` byte and converts each `Vals` string back to the typed Go value: `KindInt`→`int64`, `KindTime`→`time.Time`, `KindBytes`→`[]byte` base64-decoded, `KindNil`→`nil`, etc. This is where type fidelity pays off — the INSERT params are correctly typed.

### Close

Close the connection (`pool.Close()` / `db.Close()`). Idempotent.

---

## 6. SECRET Password Handling

Confirmed against `planx-sdk-go/sdk/schema.go:51-54`: `SecretField` is identical in signature to other field constructors; it sets `FieldType_FIELD_TYPE_SECRET`. The proto `ConfigValue` oneof (schema.proto:52-60) has only `string_value`/`int_value`/`double_value`/`bool_value` — **there is no `SecretValue` wire kind**.

Therefore:
- **ConfigSchema:** declared with `sdk.SecretField("password", sdk.Required(), sdk.WithDescription("DB password"))` — purely a UI-masking + never-logged hint for the Designer/SDK.
- **Plugin Config struct:** `Password string \`json:"password"\`` — the value arrives as a **plain JSON string** inside the Init `config []byte`. Parses normally.
- **Engine never logs it** — the SDK masks SECRET fields; the plugin honors this by **never `fmt.Printf`-ing the config or DSN**.
- **DSN construction:** the password is interpolated into the connection string inline (both pgx and mssqldb require it in the DSN; there is no separate setter). Use `net/url` with `url.UserPassword(user, password)` so it is URL-encoded safely and any special characters are handled.

---

## 7. ConfigSchema (Alpha types) — 4 schemas total

Verified field-builder signatures against `planx-sdk-go/sdk/schema.go`.

### postgres-source

```go
sdk.Schema(
    sdk.StringField("host", sdk.Required(), sdk.WithDescription("PostgreSQL host")),
    sdk.IntegerField("port", sdk.WithDefault(sdk.IntValue(5432)), sdk.WithDescription("PostgreSQL port")),
    sdk.StringField("database", sdk.Required(), sdk.WithDescription("Database name")),
    sdk.StringField("user", sdk.Required(), sdk.WithDescription("DB user")),
    sdk.SecretField("password", sdk.Required(), sdk.WithDescription("DB password")),
    sdk.StringField("query", sdk.Required(), sdk.WithExample("SELECT * FROM users"), sdk.WithDescription("SELECT query (finite result set)")),
    sdk.IntegerField("batchRows", sdk.WithDefault(sdk.IntValue(1000)), sdk.WithDescription("Rows per batch")),
    sdk.EnumField("sslmode", []string{"disable", "require", "verify-ca", "verify-full"}, sdk.WithDefault(sdk.StringValue("disable")), sdk.WithDescription("SSL mode")),
)
```

DSN: `fmt.Sprintf("postgres://%s:%s@%s:%d/%s?sslmode=%s", user, url.QueryEscape(password), host, port, database, sslmode)` — built via `net/url` for safe password encoding.

### postgres-sink

```go
sdk.Schema(
    sdk.StringField("host", sdk.Required()),
    sdk.IntegerField("port", sdk.WithDefault(sdk.IntValue(5432))),
    sdk.StringField("database", sdk.Required()),
    sdk.StringField("user", sdk.Required()),
    sdk.SecretField("password", sdk.Required(), sdk.WithDescription("DB password")),
    sdk.StringField("table", sdk.Required(), sdk.WithDescription("Target table (e.g. users or public.users)")),
    sdk.StringField("columns", sdk.WithDescription("Comma-separated column list; if empty, uses batch column schema")),
    sdk.EnumField("sslmode", []string{"disable", "require", "verify-ca", "verify-full"}, sdk.WithDefault(sdk.StringValue("disable"))),
)
```

### sqlserver-source

```go
sdk.Schema(
    sdk.StringField("host", sdk.Required(), sdk.WithDescription("SQL Server host")),
    sdk.IntegerField("port", sdk.WithDefault(sdk.IntValue(1433)), sdk.WithDescription("SQL Server port")),
    sdk.StringField("database", sdk.Required(), sdk.WithDescription("Database name")),
    sdk.StringField("user", sdk.Required(), sdk.WithDescription("DB user")),
    sdk.SecretField("password", sdk.Required(), sdk.WithDescription("DB password")),
    sdk.StringField("query", sdk.Required(), sdk.WithExample("SELECT * FROM dbo.users"), sdk.WithDescription("SELECT query (finite result set)")),
    sdk.IntegerField("batchRows", sdk.WithDefault(sdk.IntValue(1000)), sdk.WithDescription("Rows per batch")),
    sdk.EnumField("encrypt", []string{"true", "false"}, sdk.WithDefault(sdk.StringValue("false")), sdk.WithDescription("Encrypt connection (TLS)")),
)
```

DSN: built via `net/url` as `sqlserver://user:password@host:port?database=...&encrypt=...` — `url.UserPassword` handles password encoding.

### sqlserver-sink

```go
sdk.Schema(
    sdk.StringField("host", sdk.Required()),
    sdk.IntegerField("port", sdk.WithDefault(sdk.IntValue(1433))),
    sdk.StringField("database", sdk.Required()),
    sdk.StringField("user", sdk.Required()),
    sdk.SecretField("password", sdk.Required(), sdk.WithDescription("DB password")),
    sdk.StringField("table", sdk.Required(), sdk.WithDescription("Target table (e.g. dbo.users)")),
    sdk.StringField("columns", sdk.WithDescription("Comma-separated column list; if empty, uses batch column schema")),
    sdk.IntegerField("batchRows", sdk.WithDefault(sdk.IntValue(1000)), sdk.WithDescription("Rows per INSERT transaction")),
    sdk.EnumField("encrypt", []string{"true", "false"}, sdk.WithDefault(sdk.StringValue("false"))),
)
```

---

## 8. Testing Strategy

**Two tiers:** unit (no DB, TDD, fast — runs on every change) and container e2e (real DB, later, after user pulls images).

### The DB seam interface (unit-test enabler)

Both Source and Sink talk to the DB through a small interface so tests inject a fake. This is the single most important testability decision.

```go
// internal/source/conn.go (postgres); mirrored in sqlserver
type querier interface {
    Query(ctx context.Context, sql string, args ...any) (rowsIterator, error)
    Close()
}

type rowsIterator interface {
    Next() bool
    Columns() []string                  // pg: from FieldDescriptions; mssql: rows.Columns
    ScanValues() ([]any, error)         // pg: rows.Values(); mssql: reflect-Scan into ptrs
    Err() error
    Close()
}
```

The production `Source` holds a `querier`. `Init` builds a real pgxpool/sql.DB-backed `querier`; tests inject a `fakeQuerier` that yields canned rows. The Sink gets an analogous `executor` seam (`BeginTx`/`Prepare`/`Exec`/`Commit`, or `CopyFrom` for pg).

**NOTE:** For the mssql Sink the seam is the `*sql.DB` interface subset (`BeginTx`, `Close`). For the pg Sink the seam is `pgxpool.Pool`'s `CopyFrom` method — define a narrow interface around just that method.

### Unit tests cover (TDD, no DB)

Mirrors the csv test conventions:
1. **Compile-time SPI conformance:** `var _ sdk.SourceSPI = source.New()` and `var _ sdk.SinkSPI = sink.New()`.
2. **`dbbatch.DBBatch` gob round-trip:** encode via the SDK's codec (`internal/batch.NewCodec().Pack/Unpack`), assert `Columns`/`Rows`/`Types`/`Vals` survive intact. This is the regression test for the gob footgun — it fails loudly if `RegisterType` is ever dropped.
3. **`encodeRow` / `decodeRowToArgs` type fidelity:** for every Kind tag — `int64`, `float64`, `string`, `bool`, `time.Time`, `[]byte`, `nil`. Especially `float64` round-trip precision and `[]byte` base64.
4. **Init config-parse:** valid JSON, invalid JSON, missing required fields (each returns the correct `<connector> <component>: ...` error), defaults applied (`port`, `batchRows`, `sslmode`).
5. **Source ReadBatch via fake querier:** (a) full batch, (b) partial trailing batch then `io.EOF`, (c) empty result → immediate `io.EOF`, (d) `rows.Err()` surfaced as error.
6. **Sink WriteBatch via fake executor:** (a) empty batch is a no-op, (b) type-assertion failure returns clear error (`expected dbbatch.DBBatch, got %T`), (c) columns override from config takes precedence over batch columns.
7. **Close:** uninit Source/Sink `Close()` returns nil; post-Init `Close()` calls through to the seam and returns its error.
8. **NULL round-trip:** `KindNil` slot → Sink emits `nil` arg, not empty string.

### Container e2e covers (later, against real DBs)

| Test | Postgres | SQL Server |
|------|----------|------------|
| Round-trip: create table → Source SELECT → Sink INSERT into second table → assert row equality | yes | yes |
| Type fidelity across all SQL types (`int`, `bigint`, `numeric`, `text`, `varchar`, `bytea`/`varbinary`, `timestamp`/`datetime2`, `bool`/`bit`) | yes | yes |
| NULL preservation | yes | yes |
| Empty result set → DAG reaches SUCCEEDED | yes | yes |
| Large batch (10k rows) — throughput sanity | yes | yes |
| CopyFrom path (pg) vs INSERT path (mssql) at scale | yes | n/a |
| SSL/encrypt toggle | optional | optional |

E2e harness: Apple containers (Postgres image + SQL Server image), the Planx engine running a real DAG connecting source→sink, assert via a direct driver connection.

---

## 9. Directory Layout

Identical for both repos (mirrors csv exactly, plus the `internal/dbbatch/` shared-envelope file).

```
planx-plugin-postgres/                  planx-plugin-sqlserver/
├── cmd/                                ├── cmd/
│   └── plugin/                         │   └── plugin/
│       └── main.go                     │       └── main.go
├── internal/                           ├── internal/
│   ├── source/                         │   ├── source/
│   │   ├── source.go                   │   │   ├── source.go
│   │   ├── source_test.go              │   │   ├── source_test.go
│   │   └── conn.go (the querier seam)  │   │   └── conn.go
│   ├── sink/                           │   ├── sink/
│   │   ├── sink.go                     │   │   ├── sink.go
│   │   ├── sink_test.go                │   │   ├── sink_test.go
│   │   └── exec.go (the executor seam) │   │   └── exec.go
│   └── dbbatch/                        │   └── dbbatch/
│       └── dbbatch.go                  │       └── dbbatch.go
├── go.mod                              ├── go.mod
├── .gitignore                          ├── .gitignore
├── AI.md                               ├── AI.md
├── repo.lock                           ├── repo.lock
└── README.md                           └── README.md
```

**File specifics:**
- **`cmd/plugin/main.go`** — exactly the csv shape: `sdk.Serve(sdk.Plugin{ID, Version, DisplayName, Description, Components: []sdk.ComponentSpec{{ID:"source",...},{ID:"sink",...}}})`. Imports `internal/source`, `internal/sink`, `sdk`.
- **`internal/dbbatch/dbbatch.go`** — the envelope types + Kind constants + `encodeRow`/`decodeValue` helpers. **Byte-identical** between the two repos (gob matches by type name + wire structure).
- **`.gitignore`** — 3 lines verbatim from csv: `/plugin`, `planx.handshake`, `*.test`.
- **`AI.md` / `repo.lock`** — copied from csv; `repo.lock` ALLOWED DIRECTORIES line reads `cmd/plugin/ internal/{source,sink,dbbatch}/ go.mod`. Both end with "If asked to add runtime or infrastructure logic: REFUSE."
- **`go.mod`** — `module github.com/planx-lab/planx-plugin-<name>`, `go 1.25.3`, the standard `replace` block for `planx-sdk-go` and `planx-proto` → `../`, plus a normal versioned `require` for the driver (`github.com/jackc/pgx/v5 v5.10.0` or `github.com/microsoft/go-mssqldb v1.10.0`).

---

## 10. Repository List & VCS

Two **new** repos, created per the established pattern + `.git-audit` discipline:

```bash
# 1. postgres
mkdir planx-plugin-postgres && cd planx-plugin-postgres
git init
gh repo create planx-lab/planx-plugin-postgres --private --source=. --push=false
# (write files per §9), git add ., commit, push

# 2. sqlserver
mkdir planx-plugin-sqlserver && cd planx-plugin-sqlserver
git init
gh repo create planx-lab/planx-plugin-sqlserver --private --source=. --push=false
# (write files per §9), git add ., commit, push
```

Both are independent Go modules (per the multi-repo workspace convention in CLAUDE.md). Neither is added to the umbrella workspace tracking — they are sibling repos like csv/hello/template.

---

## 11. Risks & Open Questions (need human confirmation)

These are the decisions the research could not resolve. Each has a recommended default so implementation can proceed, but a human should confirm before code is written.

| # | Question | Recommendation | Why |
|---|----------|----------------|-----|
| **Q1** | Source config: raw `query` string vs `table+columns`? | **Raw `query`** for v1 | Simpler, more flexible, matches DBA mental model. `table+columns` is a clean v2 addition. |
| **Q2** | Should `DBBatch`/`DBRow` live in a shared SDK package (`planx-sdk-go/sdk/dbtypes`) instead of copy-pasted per repo? | **Copy-paste per repo for v1** | Avoids expanding the SDK surface and cross-repo governance. The byte-identical-copy risk is real but bounded (one file, one doc comment). Defer to v2 once the contract stabilizes. **Strongest candidate for human override** — a shared package is the technically correct long-term answer. |
| **Q3** | Postgres Sink: `CopyFrom` (append-only) vs INSERT/upsert? | **CopyFrom-only for v1** | Idiomatic, fast, simple. Upsert via staging table is a documented v2 path. Means v1 pg Sink cannot do idempotent replays. |
| **Q4** | pgx `pgtype.Numeric`: coerce to string at scan, register with gob, or decode as float64? | **Coerce to string at scan** | Preserves precision, no gob registration, no cross-version struct fragility. Emit `KindString`. |
| **Q5** | `sslmode`/`encrypt` default: `disable`/`false` (dev-friendly) or `require`/`true` (secure-by-default)? | **Default `disable`/`false`, ENUM-configurable** | Matches dev ergonomics; production users set it explicitly. Matches csv/hello having no security model. |
| **Q6** | Should the Source implement `Validate` (ComponentSpec) for a real Designer connectivity check? | **Leave `nil` for v1** | csv/template/hello all leave it nil (schema-only). A real `Validate` (open+ping) is a nice v2 UX win but diverges from the canonical pattern. |
| **Q7** | Init `context.Context`: honor it for cancellation, or keep `_ context.Context` like csv? | **Keep `_` for v1** | No evidence the SDK propagates a meaningful deadline into Init; deviating risks inconsistency. Surface as a follow-up if cancellation is needed. |
| **Q8** | Batch size default (`batchRows`)? | **1000** | Aligns with Agent 3's CopyFrom efficiency guidance and Agent 4's INSERT-throughput guidance. Tunable. |

**Risks (not questions — these will bite if ignored):**

- **gob registration symmetry:** if a future sink forgets `sdk.RegisterType(dbbatch.DBBatch{})` in `init()`, decode panics at runtime with no useful message. Mitigation: the gob round-trip unit test (§8 test 2) and a defensive `batch.(dbbatch.DBBatch)` assertion in every Sink's WriteBatch that surfaces a clear error.
- **`DBBatch`/`DBRow` byte-identical copy fragility:** a field rename or reorder in one repo breaks cross-connector interop silently (gob matches by name + wire). Mitigation: the e2e round-trip test (§8) and a doc comment on the type.
- **`time.Time` behind `any` (pgx):** unregistered → encode fails. Mitigated by the envelope (no `any` leaves on the wire) but worth a comment in the source's scan code so a future maintainer doesn't "simplify" it back to `[]any`.
- **CopyFrom requires binary-format-capable pgtype for every column:** almost all builtin Postgres types qualify; custom ENUMs/domains need `Conn.LoadType` + `Map.RegisterType` or CopyFrom errors. v1 documents "builtin types only."
- **Driver version vs Go toolchain:** pin pgx v5.10.0 and mssqldb v1.10.0; both are pure Go (no CGO) and compatible with `go 1.25.3`. A driver requiring a newer Go breaks the build.
- **PgBouncer transaction-mode pooling** is incompatible with pgx's prepared-statement cache. Surface a `statement_cache: false` config option in v2 if users hit this.

---

## 12. Implementation Ordering

**Postgres first.** Rationale: pgx is better documented, `rows.Values()` + `CopyFrom` give the cleanest read/write path, and the gob/type-fidelity issues are sharpest there (time.Time, pgtype.Numeric). Solving them on pg first means the mssql connector inherits the answers.

### Per-connector task breakdown

```
1. Bootstrap repo + go.mod + AI.md + repo.lock + .gitignore
   → verify: go build ./... succeeds with empty source/sink
2. internal/dbbatch/dbbatch.go + unit tests (gob round-trip via SDK codec, encodeRow/decodeValue per Kind)
   → verify: go test ./internal/dbbatch/ — gob round-trip green BEFORE touching source/sink
3. Source TDD (with fake querier seam):
   a. compile-time SPI conformance
   b. Init config-parse + required-field errors + defaults
   c. ReadBatch: full / partial-then-EOF / empty-immediate-EOF / rows.Err()
   d. Close uninit + Close post-Init
   → verify: go test ./internal/source/ — all green, no real DB touched
4. Sink TDD (with fake executor seam):
   a. compile-time SPI conformance
   b. Init config-parse + required-field errors
   c. WriteBatch: empty no-op / type-assertion error / columns override / NULL → nil arg
   d. Close uninit + Close post-Init
   → verify: go test ./internal/sink/ — all green
5. cmd/plugin/main.go (ConfigSchema per §7, both ComponentSpecs)
   → verify: go build → ./plugin runs, prints handshake JSON, serves Discover
6. Container e2e (LATER, after user pulls images):
   a. round-trip across all SQL types
   b. NULL preservation
   c. empty result → DAG SUCCEEDED
   d. 10k-row throughput sanity
   → verify: real rows land in real DB, byte-equal to source
```

Order across the two connectors: **all of postgres (tasks 1-5) first**, then sqlserver (1-5), then e2e for both. The dbbatch file and the Kind-tag table are defined once on postgres and copy-pasted verbatim into sqlserver at task 2.

---

### Footnote: confirmed against live codebase

The following were read directly from `/Users/mw/workspace/repo/github.com/planx-lab/` during synthesis and ground the recommendations above:
- `planx-sdk-go/internal/batch/codec.go` — gob-through-`batchWrapper{Batch any}`, pre-registers only `map[string]string{}`.
- `planx-sdk-go/sdk/batch.go` — `type Batch = any`; `RegisterType` is a pass-through to `gob.Register`.
- `planx-sdk-go/sdk/schema.go` — full field-builder API (`StringField`/`IntegerField`/`SecretField`/`EnumField` + `Required`/`WithDefault`/`WithDescription`/`WithExample`); `ConfigValue` helpers `StringValue`/`IntValue`/`BoolValue`; no `SecretValue`.
- `planx-plugin-csv/cmd/plugin/main.go` — canonical `sdk.Serve` shape.
- `planx-plugin-csv/internal/{source,sink}/*.go` — canonical Source+Sink patterns, two-phase EOF, `RegisterType([][]string{})`, error-wrapping convention.
- `planx-plugin-csv/go.mod`, `.gitignore`, `repo.lock`, `AI.md` — boilerplate to copy.",
  "workflowProgress": [
    {
      "type": "workflow_phase",
      "index": 1,
      "title": "Research"
    },
    {
      "type": "workflow_phase",
      "index": 2,
      "title": "Synthesize"
    },
    {
      "type": "workflow_agent",
      "index": 1,
      "label": "sdk-gob-contract",
      "phaseIndex": 1,
      "phaseTitle": "Research",
      "agentId": "a99bd4e271e807572",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888249054,
      "queuedAt": 1782888249044,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "The planx-sdk-go batch contract is a thin gob wrapper. sdk.…",
      "promptPreview": "Investigate the planx-sdk-go batch/gob serialization contract — this determines what payload types a plugin's Batch can be.

Read these files in /Users/mw/workspace/repo/github.com/planx-lab/planx-sdk-go/ :
- sdk/batch.go (the public Batch alias + RegisterType)
- internal/batch/*.go (the Codec: Pack/Unpack implementation)
- internal/runtime/source_service.go / processor_service.go / sink_service.g…",
      "lastProgressAt": 1782888486315,
      "tokens": 96614,
      "toolCalls": 23,
      "durationMs": 237261,
      "resultPreview": "{"summary":"The planx-sdk-go batch contract is a thin gob wrapper. sdk.Batch is an alias for any (sdk/batch.go:5). Batches are wrapped in batchWrapper{Batch any} and gob-encoded to bytes via gob.NewEncoder(buf).Encode (internal/batch/codec.go:35-36), then carried opaquely in pb.Batch.payload (planx-proto/v4/batch.proto:11-14). The engine never inspects payload bytes (dag_run.go only concatenates t…"
    },
    {
      "type": "workflow_agent",
      "index": 2,
      "label": "connector-pattern (retry 1)",
      "phaseIndex": 1,
      "phaseTitle": "Research",
      "agentId": "a846003e8ac4e198e",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888249055,
      "queuedAt": 1782888249044,
      "attempt": 2,
      "lastAttemptReason": "stalled",
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Extracted the canonical Planx Source+Sink connector pattern…",
      "promptPreview": "Extract the canonical Planx connector pattern from the existing real connectors, so the new SQL Server + PostgreSQL connectors follow it exactly.

Read in /Users/mw/workspace/repo/github.com/planx-lab/ :
- planx-plugin-csv/ (the FIRST real multi-component connector: cmd/plugin/main.go, internal/source/source.go, internal/sink/sink.go, internal/source/source_test.go, internal/sink/sink_test.go, go.…",
      "lastProgressAt": 1782888718992,
      "tokens": 107364,
      "toolCalls": 36,
      "durationMs": 469936,
      "resultPreview": "{"summary":"Extracted the canonical Planx Source+Sink connector pattern from planx-plugin-csv (the first real multi-component connector), cross-checked against planx-plugin-template-go and planx-plugin-source-hello, and verified every SDK signature (SPI interfaces, Schema/SecretField builders, RegisterType, Serve/ComponentSpec) directly in planx-sdk-go/sdk. The pattern is consistent across all thr…"
    },
    {
      "type": "workflow_agent",
      "index": 3,
      "label": "pgx-postgres",
      "phaseIndex": 1,
      "phaseTitle": "Research",
      "agentId": "af99094759c3a1513",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888249055,
      "queuedAt": 1782888249044,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Researched pgx/v5 (latest v5.10.0) for a Planx PG Source+Si…",
      "promptPreview": "Research the PostgreSQL driver github.com/jackc/pgx/v5 for building a Planx Source (batch SELECT) + Sink (batch INSERT) connector. Use web search + the driver's documentation/source.

Cover precisely:
1. Connection: how to connect (pgx.ParseConfig + pgx.Connect, or pgxpool.New). Connection string format (postgres:// or key=value). Recommend pool vs single conn for a plugin.
2. Source (batch SELECT…",
      "lastProgressAt": 1782888742893,
      "tokens": 130859,
      "toolCalls": 24,
      "durationMs": 493838,
      "resultPreview": "{"summary":"Researched pgx/v5 (latest v5.10.0) for a Planx PG Source+Sink connector and empirically verified the gob serialization behavior against the actual Planx SDK codec (/Users/mw/workspace/repo/github.com/planx-lab/planx-sdk-go/internal/batch/codec.go). The decisive finding: the SDK batch wire format is gob-through-interface (batchWrapper{Batch any}), which has a hard rule that EVERY concre…"
    },
    {
      "type": "workflow_agent",
      "index": 4,
      "label": "mssql-sqlserver",
      "phaseIndex": 1,
      "phaseTitle": "Research",
      "agentId": "a446f0e13e982f799",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888249055,
      "queuedAt": 1782888249044,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Researched github.com/microsoft/go-mssqldb (v1.10.0, the of…",
      "promptPreview": "Research the SQL Server driver github.com/microsoft/go-mssqldb for building a Planx Source (batch SELECT) + Sink (batch INSERT) connector. Use web search + the driver's documentation/source.

Cover precisely:
1. Connection: driver registration (mssqldb imported for side-effects), sql.Open("sqlserver", connString). Connection string format (sqlserver:// URL with user:pass@host:port?database=..., or…",
      "lastProgressAt": 1782888780946,
      "tokens": 119177,
      "toolCalls": 27,
      "durationMs": 531891,
      "resultPreview": "{"summary":"Researched github.com/microsoft/go-mssqldb (v1.10.0, the official Microsoft Go SQL Server driver) for a Planx Source (batch SELECT) + Sink (batch INSERT) connector. The driver is a standard database/sql driver registered under name \"sqlserver\", supports URL/ADO/ODBC DSN formats with the password passed inline, and returns heterogeneous Go types per column (int64/bool/float64/string/t…"
    },
    {
      "type": "workflow_agent",
      "index": 5,
      "label": "gob-heterogeneous-rows",
      "phaseIndex": 1,
      "phaseTitle": "Research",
      "agentId": "a573de2797ce23e7b",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888249055,
      "queuedAt": 1782888249044,
      "attempt": 1,
      "lastToolName": "StructuredOutput",
      "lastToolSummary": "Analyzed how to represent heterogeneous DB rows (int/string…",
      "promptPreview": "THIS IS THE CENTRAL TECHNICAL RISK. Solve: how to represent heterogeneous DB rows (mixed int/string/float/bytes/time/nil column values) as a gob-serializable Planx Batch payload, given the SDK's gob codec.

Background: csv used [][]string (uniform string type, gob-registered via init(){sdk.RegisterType([][]string{})}). DB rows are heterogeneous. gob does NOT serialize interface{}/any values unless…",
      "lastProgressAt": 1782888551930,
      "tokens": 95370,
      "toolCalls": 20,
      "durationMs": 302875,
      "resultPreview": "{"summary":"Analyzed how to represent heterogeneous DB rows (int/string/float/bytes/time/nil) as a gob-serializable Planx Batch, given the SDK wraps every batch in `batchWrapper{Batch any}` (codec.go:34-44) and Source/Sink are separate OS processes with independent per-process `gob.Register` global state. Empirically tested all six options for gob-roundtrip feasibility, payload bloat, and type fid…"
    },
    {
      "type": "workflow_agent",
      "index": 6,
      "label": "synthesize-design (retry 2)",
      "phaseIndex": 2,
      "phaseTitle": "Synthesize",
      "agentId": "a5e3c29f5982909b5",
      "model": "glm-5.2[1m]",
      "fallbackModel": "glm-5.2",
      "state": "done",
      "startedAt": 1782888780949,
      "queuedAt": 1782888780947,
      "attempt": 3,
      "lastAttemptReason": "stalled",
      "lastToolName": "Bash",
      "lastToolSummary": "cat /Users/mw/workspace/repo/github.com/planx-lab/planx-plu…",
      "promptPreview": "Synthesize a unified design document for two new Planx connectors — planx-plugin-postgres (PostgreSQL) and planx-plugin-sqlserver (SQL Server) — each a single self-describing binary with a Source component (batch SELECT, finite/EOF) and a Sink component (batch INSERT).

Here are the research findings from 5 parallel agents (SDK gob contract, connector pattern, pgx, mssqldb, gob-heterogeneous-rows)…",
      "lastProgressAt": 1782889496719,
      "tokens": 111022,
      "toolCalls": 14,
      "durationMs": 715769,
      "resultPreview":
