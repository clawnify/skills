---
name: build-a-clawnify-app
description: Author a Clawnify app end-to-end — the canonical Hono + React + Vite + @hono/zod-openapi + @clawnify/db (Drizzle) stack, schema.ts (typed queries) kept in sync with schema.sql (the DDL the deploy applies), OpenAPIHono routes (createRoute) that auto-generate an OpenAPI spec, and X-Clawnify-* identity headers. Use when building or modifying an app that deploys to the Clawnify platform.
---

# Build a Clawnify app

Canonical skill: how to author a Clawnify app end-to-end. Same content
for humans reading the docs and for AI coding agents injecting this as
instructions during an app build.

If you're an agent: this document is enough context to produce a
working app from scratch. Follow it literally. The patterns here
are exactly what `clawnify init` scaffolds and what the platform
expects.

## TL;DR

- **One stack.** Hono + React + Vite + `@hono/zod-openapi` +
  `@clawnify/db` (Drizzle) + Tailwind v4. Nothing else.
- **Schema lives in two files kept in sync** — `schema.ts` (Drizzle
  DSL, the typed view your queries use) and `schema.sql` (the DDL the
  deploy applies to your app's D1). Change one, change the other.
- **Queries through `getDB`** — `import { getDB, eq } from
  "@clawnify/db"`. JSON columns auto-serialize.
- **API routes are `OpenAPIHono` + `createRoute`** — Zod-validated
  request/response. `app.doc()` auto-generates the OpenAPI spec at
  `/api/openapi.json`. That spec *is* the tool surface.
- **`clawnify.json`** declares framework, env, and public routes.
  There is no `api.tools[]` — the agent discovers and calls your
  endpoints by reading the live OpenAPI spec.
- **Identity flows via `X-Clawnify-*` headers** injected by the
  platform. Read them, don't fake them.

## Stack rationale

One template, one set of decisions. Don't deviate unless the user has
explicitly asked. `clawnify init` (type `app`) scaffolds either a
`blank` template (one GET route + a page) or a `crud` template (a
table with list/create/update/delete + a form). Both share the exact
same stack below.

## File layout

```
my-app/
  clawnify.json              ← Manifest (validated against /schema/v1/clawnify.json)
  package.json
  tsconfig.json
  vite.config.ts             ← @vitejs/plugin-react + @tailwindcss/vite; proxies /api → :8787
  index.html
  .gitignore
  .clawnify/                 ← CLI cache (gitignored)
    wrangler.toml            ← Generated per-dev wrangler config
  src/
    server/
      index.ts               ← OpenAPIHono entry — mounts routes, serves /api/openapi.json
      routes.ts              ← OpenAPIHono + createRoute definitions — the API surface
      schema.ts              ← Drizzle table definitions (typed view for queries)
      schema.sql             ← canonical DDL the deploy applies — keep in sync with schema.ts
      worker-env.d.ts        ← Cloudflare Workers types reference
      uploads.ts             ← R2 helpers (only when storage is enabled)
    client/
      main.tsx               ← React mount point
      app.tsx                ← Root component (plain fetch against /api/*)
      index.css              ← @import "tailwindcss"
```

## Schema — `schema.ts` + `schema.sql`, kept in sync

Two files describe the same tables. `schema.ts` (Drizzle DSL) is the
typed view your queries compile against; `schema.sql` is the DDL the
deploy applies to your app's D1. **When you change one, update the
other** — they must describe the same tables.

The scaffold's starter table (blank template):

```ts
// src/server/schema.ts
import { sqliteTable, text, integer } from "@clawnify/db";

export const items = sqliteTable("items", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  name: text("name").notNull(),
  createdAt: text("created_at").notNull().$default(() => new Date().toISOString()),
});
```

Real apps are multi-tenant. Every user-facing table carries an
`org_id`, every query filters by it. JSON columns and indexes use the
Drizzle DSL re-exported from `@clawnify/db`:

```ts
import { sqliteTable, text, integer, index } from "@clawnify/db";

export const notes = sqliteTable(
  "notes",
  {
    id: integer("id").primaryKey({ autoIncrement: true }),
    orgId: text("org_id").notNull(),       // ALWAYS filter by this in queries
    title: text("title").notNull(),
    body: text("body").notNull().default(""),
    tags: text("tags", { mode: "json" }).$type<string[]>().notNull().default([]),
    createdAt: text("created_at").notNull().$default(() => new Date().toISOString()),
  },
  (t) => ({
    byOrg: index("notes_by_org").on(t.orgId, t.createdAt),
  }),
);
```

Rules:
- JSON columns: `text(col, { mode: "json" }).$type<T>()` — Drizzle
  serializes on write and parses on read.
- Multi-tenancy: every user-facing table has `org_id`. Every query
  filters by it. No exceptions.
- Identifiers: snake_case in SQL, camelCase in TS (Drizzle bridges).

## `schema.sql` — the DDL the deploy applies

Alongside `schema.ts`, keep a `schema.sql` describing the same tables.
This is the DDL applied to your app's database on deploy.

```sql
-- src/server/schema.sql  (mirror of the schema.ts tables)
CREATE TABLE IF NOT EXISTS notes (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  org_id     TEXT NOT NULL,
  title      TEXT NOT NULL,
  body       TEXT NOT NULL DEFAULT '',
  tags       TEXT NOT NULL DEFAULT '[]',        -- JSON column
  created_at TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS notes_by_org ON notes(org_id, created_at);
```

Rules:
- Always `CREATE TABLE IF NOT EXISTS` — the apply is additive and
  re-run-safe.
- Add a new column with `ALTER TABLE <t> ADD COLUMN …` at the **end** of
  `schema.sql`, and reflect it in `schema.ts`. The deploy adds it.
- Deploys are **additive**: new tables and columns are added
  automatically. Destructive changes (dropping or retyping a column) are
  not applied automatically — do those deliberately, and never drop a
  table with user data unless explicitly asked.
- **Never let `schema.ts` and `schema.sql` drift.** Change one, change
  the other — they must describe the same tables.

## `src/server/index.ts` — the OpenAPIHono entry

```ts
import { OpenAPIHono } from "@hono/zod-openapi";
import api from "./routes";

type Env = { Bindings: { DB: D1Database } };

const app = new OpenAPIHono<Env>();

app.route("/", api);

app.doc("/api/openapi.json", {
  openapi: "3.0.0",
  info: { title: "My app", version: "1.0.0" },
});

export default app;
```

The entry is thin: mount the routes, then `app.doc()` — that single
call generates a live OpenAPI 3.0 spec from your `createRoute`
definitions and serves it at `/api/openapi.json`. You never write a
`/openapi.json` handler by hand; the framework derives it.

When storage (file uploads) is enabled, the entry also wires the R2
bucket in a middleware before mounting routes:

```ts
import { OpenAPIHono } from "@hono/zod-openapi";
import { initUploads } from "./uploads";
import api from "./routes";

type Env = { Bindings: { DB: D1Database; UPLOADS: R2Bucket } };

const app = new OpenAPIHono<Env>();

app.use("*", async (c, next) => {
  initUploads(c.env.UPLOADS);
  await next();
});

app.route("/", api);

app.doc("/api/openapi.json", {
  openapi: "3.0.0",
  info: { title: "My app", version: "1.0.0" },
});

export default app;
```

## `src/server/routes.ts` — OpenAPIHono routes with `createRoute`

Each endpoint is a `createRoute({ method, path, summary, request, responses })`
definition paired with a handler via `api.openapi(route, handler)`.
Request bodies and params are Zod-validated (`c.req.valid("json")`,
`c.req.valid("param")`); response shapes are declared with
`.openapi("Name")` Zod schemas so they land in the generated spec.

Blank template — one read route:

```ts
import { OpenAPIHono, createRoute, z } from "@hono/zod-openapi";
import { getDB, desc } from "@clawnify/db";
import * as schema from "./schema";

type Env = { Bindings: { DB: D1Database } };
const api = new OpenAPIHono<Env>();

const ItemSchema = z.object({
  id: z.number(),
  name: z.string(),
  createdAt: z.string(),
}).openapi("Item");

api.openapi(
  createRoute({
    method: "get",
    path: "/api/items",
    summary: "List items",
    responses: {
      200: {
        description: "List of items",
        content: { "application/json": { schema: z.array(ItemSchema) } },
      },
    },
  }),
  async (c) => {
    const db = getDB(c.env, { schema });
    const items = await db
      .select()
      .from(schema.items)
      .orderBy(desc(schema.items.createdAt));
    return c.json(items);
  },
);

export default api;
```

CRUD template — the full list/create/update/delete surface:

```ts
import { OpenAPIHono, createRoute, z } from "@hono/zod-openapi";
import { getDB, eq, desc } from "@clawnify/db";
import * as schema from "./schema";

type Env = { Bindings: { DB: D1Database } };
const api = new OpenAPIHono<Env>();

const ItemSchema = z.object({
  id: z.number(),
  title: z.string(),
  status: z.string(),
  createdAt: z.string(),
}).openapi("Item");

const CreateItemSchema = z.object({ title: z.string().min(1) }).openapi("CreateItem");
const UpdateItemSchema = z.object({
  title: z.string().min(1).optional(),
  status: z.string().optional(),
}).openapi("UpdateItem");
const IdParamSchema = z.object({ id: z.string() });
const OkSchema = z.object({ ok: z.boolean() }).openapi("Ok");

api.openapi(
  createRoute({
    method: "get",
    path: "/api/items",
    summary: "List items",
    responses: {
      200: {
        description: "List of items",
        content: { "application/json": { schema: z.array(ItemSchema) } },
      },
    },
  }),
  async (c) => {
    const db = getDB(c.env, { schema });
    const items = await db
      .select()
      .from(schema.items)
      .orderBy(desc(schema.items.createdAt));
    return c.json(items);
  },
);

api.openapi(
  createRoute({
    method: "post",
    path: "/api/items",
    summary: "Create item",
    request: { body: { content: { "application/json": { schema: CreateItemSchema } } } },
    responses: {
      201: {
        description: "Created item",
        content: { "application/json": { schema: ItemSchema } },
      },
    },
  }),
  async (c) => {
    const { title } = c.req.valid("json");
    const db = getDB(c.env, { schema });
    const [row] = await db
      .insert(schema.items)
      .values({ title: title.trim() })
      .returning();
    return c.json(row, 201);
  },
);

api.openapi(
  createRoute({
    method: "patch",
    path: "/api/items/:id",
    summary: "Update item",
    request: {
      params: IdParamSchema,
      body: { content: { "application/json": { schema: UpdateItemSchema } } },
    },
    responses: {
      200: { description: "Updated", content: { "application/json": { schema: OkSchema } } },
    },
  }),
  async (c) => {
    const { id } = c.req.valid("param");
    const body = c.req.valid("json");
    const patch: Partial<typeof schema.items.$inferInsert> = {};
    if (body.title !== undefined) patch.title = body.title;
    if (body.status !== undefined) patch.status = body.status;
    if (Object.keys(patch).length === 0) return c.json({ ok: false }, 400);
    const db = getDB(c.env, { schema });
    await db.update(schema.items).set(patch).where(eq(schema.items.id, Number(id)));
    return c.json({ ok: true });
  },
);

api.openapi(
  createRoute({
    method: "delete",
    path: "/api/items/:id",
    summary: "Delete item",
    request: { params: IdParamSchema },
    responses: {
      200: { description: "Deleted", content: { "application/json": { schema: OkSchema } } },
    },
  }),
  async (c) => {
    const { id } = c.req.valid("param");
    const db = getDB(c.env, { schema });
    await db.delete(schema.items).where(eq(schema.items.id, Number(id)));
    return c.json({ ok: true });
  },
);

export default api;
```

Patterns to follow:
- **Give every route a clear `summary`/`description`.** The generated
  OpenAPI spec is what the agent reads to decide which endpoint to
  call — a vague summary makes your endpoint invisible in practice.
- **Validate input with Zod** via `request.body` / `request.params`
  and read it with `c.req.valid("json")` / `c.req.valid("param")`.
  Never parse `c.req.raw` by hand.
- **Declare response shapes** with `.openapi("Name")` schemas — they
  become the documented return type in the spec.
- **Filter by the org** on every user-facing query (see identity
  headers below). No exceptions.

## `src/server` — reading identity headers

The scaffold's starter routes are single-tenant for brevity. Real
apps read the caller's identity from the `X-Clawnify-*` headers the
platform injects, using Hono's `c.req.header(...)` inside the handler:

```ts
const orgId = c.req.header("X-Clawnify-Org-Id");
if (!orgId) return c.json({ error: "unauthorized" }, 401);
const db = getDB(c.env, { schema });
const rows = await db
  .select()
  .from(schema.notes)
  .where(eq(schema.notes.orgId, orgId));
```

The headers the platform injects (never set by the client — it strips
client-supplied versions and reinjects authenticated ones at the
perimeter):

| Header | Value |
|--------|-------|
| `X-Clawnify-User-Id` | Supabase user UUID |
| `X-Clawnify-Org-Id` | Organization UUID |
| `X-Clawnify-User-Email` | User email |
| `X-Clawnify-Caller` | `user`, `agent`, `agent-browser`, or `system` |

Trust these headers. Never read `Authorization`, `Bearer` tokens, or
session cookies directly.

## `src/client` — React + plain fetch

The client mounts React and talks to the API with plain `fetch`
against the `/api/*` routes. No data-layer library is scaffolded.

```tsx
// src/client/main.tsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { App } from "./app";
import "./index.css";

createRoot(document.getElementById("app")!).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

```tsx
// src/client/app.tsx (CRUD template shape)
import { useState, useEffect } from "react";

interface Item {
  id: number;
  title: string;
  status: string;
  createdAt: string;
}

export function App() {
  const [items, setItems] = useState<Item[]>([]);
  const [title, setTitle] = useState("");

  async function load() {
    const res = await fetch("/api/items");
    setItems(await res.json());
  }

  useEffect(() => { load(); }, []);

  async function addItem(e: React.FormEvent) {
    e.preventDefault();
    if (!title.trim()) return;
    await fetch("/api/items", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title }),
    });
    setTitle("");
    load();
  }

  // ...toggleStatus / deleteItem call PATCH/DELETE the same way.

  return (
    <div className="max-w-xl mx-auto mt-10 px-4">
      {/* form + list */}
    </div>
  );
}
```

In local dev, Vite proxies `/api` to the Worker on `:8787` (see
`vite.config.ts`), so the same relative fetch paths work in dev and
in production. The API is the one contract: the UI calls it over
`fetch`, and the agent calls it over the OpenAPI spec.

## Design

If the project contains a `DESIGN.md`, it is the single source of
truth for visuals — follow its tokens, don't restyle around it. The
scaffold ships Tailwind v4 (via `@tailwindcss/vite`, imported with
`@import "tailwindcss";` in `index.css`) and nothing else — no
component library. The rules every Clawnify app follows:

- **Neutral chrome.** White canvas and surfaces, one text ramp
  (foreground → muted → faint), hairline borders. Color is for
  meaning, not decoration.
- **One primary CTA per screen.** Blue is reserved for links and
  focus rings; never a second accent.
- **Status colors are muted pairs** — never pure-saturated fills.
- **Style with Tailwind utilities.** Don't install another component
  library or write a bespoke CSS system unless the user asks.
- **Sizes in rem**, so browser zoom and font settings are respected.

## `clawnify.json` — the manifest

```json
{
  "$schema": "https://app.clawnify.com/schema/v1/clawnify.json",
  "version": 1,
  "name": "Notes",
  "description": "Take and organize notes. Searchable by tag.",
  "app": {
    "framework": "react+hono",
    "database": true,
    "storage": false
  },
  "env": {
    "optional": ["OPENROUTER_API_KEY"]
  },
  "api": {
    "public_routes": [
      { "path": "/api/feed", "methods": ["GET"] }
    ]
  }
}
```

`clawnify init` writes the minimal form:
`{ app: { framework: "react+hono", database: true, storage } }`. Add
the rest as needed.

If the app talks to third-party services the user has connected
(ads platforms, CRMs, messaging), declare them:

```json
{ "app": { "credentials": ["metaads", "googleads"] } }
```

Get the ids and names from `clawnify connections list` — see the
naming rule below.

`env` supports `required` (deploy warns if missing), `optional`
(injected when available), and `oneOf` (at least one per group).

Rules:
- **There is no `api.tools[]`.** The manifest's `api` object only
  accepts `public_routes` and `automation_bypass`. Your API surface
  is discovered from the generated OpenAPI spec, not a hand-authored
  tools list (see the next section).
- **Public routes are the exception**, not the default. Every route
  is behind the Clawnify perimeter (auth-required) unless you
  explicitly add a matching path to `api.public_routes`. Use only for
  webhooks, RSS, embedded feeds.
- **`framework: "react+hono"`** for new apps. Never `preact+hono` or
  `vite-preact` — those are legacy.
- **`database: true`** if the app uses a DB. Ship both `schema.ts`
  (typed) and `schema.sql` (the applied DDL), kept in sync.

## How the agent reaches your API (no `api.tools[]`)

The old tRPC stack emitted an `api.tools[]` array from procedure meta.
That mechanism is gone. Today the flow is:

1. Your `createRoute` definitions + `app.doc("/api/openapi.json", …)`
   auto-generate a live OpenAPI 3.0 spec.
2. The user's agent calls the platform's built-in tools —
   `get_app_openapi` (fetch the app's OpenAPI JSON) and `call_app_api`
   (invoke a `method` + `path` on the live app) — to discover and
   drive your endpoints. No per-app MCP registration, no manifest
   tools array.

So you make an endpoint "agent-callable" simply by shipping it with a
clear `summary`/`description` and Zod-typed request/response. The
richer and clearer the spec, the more reliably the agent uses it.

## Connections & secrets — `@clawnify/connections`

**The naming rule — never invent an identifier.** Before declaring
`credentials` or env names, run `clawnify connections list`: it shows
the org's canonical integration ids (`googleads`, never `google_ads`),
the fixed provider-key names (`OPENROUTER_API_KEY`, never
`OPENROUTER_KEY`), and custom env vars already defined — each with
connected/set state. Reuse those exact identifiers; a misspelled one
doesn't fail the build, it silently resolves to nothing at runtime. If
you need a custom secret that isn't listed, pick a clear name and tell
the user to set it in the dashboard.

If the app needs a third-party service or a provider API key, use
the `@clawnify/connections` SDK. Never read credential bindings
directly, never paste API keys, never hardcode a token or endpoint.

```ts
import { connect, secret, describe, searchActions } from "@clawnify/connections";

