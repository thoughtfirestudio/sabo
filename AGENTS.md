# Sabo — Agent Context

**Sabo** (_yo sabo_ — "I know") is the open-source brain for your AI, pointed
at outcomes. Teams keep a corpus of OKF documents (portable markdown, no
vendor lock-in); any LLM harness of their choice connects via MCP to search,
read, and write that knowledge. The harness is the customer's; the outcomes
are yours. Sabo is the brain.

---

## What this repo is

A multi-tenant OKF knowledge corpus + MCP server:

- **The corpus** — plain markdown with YAML frontmatter (`type`, `title`,
  `description`, `tags`, `timestamp`). One concept per file. The path is the
  identity. Human- AND agent-readable.
- **The MCP server** — any LLM harness (Claude Desktop, Cursor, Claude Code,
  a custom MCP client) connects over Streamable HTTP and gets tools to search,
  read, write, and sync the corpus.
- **Connectors** — the `sync` tool pulls entities from external tools
  (Salesforce, HubSpot, Outreach, …) and turns them into OKF documents, so an
  LLM can ground itself on real account data.

Everything below is for people who build or extend Sabo itself. Users connect
to a running server over MCP — they never touch this code.

---

## Quick start (self-host)

Requirements: **Node 24+** and **Postgres 16+**.

```bash
npm install
```

Set up the database and an encryption key:

```bash
export DATABASE_URL="postgres://user:pass@host:5432/sabo"
export WEBHOOK_ENCRYPTION_KEY="$(openssl rand -hex 32)"   # prod only
export ADMIN_TOKEN="$(openssl rand -hex 24)"              # prod only
PORT=3000 node server.js
```

The server runs `db/schema.sql` idempotently at boot — no separate migrate
step in dev.

Create a workspace + API key:

```bash
node scripts/bootstrap.js --name "My Team" --slug my-team --show-encryption-key
```

Connect any MCP-aware client:

```json
{ "sabo": { "type": "http", "url": "http://localhost:3000/mcp",
            "headers": { "Authorization": "Bearer <key>" } } }
```

Admin panel: `http://localhost:3000/admin` (paste the same API key).

---

## Architecture

```
server.js              Boot: migrate → Express app → REST (/api) + setup
                       (/api/setup) + MCP (/mcp) + admin (/admin) →
                       register connectors → start webhook worker. Read first.
├─ db/schema.sql       Full schema, idempotent (IF NOT EXISTS everywhere).
│                      workspaces, api_keys, documents, document_versions,
│                      document_search, webhooks, webhook_deliveries,
│                      audit_log, users, workspace_members,
│                      connector_credentials
├─ db/client.js        pg pool + migrate() on boot
├─ lib/okf.js          Core: parse + validate OKF docs, search vectors, checksums
├─ lib/documents.js    Corpus engine: upsert/get/delete/list/search + version
│                      history (every write snapshots to document_versions)
├─ lib/auth.js         API keys (sha256-hashed, shown once), scopes, lifecycle
├─ lib/crypto.js       AES-256-GCM encryption for webhook secrets + credentials
├─ lib/ratelimit.js    Per-key fixed-window rate limiter
├─ lib/audit.js        Append-only audit trail (fire-and-forget)
├─ lib/webhooks.js     Outbound webhooks, HMAC-SHA256 signing, delivery worker
├─ lib/validate.js     Zod wrapper + asyncHandler (Express 4 doesn't catch async)
├─ lib/connectors/     Connector registry + encrypted credential store.
│                      Each connector: { id, name, credentialSchema, test,
│                      discover, sync }
├─ routes/api.js       REST surface: docs, search, keys, webhooks, connectors
├─ mcp/server.js       MCP server — per-session McpServer bound to the auth'd
│                      workspace. Self-discovery instructions included.
└─ public/admin.html   Single-page admin panel (vanilla HTML+JS, no framework)
```

**The one invariant:** every authenticated request carries
`req.auth = { workspaceId, workspaceName, workspaceSlug, keyId, keyHash,
keyLabel, scopes }`. Routes never derive identity themselves. MCP sessions
bind the same shape per session.

---

## Auth model

- **API keys only.** One key = one workspace. `Authorization: Bearer <key>`
  for both REST and MCP.
- Raw key shown exactly once (`sabo_live_...`); only its sha256 hash is
  stored. Scopes: `read`, `write`. Optional expiry. Revocation deletes
  permanently.
- `resolveKey()` is the one seam where SSO/OIDC plugs in later — the rest of
  the system never knows whether auth came from a key or a session.

## MCP surface

