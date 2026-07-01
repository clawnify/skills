---
name: build-a-clawnify-app
description: Author a Clawnify app end-to-end — the canonical Hono + React + Vite + tRPC + @clawnify/db (Drizzle) stack, schema.ts as the single source of truth, MCP-tagged tRPC procedures, and X-Clawnify-* identity headers. Use when building or modifying an app that deploys to the Clawnify platform.
---

# Build a Clawnify app

Canonical skill: how to author a Clawnify app end-to-end. Same content
for humans reading the docs and for AI coding agents injecting this as
instructions during an app build.

If you're an agent: this document is enough context to produce a
working app from scratch. Follow it literally. The patterns here
are the patterns the platform expects.

## TL;DR

- **One stack.** Hono + React + Vite + tRPC + `@clawnify/db` (Drizzle)
  + Tailwind + shadcn/ui. Nothing else.
- **Schema lives in `schema.ts`** (Drizzle DSL) — single source of
  truth. Never write SQL DDL by hand.
- **Queries through `getDB`** — `import { getDB, eq } from
  "@clawnify/db"`. JSON columns auto-serialize.
- **`clawnify.json`** declares framework, env, and public routes.
  Build emits `api.tools[]` from tRPC procedure meta.
- **No `/mcp` endpoint.** The org's MCP gateway projects tools
  from the manifest.
- **Identity flows via `X-Clawnify-*` headers** injected by the
  platform. Read them, don't fake them.

## Stack rationale

One template, one set of decisions. Don't deviate unless the user has
explicitly asked.

## File layout

```
my-app/
  clawnify.json              ← Manifest (validated against /schema/v1/clawnify.json)
  package.json
  tsconfig.json
  vite.config.ts
  tailwind.config.ts
  components.json            ← shadcn/ui config
  index.html
  drizzle.config.ts          ← Points at src/server/schema.ts → outputs to .clawnify/drizzle
  .clawnify/                 ← CLI cache (gitignored)
    drizzle/                 ← drizzle-kit generate output — never hand-edit
      0000_<name>.sql
      meta/
    applied-migrations.json  ← Local-D1 applied tracker
    wrangler.toml            ← Generated per-dev wrangler config
  src/
    server/
      index.ts               ← Hono entry — mounts the tRPC router
      schema.ts              ← Drizzle table definitions — THE schema
      router.ts              ← tRPC router with MCP-tagged procedures
      auth.ts                ← Read X-Clawnify-* identity headers
      env.d.ts               ← Worker bindings types
    client/
      main.tsx               ← React mount point
      app.tsx                ← Root component
      lib/
        trpc.ts              ← Typed tRPC client
      components/            ← shadcn/ui + your components
      styles.css
```

## `schema.ts` — the only place tables exist

```ts
// src/server/schema.ts
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

## `drizzle.config.ts`

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  schema: "./src/server/schema.ts",
  out: "./.clawnify/drizzle",
  dialect: "sqlite",
});
```

You never run `drizzle-kit` manually. The CLI runs it for you:

- **`clawnify dev`** — runs `drizzle-kit generate`, applies any new
  migrations to local D1 via `wrangler d1 execute --local`, tracks
  applied state in `.clawnify/applied-migrations.json` (idempotent
  on rerun).
- **`clawnify deploy`** — same generate step, then auto-includes
  `.clawnify/drizzle/*.sql` in the upload tar (remapped to
  `drizzle/<name>` for the deploy pipeline, which applies them
  transactionally to D1 + Facet and records in `__clawnify_migrations`).

The output dir (`.clawnify/drizzle/`) is gitignored. Migrations
are derived artifacts, not source — never commit them, never edit
them. The single source of truth is `schema.ts`.

## `src/server/index.ts` — Hono entry

```ts
import { Hono } from "hono";
import { trpcServer } from "@hono/trpc-server";
import { appRouter } from "./router";
import { createContext } from "./auth";

const app = new Hono<{ Bindings: Env }>();

app.use("/trpc/*", trpcServer({
  router: appRouter,
  createContext: (opts, c) => createContext(opts, c.env, c.req.raw),
}));

// SPA fallback — Vite-built static assets served by the platform
app.notFound((c) => c.env.ASSETS.fetch(c.req.raw));

export default app;
```

