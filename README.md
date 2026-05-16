# ⏱ mcp-time-traveler

> **Catch database migration footguns before they page you at 3am.**

A zero-config [Model Context Protocol](https://modelcontextprotocol.io) server that validates PostgreSQL migrations inside an ephemeral in-memory WASM database — returning real lock profiles, AST-level warnings, and production impact extrapolations directly to your AI agent.

```json
{
  "astWarnings": [
    "AST DANGER: Adding a NOT NULL column without a DEFAULT will rewrite and heavily lock the table."
  ],
  "executionStatus": "success",
  "empiricalLocks": [
    {
      "target_table": "users",
      "lock_mode": "AccessExclusiveLock",
      "granted": true
    }
  ],
  "productionImpactExtrapolation": "CRITICAL: This migration is projected to hold AccessExclusiveLock for ~47.50 seconds on a 50M row production table."
}
```

---

## The Problem

AI agents write migrations fast. Sometimes dangerously fast.

```sql
-- Your agent writes this. Looks fine.
ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL;

-- On a 40M row production table, this rewrites the entire table.
-- Your app is down for 12 minutes.
-- You are paged at 3am.
```

Standard CI pipelines run migrations against empty test databases. They pass. The footgun ships.

**mcp-time-traveler gives your AI agent a time machine** — a sandboxed environment to run, measure, and profile migrations against realistic data *before a single line touches staging.*

---

## How It Works

```
Developer / AI Agent
       │
       │  "test this migration"
       ▼
┌─────────────────────────────────┐
│        MCP Time Traveler        │
│                                 │
│  1. AST Parse (pgsql-ast-parser)│  ← catches syntax-level footguns
│  2. PGlite WASM spin-up (~20ms) │  ← zero Docker, zero daemon
│  3. Schema hydration            │  ← up to 100k rows via generate_series
│  4. AST Execution Router        │  ← CONCURRENTLY-aware transaction mode
│  5. pg_locks query              │  ← real empirical lock data
│  6. Impact extrapolation        │  ← projects lock duration to prod scale
│  7. RAM wipe (db.close())       │  ← instant isolation, no cleanup needed
└─────────────────────────────────┘
       │
       │  structured safety report
       ▼
  Agent corrects migration autonomously
```

**Two execution paths, automatically routed:**

| Migration type | Execution mode | Why |
|---|---|---|
| Standard `ALTER`, `CREATE INDEX` | Transaction + `pg_locks` query | Locks held until rollback — captured accurately |
| `CREATE INDEX CONCURRENTLY` | Auto-commit mode | Postgres forbids CONCURRENTLY inside a transaction block |

---

## Quickstart

Add to your `claude_desktop_config.json` or Cursor MCP settings:

```json
{
  "mcpServers": {
    "time-traveler-db": {
      "command": "npx",
      "args": ["-y", "@mcp-builders/time-traveler"]
    }
  }
}
```

That's it. No Docker. No Postgres installation. No port conflicts. Runs entirely in-process via WebAssembly.

---

## Tool Reference

### `test_migration_ultimate`

| Parameter | Type | Required | Description |
|---|---|---|---|
| `baselineSchemaSql` | `string` | ✅ | Your current production schema DDL |
| `migrationSql` | `string` | ✅ | The migration to test |
| `scaleFactor` | `"none" \| "light" \| "heavy"` | ❌ | Seed data volume (0 / 10k / 100k rows). Default: `light` |

**Example agent prompt:**

> "Before applying this migration, use the time-traveler MCP to test it against our current schema with a light scale factor."

---

## What Gets Caught

### AST-Level (static, before execution)

| Pattern | Warning |
|---|---|
| `ADD COLUMN ... NOT NULL` without `DEFAULT` | Full table rewrite — will lock on large tables |
| `CREATE INDEX` without `CONCURRENTLY` | Write lock during index build |
| `DROP INDEX` without `CONCURRENTLY` | Write lock during index removal |
| `DROP TABLE` | Destructive operation flagged |

### Empirical (runtime, from `pg_locks`)

Real PostgreSQL lock modes returned from the live execution:

| Lock mode | Impact |
|---|---|
| `AccessExclusiveLock` | Blocks all reads and writes |
| `ExclusiveLock` | Blocks writes and most reads |
| `ShareLock` | Blocks writes |
| `RowExclusiveLock` | Normal DML — typically safe |

### Production Impact Extrapolation

If `scaleFactor` is `light` or `heavy`, the server measures actual execution time against mock data and extrapolates projected lock duration at 50M row production scale — flagging migrations likely to cause pool exhaustion or timeout events.

---

## Why WASM Instead of Docker

| | Docker (postgres:15-alpine) | PGlite (WASM) |
|---|---|---|
| Cold start | ~3–7 seconds | ~20ms |
| External dependency | Docker daemon required | None |
| Port conflicts | Possible | Impossible |
| Works on CI | Requires DinD | Native |
| Cleanup | `container.stop()` + `container.remove()` | `db.close()` |

The entire Postgres engine runs in-process. When `db.close()` is called, the instance is garbage collected. No orphaned containers, no leaked ports.

---

## Tech Stack

- **[Model Context Protocol SDK](https://github.com/modelcontextprotocol/sdk)** — stdio transport layer for agent communication
- **[PGlite](https://github.com/electric-sql/pglite)** — PostgreSQL compiled to WASM via Emscripten, running in-process
- **[pgsql-ast-parser](https://github.com/oguimbal/pgsql-ast-parser)** — PostgreSQL AST parser using the actual Postgres grammar
- **TypeScript** — strict mode, ESM

---

## Local Development

```bash
git clone https://github.com/SameerFGupta/mcp-time-traveler
cd mcp-time-traveler
npm install
npm run build
node dist/index.js
```

Test manually via stdin:

```bash
echo '{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "test_migration_ultimate",
    "arguments": {
      "baselineSchemaSql": "CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR(255) UNIQUE NOT NULL);",
      "migrationSql": "ALTER TABLE users ADD COLUMN status VARCHAR(50) NOT NULL;",
      "scaleFactor": "none"
    }
  }
}' | node dist/index.js
```

---

## Roadmap

- [ ] Volume mount support for large `.sql` schema dumps (bypass JSON-RPC size limits)
- [ ] Multi-statement migration sequencing
- [ ] `pg_stat_activity` integration for query plan analysis
- [ ] Named migration profiles (store and reuse baseline schemas)
- [ ] VS Code extension

---

## License

MIT