// A connected integration:
const meta = connect("metaads", env);
const accounts = await meta.adAccounts("name,currency");

// A provider key the user supplied (null if absent):
const key = secret("OPENROUTER_API_KEY", env);

// What's wired vs missing — drive your setup/empty state from this:
const status = await describe(env, undefined, REQUIRES);
```

How to call a connected service — the mistake that silently breaks
integration apps, so get it right:

- **Prefer the typed client's semantic methods** where they exist
  (`meta.adAccounts(...)`, `googleads.query(gaql)`).
- **For anything else, discover and run an action:**
  ```ts
  const [best] = await searchActions("list google ads campaigns", env);
  const data = await connect(best.service, env).run(best.slug, body);
  ```
- Some integrations also expose a raw REST client
  (`connect("bird", env).get("/path")`) — but if `.get()`/`.post()`
  throws for a service, that service only supports semantic methods
  and `run()`. When unsure, use `run()`.
- **Need exact paths/arguments for a raw-REST provider?** Some carry
  a `spec_url` (the provider's published OpenAPI spec) —
  `clawnify connections list --json` shows it. Download and query it —
  `curl -s <spec_url> -o spec.json && jq '.paths | keys' spec.json` —
  never read a whole spec into context (they run to 10+ MB).
- **Never fall back to raw `fetch`** against a provider API, and
  never invent endpoints or guess tokens.

**The one exception — a service we don't support at all.** If a
provider isn't in the connections registry (nothing to `connect()`
to, and `clawnify connections list` doesn't show it), the sanctioned
path *is* a raw call with a user-supplied key: declare the key(s) it
needs in `clawnify.json` `env` (`required`/`optional`/`oneOf`), tell
the user to set them in the dashboard, read them with
`secret("THAT_KEY", env)`, and `fetch` the provider's documented
endpoints directly. This is the escape hatch for the long tail of
integrations we don't model yet — it does **not** apply to a service
we *do* support (use `connect()` / `run()` for those, always). Keep
the endpoint and token server-side, never in the client bundle.

Build a `describe()`-driven setup state instead of assuming
connections exist — show the user what's missing and where to
connect it.

## Scheduling & deferred work — the managed queue

A deployed app **cannot bind Cloudflare Queues, Durable Objects, or cron
triggers** (the platform builds every app with a fixed binding set: `DB` +
`ASSETS` + `UPLOADS`). To run work later — a scheduled publish, a reminder, a
retry, a "do this in 10 minutes" — use the **Clawnify managed queue service**
over HTTP. It owns the clock, retries, and at-least-once delivery, then calls an
endpoint on *your own app* back at the scheduled time.

It's a thin seam over a platform primitive, same shape as `@clawnify/db` /
`@clawnify/connections` (a `@clawnify/queue` package is coming; inline the seam
until then). Auth is the build-injected `env.CLAWNIFY_TOKEN`; with no token
(local dev) the seam degrades to a no-op.

**Enqueue** a deferred call to your own endpoint:

```ts
// POST https://services.clawnify.com/queue/enqueue  (Bearer CLAWNIFY_TOKEN)
const res = await fetch("https://services.clawnify.com/queue/enqueue", {
  method: "POST",
  headers: { Authorization: `Bearer ${env.CLAWNIFY_TOKEN}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    target_url: `${origin}/api/internal/publish`, // an endpoint on THIS app
    payload: { post_id: id },                     // echoed back to you on fire
    run_at: new Date(whenMs).toISOString(),
    idempotency_key: `post:${id}:${whenMs}`,       // same key dedupes; new time = new job
  }),
});
const { job_id } = await res.json();  // store it to cancel/reschedule
```

Persist `job_id` on the row (e.g. `posts.queue_job_id`). **Cancel / reschedule**:
`DELETE https://services.clawnify.com/queue/jobs/{job_id}` (Bearer token);
reschedule = cancel + a fresh enqueue.

**The callback endpoint is a public, signature-verified route.** The queue calls
`target_url` from outside the perimeter, so the endpoint must be declared in
`clawnify.json` `public_routes` **and** verify the delivery signature itself —
the signature *is* its auth (never trust an unsigned call):

```jsonc
// clawnify.json
"api": { "public_routes": [ { "path": "/api/internal/publish", "methods": ["POST"] } ] }
```

Deliveries are signed with the platform's **ECDSA P-256 (ES256)** key,
Stripe-style over `${timestamp}.${rawBody}`, verified against
`https://services.clawnify.com/.well-known/jwks.json` (public key — nothing
breaks on token rotation). Read the raw body, check the signature + a timestamp
tolerance (~5 min) before doing anything:

```ts
app.post("/api/internal/publish", async (c) => {
  const raw = await c.req.text();
  const ok = await verifyDelivery(raw, {
    signature: c.req.header("x-clawnify-signature"),
    timestamp: c.req.header("x-clawnify-timestamp"),
    keyId: c.req.header("x-clawnify-key-id"),
  });
  if (!ok) return c.json({ error: "bad signature" }, 401);
  const { post_id } = JSON.parse(raw);
  // …do the deferred work (short: it's one normal Worker invocation)…
  return c.json({ ok: true });
});
```

Each fire is a normal Worker request — size the work to one invocation (publish a
post, send a digest). For long compute *now* (not deferred), hold the request
open and return when done; don't fire-and-forget past the response.

## Migrations

Keep `schema.ts` and `schema.sql` in sync by hand. The workflow is:

1. Edit `src/server/schema.ts` for the new shape (types + queries).
2. Make the matching edit in `src/server/schema.sql` — a new
   `CREATE TABLE IF NOT EXISTS`, or `ALTER TABLE … ADD COLUMN …` at the
   **end** of the file for a new column.
3. Run `clawnify dev` locally, then `clawnify deploy`.

On deploy your schema changes are applied to the app's database — new
tables and columns are added automatically.

Destructive changes (drop column, rename) are not applied automatically;
do them deliberately, and only when the user explicitly asks. Renames
are risky on production data — prefer "add new column, backfill, switch
queries, drop old column" over a single-step rename.

## What the agent must NOT do

- Author a `/openapi.json` handler by hand. `app.doc("/api/openapi.json", …)`
  generates it from your `createRoute` definitions.
- Let `schema.ts` and `schema.sql` drift. They describe the same
  tables — change both, every time.
- Hand-author migration SQL files. Make schema changes in `schema.sql`
  (and mirror them in `schema.ts`); the deploy applies them.
- Invent an `api.tools[]` array in the manifest. It isn't a valid
  field — the OpenAPI spec is the tool surface.
- Use `JSON.stringify` on query params. JSON columns auto-serialize
  via Drizzle. The raw SQL API has `json()` as a helper.
- Skip the `org_id` filter on a user-facing query. Multi-tenant
  leak is the worst-class bug we can ship.
- Read auth from `Authorization` / cookies / `Bearer` tokens. Use
  the `X-Clawnify-*` headers via `c.req.header(...)`.
- Call third-party APIs with raw `fetch` or hardcoded tokens. Go
  through `@clawnify/connections` (`connect` / `secret` / `run`).
- Install `drizzle-orm` as a separate top-level dependency.
  `@clawnify/db@^0.4.1` pulls it in transitively; only add it if you
  need to pin a specific version (rare).

## What the agent SHOULD do

- Import everything from `@clawnify/db` where possible
  (`getDB`, `eq`, `and`, `desc`, `sql`, `sqliteTable`, `text`,
  `integer`, `index`, etc. are all re-exported there). Drop to
  `drizzle-orm` directly only for things not re-exported.
- Give every route a clear `summary`/`description` and Zod-typed
  request/response so the agent can discover and call it via the
  OpenAPI spec.
- Test on the draft URL (`<slug>.draft.clawnify.com`) before
  publishing — the preview tier catches schema drift and type errors
  at build time.
- When unsure about a Drizzle pattern, check the `@clawnify/db`
  README for the current API + version notes.

## Schema-as-data exception

Apps that let END users define data shapes at runtime (CMS, no-code
table editor) can't model their schema in `schema.ts` — the table
definitions live in rows. Those apps fall back to the raw SQL API
from `@clawnify/db` (`query`, `get`, `run`, `json`). Open-cms is
the canonical example.

If you're building this kind of app, you still use Drizzle's
`schema.ts` for the **registry** tables (content types, field
definitions). Only the dynamically-spawned per-collection tables
need raw SQL. See open-cms's `src/server/schema-sync.ts` for the
runtime DDL pattern.
