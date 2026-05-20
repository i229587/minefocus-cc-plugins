---
name: nextjs-devtools
description: Use whenever working with Next.js 16+ — user-reported errors (hydration, build, type, runtime), concept/API questions (`'use client'`, Server Actions, RSC, App Router, caching, routing, middleware/proxy, i18n, metadata), upgrades or migrations to Next.js 16, browser-based page verification, or any time you're about to write/edit Next.js code and aren't 100% certain the current API, file name, or pattern. Next.js 16 has breaking changes from training-data Next.js, so guessing is unsafe — trigger even when the user doesn't mention "MCP".
---

# Working in Next.js 16+ projects

## ⚠️ This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.

That warning is non-negotiable. Concretely: **before writing or modifying any Next.js file**, confirm the current shape of the API. Your model weights know a Next.js that no longer exists.

Examples of things that moved or changed in 16+ (you will be wrong if you go on memory):
- `middleware.ts` is now `proxy.ts` (and the matcher format / default behavior changed).
- Default caching for `fetch` and Route Handlers in App Router has flipped — explicit opt-in is now the norm.
- **Cache Components** is a real, separately-enabled feature, not a synonym for the old cache.
- Route segment config conventions and some `metadata` API shapes have shifted.
- Server Actions, `generateStaticParams`, and the streaming/Suspense story have all been tightened.

If a snippet on your tongue looks "obviously correct" — that is exactly the moment to verify it.

## Two sources of truth (use both)

1. **Local bundled docs** — `node_modules/next/dist/docs/`. When present, this is the reference for the version actually installed in *this* project. Use `Glob` to locate the right file (e.g. `**/app/api-reference/file-conventions/proxy.md`), then `Read` it.
2. **`next-devtools-mcp` tools** — when local docs are missing, sparse, or you need real-time state (errors, routes, server actions in *this* running app), use the MCP tools below.

If local docs do not exist, do **not** silently fall back to training data — switch to `nextjs_docs` (Pattern B). Many installs do not ship the docs directory; that is normal, not a reason to give up on verification.

## Tool decision tree

The MCP tools are exposed as `mcp__next-devtools__<name>`. Pick by intent:

| Situation | Tool | Notes |
|---|---|---|
| User reports an error / "it broke" | `nextjs_call` → `get_errors` | Hits the running dev server. Returns build, runtime, and type errors with per-session detail. Run this **first**, before grepping the codebase. |
| Need console output / server logs | `nextjs_call` → `get_logs` | Returns a path to the log file; read it to see browser console + server stdout. |
| "What routes does this app have?" / structure questions | `nextjs_call` → `get_routes` or `get_project_metadata` | Grouped by `appRouter` / `pagesRouter`. Dynamic segments appear as `[param]` / `[...slug]`. |
| Drilling into a specific page | `nextjs_call` → `get_page_metadata` | Returns rendering mode, component tree, and layout chain for that route. |
| Trace a Server Action by ID (from an error or network panel) | `nextjs_call` → `get_server_action_by_id` | Maps action ID → source file + function name. |
| Conceptual question / "is this API still right?" | `nextjs_docs` | Queries the bundled Next.js knowledge base for the installed version. Prefer over web search and over memory. |
| Indexing project for richer context | `nextjs_index` | Run once per session if other tools complain about cold state. |
| "Upgrade me to Next.js 16" / "run the codemods" | `upgrade_nextjs_16` | Drives the official migration codemods. Read its output carefully — it lists every breaking change it touched. |
| "Turn on Cache Components" / opt-in to the new caching model | `enable_cache_components` | Walks through config + code changes; do not hand-roll this. |
| "Does this page render correctly?" / visual verification | `browser_eval` | Backed by Playwright MCP — opens a real browser. Use when a fix is visual or runtime-only. |
| First connection / "MCP isn't responding" | `init` | Re-discovers the running dev server. Run this if other calls return "no instance found". |

If the exact tool name or parameter shape above does not match what you see at call time, search via `ToolSearch` (`select:mcp__next-devtools__<name>`) for the real schema rather than guessing arguments.

## Workflow patterns

### Pattern A — Debug a reported error

**Ideal sequence (dev server reachable):**

1. `nextjs_call` → `get_errors`. Get the raw error first.
2. If the error references a route, follow up with `get_page_metadata` for that route.
3. Read the source file the error names, then fix.
4. If the fix is visual or hydration-related, finish with `browser_eval` to confirm the page actually renders.