`POST /mcp` (Streamable HTTP, stateful). GET for SSE notification streams.
First request authenticates and binds a fresh `McpServer` to that session's
workspace — isolation is structural, not query-based.

| Tool | What it does |
|------|-------------|
| `workspace_info` | Overview: doc count, types, recent docs, webhooks |
| `search(query, type?, limit?)` | Full-text search with ranked hits |
| `list_docs(prefix?, type?, tag?, limit?, offset?)` | Browse the corpus |
| `read_doc(path)` | Full document content |
| `write_doc(path, content)` | Upsert (bumps version, fires webhook, audits) |
| `delete_doc(path)` | Soft delete — history preserved, write revives |
| `sync(query, connector_ids?, folder?)` | Discover + import from connected tools |

**MCP self-discovery:** the server `instructions` field tells every connecting
LLM to explore the workspace first — call `workspace_info` and `list_docs`,
read key structural docs, and write/read `mcp/connected/_index.md` as the
orientation map. The workspace describes itself; no onboarding docs.

---

## OKF format (the corpus)

Every document is markdown with YAML frontmatter:

```markdown
---
type: account
title: Acme Corp
description: Renewal risk flagged in Q3 sync
tags: [renewal, high-priority]
timestamp: 2026-10-06T20:00:00Z
---

# Acme Corp

Account context, links to related docs by relative path, plain prose.
```

Rules:

- `type`, `title`, `description`, `tags`, `timestamp` are required
- One concept per file
- The **path is the identity** — writing to an existing path updates it
- Cross-link by relative markdown path, not URLs
- Paths are lowercase, hyphen-separated, end in `.md`

Version history: every write appends an immutable row to `document_versions`
(full content + actor + checksum). `DELETE` is a soft delete — the doc leaves
the live corpus but history stays reachable; writing to the path revives it.

---

## Conventions

- **Node 24, plain ESM, Express 4.** No framework beyond Express. Vanilla
  HTML+JS for the admin panel. This is a ThoughtFire house style: plain and
  lean. Do not introduce React, TypeScript build steps, or ORMs without a
  strong argument.
- **Config lives in env vars, not code.** Any operational value that might be
  tuned — limits, timeouts, keys, addresses — is an env var with a sensible
  default. See `.env.example`.
- **Every write is audited.** Mutations through routes or MCP tools call
  `audit()` with workspaceId, actor, action, resource. Never skip it.
- **Async handlers use `asyncHandler` from `lib/validate.js`.** Express 4
  does not catch async rejections; the raw route would crash the process.
- **Comment the "why" on non-obvious code** — coordinate math, security
  decisions, bug fixes with a dated note. Skip comments that restate the code.
- **Security headers on every response** (HSTS, nosniff, frame denial, etc.)
  — set centrally in `server.js`, never per-route.

---

## Adding a connector

Connectors turn an external tool's API into OKF documents. One file in
`lib/connectors/`, then register it in the registry:

```js
export default {
  id: 'sfdc',
  name: 'Salesforce',
  credentialSchema: [ /* zod-ish field defs rendered by the admin panel */ ],
  async test(creds) { /* verify the connection */ },
  async discover(query, creds) { /* find best-matching entity */ },
  async sync(entity, creds) { /* build OKF doc(s) from the entity */ },
};
```

Credentials are AES-256-GCM encrypted at rest; the admin panel renders the
form from `credentialSchema` automatically. The `sync` MCP tool discovers
across all configured connectors and picks the best match.

Reference implementation: `lib/connectors/sfdc.js`.

---

## Testing

```bash
node --test
```

Existing tests: `test/` — unit tests for the event system (redaction,
timeouts, cancellation, observer isolation). Tests run against mocked
boundaries; they must not require a live Postgres.

---

## Security

Read `SECURITY.md` before pushing anything — it is the trust story and the
checklist against which this project is measured in enterprise procurement:

- API keys: sha256 hashes only, raw shown once
- Rate limits per key (REST 300/min, MCP 30 sessions/min), per-IP for
  bootstrap only
- AES-256-GCM at rest for webhook secrets and connector credentials
- Append-only audit log; version history + soft delete on documents
- Open source is the trust story — anyone can read the audit trail, the
  encryption, and the key handling

Report vulnerabilities privately (see SECURITY.md) — no public issues for
security bugs.

---

## Working here

- Commit style: warm, specific, plain. Tell the story of what changed and why
  it matters, not a bullet list of files.
- Keep AGENTS.md accurate — if you change architecture, data flow, env vars,
  or conventions, update this file in the same commit.
- The repo is public: never commit secrets, tokens, `.env*` files, or
  infrastructure addresses. `resolveKey`, the schema, and `SECURITY.md` are
  the contract; everything else is implementation.