That's it. The Worker entry is thin. No MCP mounting, no inline
route handlers — tRPC owns the API surface.

## `src/server/auth.ts` — identity from headers

```ts
import type { Context } from "hono";

export interface User {
  id: string;
  orgId: string;
  email: string;
  caller: "user" | "agent" | "agent-browser" | "system";
}

export function readUser(req: Request): User | null {
  const id = req.headers.get("X-Clawnify-User-Id");
  const orgId = req.headers.get("X-Clawnify-Org-Id");
  const email = req.headers.get("X-Clawnify-User-Email") ?? "";
  const caller = (req.headers.get("X-Clawnify-Caller") ?? "user") as User["caller"];
  if (!id || !orgId) return null;
  return { id, orgId, email, caller };
}

export function createContext(_opts: unknown, env: Env, req: Request) {
  const user = readUser(req);
  return { env, user };
}
```

Trust these headers — the platform strips client-supplied versions
and reinjects authenticated ones at the perimeter. Never read
`Authorization` or session cookies directly.

## `src/server/router.ts` — tRPC procedures with MCP meta

```ts
import { initTRPC } from "@trpc/server";
import { z } from "zod";
import { getDB, eq, desc, and } from "@clawnify/db";
import * as schema from "./schema";
import type { User } from "./auth";

const t = initTRPC.context<{ env: Env; user: User | null }>().create();

const authed = t.procedure.use(({ ctx, next }) => {
  if (!ctx.user) throw new Error("UNAUTHORIZED");
  return next({ ctx: { ...ctx, user: ctx.user } });
});

export const appRouter = t.router({
  list_notes: authed
    .meta({ mcp: { name: "list_notes", description: "List the user's notes." } })
    .input(z.object({ limit: z.number().int().min(1).max(100).default(50) }))
    .output(z.array(z.object({
      id: z.number(),
      title: z.string(),
      body: z.string(),
      tags: z.array(z.string()),
      createdAt: z.string(),
    })))
    .query(async ({ ctx, input }) => {
      const db = getDB(ctx.env, { schema });
      return db
        .select()
        .from(schema.notes)
        .where(eq(schema.notes.orgId, ctx.user.orgId))
        .orderBy(desc(schema.notes.createdAt))
        .limit(input.limit);
    }),

  create_note: authed
    .meta({ mcp: { name: "create_note", description: "Create a new note." } })
    .input(z.object({
      title: z.string().min(1),
      body: z.string().default(""),
      tags: z.array(z.string()).default([]),
    }))
    .mutation(async ({ ctx, input }) => {
      const db = getDB(ctx.env, { schema });
      const [row] = await db
        .insert(schema.notes)
        .values({ ...input, orgId: ctx.user.orgId })
        .returning();
      return row;
    }),
});

export type AppRouter = typeof appRouter;
```

Patterns to follow:
- **Every state-mutating procedure** that an agent might want to
  call gets `.meta({ mcp: { name, description } })`. The build
  picks these up into `clawnify.json api.tools[]`. The Clawnify
  MCP gateway projects them as MCP tools.
- **Auth via middleware** (`authed`). Never check headers inside
  a procedure body.
- **Filter by `ctx.user.orgId`** on every query. No exceptions.
- **Schema-validated input + output** with Zod. The output schema
  is what the agent sees as the tool's return shape.
- **Procedure names are intent verbs** (`archive_old_notes`,
  `publish_post`), not REST resource shapes (`get_notes`,
  `update_note`). Tools describe what the agent wants to do.

## `src/client` — React + tRPC + shadcn

```tsx
// src/client/lib/trpc.ts
import { createTRPCReact, httpBatchLink } from "@trpc/react-query";
import type { AppRouter } from "../../server/router";

export const trpc = createTRPCReact<AppRouter>();
export const trpcClient = trpc.createClient({
  links: [httpBatchLink({ url: "/trpc" })],
});
```