**When the dev server is unreachable** (no `pnpm dev` running, or `nextjs_call` returns "no instance"): you can't pull the raw error, but the user is still asking *you* to diagnose. **Do not punt the question back.** Do this instead:

1. Read the project — the stack (`next-themes`, `next-intl`, custom fonts, etc.) and the file most likely to host the issue given the error class.
2. If the symptom matches a well-known pattern in the stack you just read (hydration mismatch + `next-themes` without `suppressHydrationWarning` on `<html>`; `Date.now()` / `Math.random()` / `new Date()` in render; `localStorage` / `window` read during SSR; date formatting that depends on host locale), **state your best diagnosis with confidence, write the fix, and end with a one-line verification step** ("after applying, start `pnpm dev` and run `get_errors` once — should be clean").
3. Only ask the user to bring back tool output if step 2 genuinely cannot narrow down the cause.

The principle: avoid blind training-data guessing, *and* avoid kicking the ball back when you have enough project context to diagnose. One call to `get_errors` is the best signal; project-aware pattern recognition is the second-best; asking the user to fetch data is the last resort, not the default.

### Pattern B — User asks "how do I do X in Next.js?"

1. `Glob` `node_modules/next/dist/docs/**` for the topic. If anything matches, `Read` it.
2. If no local docs (path may not be shipped in every install), call `nextjs_docs` with the user's question.
3. Only after one of (1) or (2) succeeds, write code.
4. Briefly tell the user the source you consulted ("from the proxy.md guide" or "per `nextjs_docs`") so they can verify — say it in your reply, not in a code comment.

### Pattern C — You are about to write Next.js code and feel even a little unsure

Trigger the skill anyway. Concretely: if you are about to type `middleware.ts`, `getServerSideProps`, a Route Handler signature, a `metadata` object, a `fetch` cache option, an `i18n` config, a `generateStaticParams`, or a `'use client'` directive — and you have not consulted the docs **in this session** — stop and run Pattern B before the keystroke. The cost of one tool call is far less than the cost of plausible-looking-but-wrong code that the user has to clean up.

Things that should specifically tip you toward verification:
- The file or function name in your head ends in something that *used* to exist (`middleware.ts`, `_app.tsx`, `getStaticProps`).
- You're writing a config block with magic keys (`experimental.*`, `images.*`, `i18n.*`).
- The user just upgraded Next.js — anything goes for a while after that.

### Pattern D — Upgrade / migration

1. Read `package.json` to confirm the current Next.js version.
2. Call `upgrade_nextjs_16`. Let it run codemods and report.
3. For each breaking change it surfaces, read the corresponding doc (Pattern B) before touching the files yourself.
4. Once the dev server is back up, run `get_errors` to mop up anything the codemods missed.

Don't bump `package.json` by hand for a major Next.js upgrade — you will miss codemods.

### Pattern E — Browser verification

When the user says "look at the page", "check if it renders", or "is the layout right":
1. Make sure the dev server is running. If not, ask the user to `pnpm dev` (or `npm run dev` / `yarn dev` / `bun dev` based on their lockfile).
2. Use `browser_eval` to load the page and inspect.
3. Pair with `get_errors` if anything looks off — visuals lie, errors don't.

## When NOT to use the MCP tools

- Pure JavaScript / React questions with no Next.js framework involvement — no point spinning up MCP.
- The user is in a different framework (Remix, Vite + React, plain React, Astro) — these tools only know Next.js.
- The dev server isn't running and the task is offline doc lookup — `nextjs_docs` and local docs both still work, but `get_errors` / `get_routes` / `browser_eval` will return "no instance found"; in that case, either ask the user to start the dev server or work from docs only.

## Why this matters

Next.js's surface area is **conventional** — file names, default behaviors, magic exports, "framework knows what you mean" defaults. That is exactly the kind of API a language model gets confidently wrong after a version bump:

- A file named `middleware.ts` in a Next.js 16 project may silently do nothing — the rename to `proxy.ts` is the kind of change a model will confidently *undo* if it isn't checking.
- Writing `fetch(url, { cache: 'force-cache' })` "to be safe" can quietly opt you out of the new caching model.
- A `'use client'` placed at the wrong layer breaks streaming in ways that don't surface until production.

The MCP server and the bundled docs exist precisely to prevent this class of error. They are fast, free, and authoritative for the version actually installed. Use them.