```tsx
// src/client/app.tsx
import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { trpc, trpcClient } from "./lib/trpc";
import { Button } from "./components/ui/button";

const queryClient = new QueryClient();

export function App() {
  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>
        <NotesList />
      </QueryClientProvider>
    </trpc.Provider>
  );
}

function NotesList() {
  const notes = trpc.list_notes.useQuery({ limit: 50 });
  const create = trpc.create_note.useMutation({
    onSuccess: () => notes.refetch(),
  });
  const [title, setTitle] = useState("");

  return (
    <div>
      {notes.data?.map((n) => <div key={n.id}>{n.title}</div>)}
      <Button onClick={() => create.mutate({ title, body: "" })}>
        Add note
      </Button>
    </div>
  );
}
```

The same tRPC procedures power the agent (via MCP) and the UI (via
typed client). One backend, two front doors.

## `clawnify.json` — the manifest

```json
{
  "$schema": "https://app.clawnify.com/schema/v1/clawnify.json",
  "version": 1,
  "name": "Notes",
  "description": "Take and organize notes. Searchable by tag.",
  "icon": "icon.svg",
  "tags": ["productivity"],
  "app": {
    "framework": "react+hono",
    "database": true,
    "storage": false
  },
  "env": {
    "required": ["OPENROUTER_API_KEY"]
  },
  "api": {
    "public_routes": [
      { "path": "/api/feed", "methods": ["GET"] }
    ]
  }
}
```

Rules:
- **Don't hand-edit `api.tools[]`** — generated at build from tRPC
  `.meta({ mcp: { ... } })` annotations.
- **Public routes are the exception**, not the default. Every tRPC
  procedure is private (auth-required) unless you explicitly add a
  matching path to `public_routes`. Use only for webhooks, RSS,
  embedded feeds.
- **`framework: "react+hono"`** for new apps. Never `preact+hono` or
  `vite-preact` — those are legacy.
- **`database: true`** if the app uses a DB. Schema is generated
  from `schema.ts` via drizzle-kit; no `schema.sql` to ship.

## Migrations

You don't write migration SQL by hand. You don't even run a CLI
command. The workflow is:

1. Edit `src/server/schema.ts` to reflect the new shape.
2. Restart `pnpm dev` (or it auto-restarts on save in some configs).

That's it. `clawnify dev` runs `drizzle-kit generate` under the
hood, applies the diff to local D1, and tracks what's been applied
in `.clawnify/applied-migrations.json` so reruns are no-ops.

On deploy, `clawnify deploy` does the same generate step, then the
deploy pipeline applies the migrations transactionally to live D1
and the Facet (preview tier). Tracked in `__clawnify_migrations`.

For destructive changes (drop column, rename), Drizzle's generator
emits the SQL; the agent should default to additive changes and
only do destructive ones when the user explicitly asks. Renames
are risky on production data — prefer "add new column, backfill,
switch queries, drop old column" over a single-step rename.

## What the agent must NOT do

- Mount `/mcp` manually. The gateway projects tools from the
  manifest.
- Author `schema.sql` by hand. Use `schema.ts`.
- Author migration SQL by hand. The CLI runs `drizzle-kit generate`
  on dev + deploy. Never commit `drizzle/` or `.clawnify/drizzle/`.
- Use `JSON.stringify` on query params. JSON columns auto-serialize
  via Drizzle. The raw API has `json()` as a helper.
- Bind tRPC procedures by REST resource shape (`get_user_by_id`).
  Use intent verbs (`onboard_user`, `archive_old_orders`).
- Skip the `org_id` filter on a user-facing query. Multi-tenant
  leak is the worst-class bug we can ship.
- Read auth from `Authorization` / cookies / `Bearer` tokens. Use
  the `X-Clawnify-*` headers via `auth.ts`.
- Install `drizzle-orm` as a separate top-level dependency.
  `@clawnify/db@^0.4.0` pulls it in transitively; only add it if
  you need to pin a specific version (rare).

## What the agent SHOULD do

- Import everything from `@clawnify/db` where possible
  (`getDB`, `eq`, `and`, `sql`, `sqliteTable`, `text`, `integer`,
  etc. are all re-exported there). Drop to `drizzle-orm` directly
  only for things not re-exported.
- Add `.meta({ mcp: { name, description } })` to every state-
  mutating procedure that the user's agent might want to call.
- Test on the draft URL (`<slug>.draft.clawnify.com`) before
  publishing — the Facet preview tier catches schema drift and
  type errors at build time.
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
