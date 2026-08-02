# Research: TypeScript 7 / `tsgo` integration with a Vite + React + Hono + Node stack

- **Date of research:** 2 August 2026
- **Context:** Vite + React SPA (frontend workspace), Hono backend API (backend workspace), plain Node.js runtime, two workspaces in one repo.
- **Status:** Research only. No application code was written.

## Executive summary

TypeScript 7.0 shipped stable on **8 July 2026** as a native (Go) port of the compiler, codename "Project Corsa". It is installed exactly like before — `npm install -D typescript` — and invoked as **`tsc`**. The `tsgo` binary now only exists in the nightly preview package `@typescript/native-preview`.

For this stack, the picture is: **Vite never used `tsc` for transpilation and still doesn't** (it transpiles with Oxc/Rolldown); `tsgo` slots in cleanly as the *type-checker* (`tsc --noEmit` replacement). **Hono type-checks cleanly under `tsgo`** — its own CI pins `@typescript/native-preview`. For the **Node backend**, you don't need `tsgo` to emit JS at all: Node.js has run `.ts` natively via type stripping since v22.18/v23.6 (stable since v24.12/v25.2), so the recommended setup is `node` + `tsgo --noEmit` for checking, with `tsx` only if you need non-erasable syntax. **Editors** now use a native LSP (VS Code extension, built-in in WebStorm 2026.2 / upcoming in VS Code proper), though embedded-language tooling (Vue/Astro/Svelte/MDX/Angular templates) still requires TS 6.0 until the 7.1 programmatic API lands.

Key caveat for a monorepo: TS 7.0 ships **no programmatic API** (deferred to 7.1). Tooling that calls `import * as ts from "typescript"` (typescript-eslint, ts-morph, ts-patch, ts-jest, tsup `--dts`, API Extractor, TypeDoc, Volar-style plugins) must stay pinned to the JS-based TS 6.0 via the `@typescript/typescript6` alias package.

---

## 1. What TypeScript 7 / `tsgo` is, and how to install/run it

### What it is

- TypeScript 7.0 is a **native port of the compiler and language service from the self-hosted JavaScript ("Strada") codebase to Go**, codename **"Project Corsa"**. It is a *port*, not a rewrite: type-checking semantics are structurally identical to TS 6.0. Performance comes from native code speed + shared-memory multithreading (parsing, type-checking, and emitting are parallelized). [1][2][3]
- Microsoft's published benchmarks: full builds typically **8–12× faster** (VS Code: 125.7s → 10.6s, ~11.9×; at `--checkers 8`: 16.7×), with **6–26% less peak memory**. [1][3]
- Timeline: native-port announced 11 Mar 2025 ("A 10x Faster TypeScript") [4]; public previews from May 2025 via `@typescript/native-preview` [5]; TS 6.0 (last JS-based release, "bridge" release) 23 Mar 2026 [6]; TS 7.0 Beta 21 Apr 2026 [7]; **RC 18 Jun 2026** [8]; **stable 8 Jul 2026** [3].

### Installing and invoking

- **Stable (current):** `npm install -D typescript` → the Go-native binary ships under the normal **`tsc`** name. Run `npx tsc --noEmit`. There is no `tsgo` command in the stable package. [3][8][9]
- The npm package wraps native per-platform binaries. The `typescript@7.0.2` package (current `latest`, published ~Jul 2026) declares `"bin": { "tsc": "bin/tsc" }` and pulls platform binaries as optional dependencies (e.g. `@typescript/typescript-linux-x64`, `...-darwin-arm64`, etc.). [10] The `bin/tsc` entry is a small JS shim that spawns the native binary — confirmed by a typescript-go bug report showing `bin/tsgo.js` doing `execFileSync` against `.../lib/tsgo(.exe)`. [11]
- **Nightly preview (bleeding-edge only):** `npm install -D @typescript/native-preview`, invoked as **`tsgo`** (`npx tsgo --help`). This is the only place the name `tsgo` survives; it is for previewing what comes *after* 7.0. Nightlies will move back under `typescript@next`. [3][12][13]
- **Side-by-side with TS 6.0 (the officially supported coexistence):** `npm install -D typescript@npm:@typescript/typescript6` installs a `tsc6` binary plus a re-export of the TS 6.0 API. The TS 7 blog explicitly recommends this arrangement so that API-consuming tools (e.g. typescript-eslint) keep working while `tsc` is TS 7. [3][8]

### Minimum Node version

- The `typescript@7.0.2` npm manifest declares **`"engines": { "node": ">=16.20.0" }`**. [10] The compiler itself is a native binary, but the npm `tsc`/`tsgo` CLI is launched via a Node shim, so Node is still required to run the command. [11] (Exact Node-version ceiling for the shim beyond 16.20.0 could not be independently verified; no higher engine constraint was found.)

---

## 2. Vite + `tsgo` today

### Does Vite use `tsgo`/`tsc` for transpilation? No — it never did.

- Vite docs (current, v8.1.x, fetched 2 Aug 2026): **"Vite only performs transpilation on `.ts` files and does NOT perform type checking."** Vite transpiles TypeScript with the **Oxc Transformer** (Rolldown is used for dependency pre-bundling). HMR updates reflect in under 50 ms. [14]
- Earlier Vite versions used esbuild; the current docs and the releases page confirm the current engine is **Oxc** (the vite-ecosystem references Vite 8.1, 7.3, 8.0, 6.4 as supported lines). [14][15] One real-world write-up from May 2026 described Vite as "esbuild and Rolldown" [16]; the official docs as of Aug 2026 say Oxc Transformer, which is the more authoritative statement of current state. Either way: **neither Vite nor any Vite plugin invokes `tsc`/`tsgo` for transpilation.**

### Vite's recommended type-checking workflow

The Vite docs recommend **decoupling type checking from the transform pipeline** [14]:

- **Production builds:** run `tsc --noEmit` *in addition to* `vite build`.
- **Dev:** run `tsc --noEmit --watch` in a separate process, or use `vite-plugin-checker` to surface type errors in the browser.

Because `tsc` in TS 7 *is* the native compiler, these documented steps become `tsgo`-powered simply by installing `typescript@7` — **no Vite plugin is needed for type checking**. That is the current recommended way to use `tsgo` in a Vite project: point your existing `tsc --noEmit` / `tsc --noEmit --watch` scripts at TS 7.

### Is there a Vite plugin that drives `tsgo`?

- **No official plugin.** `vite-plugin-checker` (the de-facto browser-error checker) tracks "Support TypeScript Native (`tsgo`)" in **open issue #516** (opened 23 May 2025; last updated Mar 2026). The blocker stated in-thread: tsgo only exposes a CLI, not the compiler API, so checker can't drive it programmatically. [17]
- A small community plugin exists: **`@mkljczk/vite-tsgo-checker`** (spawns the `tsgo` CLI from a Vite plugin), shared in that same issue thread (Mar 2026). Not affiliated with the Vite team; do not treat as primary. [17][18]
- Practical guidance from a real Vite + Nx migration (May 2026): split type checking out of the Vite build into a separate, cacheable `typecheck` task running `tsgo --noEmit -p <tsconfig>`, and set `skipTypeCheck: true` on the Vite build executor so you don't type-check twice. Cold pipeline went from 53.0s (tsc) to 23.8s (tsgo) on their largest app. [16]

### Vite-specific tsconfig notes (relevant to `tsgo`)

- `compilerOptions.isolatedModules` should be `true` (Oxc-only transpilation can't handle const-enum / implicit type-only imports; `tsgo` won't enforce this for you since it does full checking, but the flag keeps both in agreement). [14]
- Vite starter templates ship `skipLibCheck: true`. [14]
- Vite reads `jsx`, `jsxImportSource`, `verbatimModuleSyntax`, etc. from tsconfig for the Oxc transformer, but **ignores `target`** (dev target defaults to `esnext`; build target via `build.target`). [14]
- The TS 7 defaults change (section 6) apply: set `"types": ["vite/client", ...]` explicitly since `types` now defaults to `[]`. [3][14]

---

## 3. Hono under `tsgo`

### Does Hono type-check cleanly under `tsgo`?

**Yes — and the strongest evidence is Hono's own repository.** As of the fetched `honojs/hono` `package.json` (main branch, 2 Aug 2026; hono version 4.12.33), the devDependencies include **both**:

```json
"@typescript/native-preview": "7.0.0-dev.20260210.1",
"typescript": "^6.0.3",
```

Hono runs its own type checks against the native preview (tsgo) in CI. [19] I found **no open issue in `honojs/hono` reporting Hono types breaking under `tsgo`/TS 7** in the issue-tracker search ("tsgo" scoped to the repo returns only the two issues below). [20]

### Relevant Hono issue-tracker context

- **honojs/hono #3869 — "Hono Type Inference is taking too long during builds"** (opened 30 Jan 2025, enhancement): Hono's route/validator generics are heavy; type-check time is a known cost. This predates tsgo but is the reason Hono tracks compiler performance. [20][21]
- **honojs/hono #4159 — "CI: add typescript-go type perf benchmark"** (opened 24 May 2025, closed): Hono maintainers proposed adding tsgo type-perf numbers to their existing type benchmarks. [22]

### Caveat

Hono is one of the most generic-heavy libraries in the ecosystem (its type inference is a known slow point [21]); it benefits disproportionately from `--checkers` parallelism. Because TS 7.0 is a faithful port, Hono's types should resolve identically, but the practical recommendation is the standard one: **run `tsgo --noEmit` side-by-side with `tsc --noEmit` and diff diagnostics before switching.** Real-world migrations have reported small diagnostic mismatches between the two compilers (e.g. tsgo flagging overload errors `tsc` missed) — file those as issues rather than silencing them. [16][23]

---

## 4. Running a TypeScript Node backend (Hono) in Aug 2026

### Does `tsgo` emit runnable JS for Node?

Yes — TS 7.0's JS emit ("Emit (JS output)") is marked **done** in the typescript-go README [24], and `es2015` is the lowest supported `target` in 7.0 [3]. But for a modern Node backend you generally **don't need emit at all**:

### Node.js native TypeScript (type stripping) — the recommended path

- **Node runs `.ts`/`.mts`/`.cts` files natively by stripping erasable type syntax.** Milestones (Node.js docs, v26.5.1): flag introduced v22.6.0; **enabled by default since v22.18.0 / v23.6.0**; experimental warning removed v24.3.0 / v22.18.0; **stable since v24.12.0 / v25.2.0**; the separate `--experimental-transform-types` flag (for enums/namespaces) was removed in v26.0.0. [25]
- Node's built-in stripping ("Amaro"/SWC-based) **does not type-check** — it replaces types with whitespace. Type checking must be a separate `tsc`/`tsgo --noEmit` step (in CI and/or `--watch`). [26]
- Recommended tsconfig for running `.ts` directly under Node (Node docs): `noEmit`, `target: "esnext"`, `module: "nodenext"`, `rewriteRelativeImportExtensions: true`, `erasableSyntaxOnly: true`, `verbatimModuleSyntax: true`. [26]
- What's *not* supported by stripping and needs a runner/compiler: `enum`, parameter properties, namespaces with runtime code, import aliases, legacy decorators. [25][26]

### Recommended setup for a small Hono + Node service (Aug 2026)

Based on Node's official guidance [25][26], Hono's own Docker/build docs [27], and current write-ups [28]:

- **Development:** `node --watch src/index.ts` (zero build step), or `tsx watch src/index.ts` if you need non-erasable syntax or extra features.
- **Type checking:** `tsgo --noEmit` (or `tsc --noEmit`) as its own script/CI step — this is where TS 7 replaces nothing functionally but makes the check ~10× faster.
- **Production:** either run `node dist/index.js` after a `tsc`/`tsgo` build, **or** run `node src/index.ts` directly (type stripping is stable and this is now a mainstream 2026 pattern for erasable-only code). [28]
- **Hono specifics:** Hono runs on Node ≥ 18.14.1 via the `@hono/node-server` adapter (`serve(app)`); the `create-hono` nodejs template is the starting point. Hono's own Dockerfile example builds to `dist/` and runs `node /app/dist/index.js`. [27] Hono's package.json declares `engines: node >=16.9.0`. [19]
- `tsx` remains the fallback runner when you need `enum`/decorators or legacy transpilation, and `tsc` remains the emit path if you target old Node versions or need `.d.ts`. [28]

---

## 5. Editor/IDE integration: does `tsgo` power the language server?

**Yes — the TS 7 language server is the native (Go) implementation speaking standard LSP.** It replaces the old JS `tsserver`/TS Server protocol.

- **VS Code:** dedicated extension **"TypeScript 7" (`TypeScriptTeam.native-preview`)** on the marketplace. Install it, then either run the command `TypeScript: Enable TypeScript 7` or set `"js/ts.experimental.useTsgo": true` in settings. It "will automatically become the default experience"; support ships inside VS Code itself "in the coming weeks" after 8 Jul 2026. You can toggle back to TS 6.0 with `Disable TypeScript 7 Language Server`. [3][29]
- **LSP protocol:** the native server speaks standard LSP over stdio — run `tsc --lsp --stdio` (nightly: `tsgo --lsp`). This is a protocol-level change from tsserver. [13][30]
- **Visual Studio:** 2026 versions enable TS 7 automatically based on workspace. [3][31]
- **WebStorm / JetBrains:** WebStorm has "basic TypeScript-Go support" (point it at `@typescript/native-preview` or the cloned typescript-go repo) since **2025.2** [32][33]; **WebStorm 2026.2 (July 2026) ships built-in TypeScript 7 support** [34]. JetBrains tracks native-preview support in **WEB-72048** [35].
- **Zed:** a `zed-extensions/tsgo` extension exists and has a PR to switch to TypeScript 7's native LSP built into `tsc` [36].
- **What still uses TS 6.0 in the editor:**
  - **Embedded-language tooling** — "workflows that use Vue, MDX, Astro, Svelte, and others will likely not yet be able to leverage TypeScript 7... tools (such as Volar) which embed TypeScript into their own compilers and language services can only currently rely on TypeScript 6.0." The TS team expects this to be fixed after the 7.1 API. For a React SPA (no embedded-language plugins) this is a non-issue. [3]
  - **TS server plugins** (the old plugin mechanism) don't exist in the native server; there is no plugin API yet. [13][37]
  - **typescript-eslint** type-aware linting still runs on the JS API → keep `typescript` aliased to `@typescript/typescript6`. [3][8]
  - In practice, editor correctness across WebStorm/VS Code for a *plain .ts/.tsx React + Hono* project is the supported, mainstream case for TS 7 as of Jul–Aug 2026. [3][34]

---

## 6. Minimal `tsconfig.json` for a `tsgo`-first project (with `tsc` fallback)

### TS 7 default/breaking changes you must get right (from the 7.0 announcement) [3]

| Option | TS 7 default/behavior |
| --- | --- |
| `strict` | `true` by default |
| `module` | defaults to `esnext` |
| `target` | defaults to latest stable ES version before `esnext` |
| `rootDir` | defaults to `./` — set it explicitly if source lives under `src/` |
| `types` | defaults to `[]` — global `@types/*` are no longer auto-included; list them or set `["*"]` |
| `baseUrl` | **removed** (hard error) — `paths` must be relative |
| `moduleResolution: node/node10` | **hard error** — use `nodenext` or `bundler` |
| `module: amd/umd/systemjs/none` | **hard error** — use `esnext`/`preserve` |
| `target: es5` | **hard error** — `es2015` is the floor |
| `esModuleInterop`/`alwaysStrict` | cannot be disabled |

### Recommended base config (shared across both workspaces)

```jsonc
// tsconfig.base.json
{
  "compilerOptions": {
    "target": "esnext",
    "strict": true,
    "skipLibCheck": true,
    "isolatedModules": true,          // keep in agreement with Vite's Oxc transpiler
    "verbatimModuleSyntax": true,     // required for Node type-stripping + Oxc
    "noEmit": true,                   // tsgo is used for checking; runtimes/builders emit
    "types": []                       // each workspace lists its own (see below)
  }
}
```

**Frontend workspace** (`tsgo --noEmit` + Vite) [3][14]:

```jsonc
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "types": ["vite/client", "react", "react-dom"]
  },
  "include": ["src"]
}
```

**Backend workspace** (Node type-stripping + Hono) [25][26]:

```jsonc
{
  "extends": "../tsconfig.base.json",
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "erasableSyntaxOnly": true,          // only erasable syntax — matches node's stripper
    "rewriteRelativeImportExtensions": true, // enable .ts→.js rewrite in source imports
    "types": ["node"]
  },
  "include": ["src"]
}
```

### Running `tsgo` with `tsc` fallback

Keep both compilers installed (TS 7 stable + `typescript@npm:@typescript/typescript6` for `tsc6`) [3], and run both in CI until diagnostics match:

```bash
npx tsc --noEmit      # TS 7 native (tsgo build)
npx tsc6 --noEmit     # TS 6 JS compiler, for API-consuming tooling / fallback
```

`npx tsc --noEmit -p apps/foo/tsconfig.json` is the swap point if you ever want to revert. [16] Optionally tune parallelism: `--checkers N` (default 4), `--builders N` (project-reference builds), `--singleThreaded`. [3]

---

## 7. Known limitations / rough edges relevant to a small React + Hono project

1. **No programmatic API in 7.0.** Any tool that does `import ts from "typescript"` (typescript-eslint type-aware rules, ts-morph, ts-patch/ttypescript custom transformers, ts-jest, tsup `--dts`, API Extractor, TypeDoc) cannot use TS 7. Deferred to **7.1**. This is the single biggest reason to keep the `@typescript/typescript6` alias. [3][8][13][38][39]
2. **JS/JSDoc behavior intentionally differs.** TS 7 rewrote JavaScript file support; `@enum`, `@class`/`@constructor`, Closure function-type syntax, standalone `?` types, constructor functions/expandos, etc. are removed or changed. `.d.ts` emit from `.js` is intentionally *not* byte-identical. If the project is pure `.ts` (React + Hono), this is a non-issue; with `.js`/JSDoc it needs a migration audit. [40][41][42]
3. **Diagnostic drift is real.** Multiple real-world reports of tsgo and tsc 6 disagreeing on edge-case errors (tsgo catching overload errors tsc 6 missed; variance differences) [16][23][43]. Not a blocker, but the recommended workflow is side-by-side diffing in CI for a transition period. [44]
4. **`rootDir`/`types` defaults surprise.** Projects coming from older TS will suddenly error on `types` (now `[]`) and cross-directory `rootDir`. Easy one-line fixes (section 6), but expect a burst of errors on first `tsgo` run. [3][8]
5. **`--incremental`/`tsbuildinfo` interop.** `tsgo` and `tsc` write tsbuildinfo in "similar but not identical" formats; a stale TS 6 cache forces a first full rebuild. Clear `*.tsbuildinfo` on migration. [16]
6. **`skipLibCheck` conflict-reporting change.** In TS 7, conflicting declarations now error at *all* contributing sites, so some `.d.ts` conflicts previously swallowed by `skipLibCheck` may now surface in your code. [40]
7. **Emit (JS) caveats that mostly don't apply here.** TS 7's JS emit is done [24] but downlevel support bottoms out at `es2015` [3], and there's no `es5`. Since Vite (Oxc) and Node (type stripping) do the transpiling in this stack, `tsgo` is used for checking only — avoiding emit risk entirely. [14][26]
8. **Template literal Unicode behavior change.** Inferred `Head`/`Rest` of template literal types now consume whole code points instead of UTF-16 units (emoji no longer split into surrogates). Only matters for type-level string manipulation that modeled UTF-16 — unusual in app code. [3][40]
9. **`--watch` rebuild is new.** TS 7's watch mode was rebuilt on a Go port of Parcel's watcher and is "done" [24], but if you rely on old watch quirks, validate. [3]
10. **Preview-package rough edges (nightly only).** Platform-binary path issues with pnpx (long-path ENOENT, fixed via #2448) [11]; not applicable to the stable `typescript` package.
11. **Editor ecosystem lag for non-`.ts` files.** Vue/MDX/Astro/Svelte/MDX/`template` blocks still require TS 6.0; React `tsx` is fully supported by TS 7's language server. [3][29]

**Bottom line for the vinted stack (as of 2 Aug 2026):** install `typescript@7` (stable), run `tsc --noEmit` (i.e. tsgo) as the type-check gate in both workspaces, transpile the SPA with Vite (Oxc) and run the Hono API directly under Node's native type stripping (`node src/index.ts`), keep `tsx` as a dev fallback only if non-erasable syntax is needed, and keep a `typescript@npm:@typescript/typescript6` alias only for any tooling that still imports the JS compiler API (e.g. typescript-eslint). Use the two tsconfigs in section 6.

---

## Sources

**Official Microsoft / TypeScript:**

1. Anders Hejlsberg, "A 10x Faster TypeScript", TypeScript DevBlog, 11 Mar 2025 — https://devblogs.microsoft.com/typescript/typescript-native-port/
2. Daniel Rosenwasser, "Announcing TypeScript 7.0 Beta", 21 Apr 2026 — https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/
3. Daniel Rosenwasser, "Announcing TypeScript 7.0", 8 Jul 2026 — https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/
4. "A 10x Faster TypeScript" (Project Corsa announcement) — https://devblogs.microsoft.com/typescript/typescript-native-port/
5. "Announcing TypeScript Native Previews", ~May 2025 — https://devblogs.microsoft.com/typescript/announcing-typescript-native-previews
6. Daniel Rosenwasser, "Announcing TypeScript 6.0", 23 Mar 2026 — https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/
7. "Announcing TypeScript 7.0 Beta", 21 Apr 2026 — https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/
8. "Announcing TypeScript 7.0 RC", 18 Jun 2026 — https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-rc/
9. npm — `typescript@7.0.2` registry manifest (fetched 2 Aug 2026) — https://registry.npmjs.org/typescript/latest
10. `@typescript/native-preview` npm page (README: "For TypeScript 7.0 RC and later, use `tsc`... For other builds, use the `tsgo` command") — https://www.npmjs.com/package/@typescript/native-preview
11. microsoft/typescript-go issue #2441 "using tsgo fails with pnpx" (confirms JS wrapper `bin/tsgo.js` spawns native binary), closed Jan 2026 — https://github.com/microsoft/typescript-go/issues/2441
12. microsoft/typescript-go README ("For TypeScript 7.0 RC and later, the command name is `tsc`"; "What Works So Far" table; repo will be merged into microsoft/TypeScript) — https://github.com/microsoft/typescript-go
13. microsoft/typescript-go discussion #455 "API story" — https://github.com/microsoft/typescript-go/discussions/455
14. Vite docs, "Features — TypeScript" (v8.1.5, fetched 2 Aug 2026): Oxc Transformer, no type checking, `tsc --noEmit` recommendations, `isolatedModules`, `types: ["vite/client", ...]` — https://vite.dev/guide/features.html#typescript
15. Vite releases page (supported versions 8.1/8.0/7.3/6.4) — https://vite.dev/releases
16. Uday Nayak, "Migrating to tsgo (TypeScript 7 Native Preview) in a Vite + NX Monorepo", 20 May 2026 — https://www.udaynayak.com/blog/migrating-from-tsc-to-tsgo-typescript-7-native-preview
17. fi3ework/vite-plugin-checker issue #516 "Support TypeScript Native (`tsgo`)" (open; community plugin `@mkljczk/vite-tsgo-checker` linked) — https://github.com/fi3ework/vite-plugin-checker/issues/516
18. `@mkljczk/vite-tsgo-checker` (referenced in #17; npm page returned 403 at fetch time, cited via the issue) — https://www.npmjs.com/package/@mkljczk/vite-tsgo-checker
19. honojs/hono `package.json` (main, fetched 2 Aug 2026; devDeps include `@typescript/native-preview@7.0.0-dev.20260210.1` and `typescript@^6.0.3`; engines node >=16.9.0; hono 4.12.33) — https://raw.githubusercontent.com/honojs/hono/main/package.json
20. GitHub issue search `repo:honojs/hono tsgo` (2 results: #3869, #4159; no Hono-types-broken-under-tsgo issues) — https://github.com/search?q=repo%3Ahonojs%2Fhono+tsgo&type=issues
21. honojs/hono issue #3869 "Hono Type Inference is taking too long during builds" — https://github.com/honojs/hono/issues/3869
22. honojs/hono issue #4159 "CI: add typescript-go type perf benchmark" (closed) — https://github.com/honojs/hono/issues/4159
23. "I Ran TypeScript 7's Native Compiler (tsgo) on Our Monorepo", DEV Community / var.gg, 29 Jun 2026 (diagnostics mismatch; ~3.2× at small scale) — https://dev.to/curioustore_48788631d0e2e/i-ran-typescript-7s-native-compiler-tsgo-on-our-monorepo-4d6
24. microsoft/typescript-go README "What Works So Far" (Emit done; Language service in progress; API not ready) — https://github.com/microsoft/typescript-go
25. Node.js docs, "Modules: TypeScript" (v26.5.1): stripping timeline (v22.18/v23.6 default-on, v24.12/v25.2 stable, v26.0 removed transform flag) — https://nodejs.org/api/typescript.html
26. Node.js docs, "Running TypeScript Natively" (recommended tsconfig: `noEmit`, `erasableSyntaxOnly`, `verbatimModuleSyntax`, `rewriteRelativeImportExtensions`, `module: nodenext`) — https://nodejs.org/learn/typescript/run-natively
27. Hono docs, "Getting Started — Node.js" (adapter `@hono/node-server`, Node ≥18.14.1, Dockerfile `node /app/dist/index.js`) — https://hono.dev/docs/getting-started/nodejs
28. "Running TypeScript Directly in 2026 — tsx, ts-node, and Native Node.js", Webcoderspeed, Mar–Aug 2026 — https://webcoderspeed.com/blog/scaling/tsx-runtime-2026
29. VS Code Marketplace, "TypeScript 7" extension (`TypeScriptTeam.native-preview`, `js/ts.experimental.useTsgo`) — https://marketplace.visualstudio.com/items?itemName=TypeScriptTeam.native-preview
30. oraios/serena issue #1402 (tsgo LSP via `tsgo --lsp` / stable `tsc --lsp --stdio`; closed Jul 2026) — https://github.com/oraios/serena/issues/1402
31. "TypeScript 7 native preview in Visual Studio 2026", Microsoft Developer Blog — https://developer.microsoft.com/blog/typescript-7-native-preview-in-visual-studio-2026
32. WebStorm Blog, "WebStorm 2025.2: TypeScript-Go Language Server Support...", 7 Aug 2025 — https://blog.jetbrains.com/webstorm/2025/08/webstorm-2025-2
33. WebStorm Help, "TypeScript — TypeScript Native Previews (TypeScript-Go)" (2026.2 docs, fetched 2 Aug 2026) — https://www.jetbrains.com/help/webstorm/typescript-support.html
34. "WebStorm 2026.2 supports TypeScript 7...", 22 Jul 2026 — https://alternativeto.net/news/2026/7/webstorm-2026-2-supports-typescript-7-and-adds-native-copilot-integration/
35. JetBrains YouTrack WEB-72048 "Support TypeScript Native Previews (@typescript/native-preview)" (youtrack.jetbrains.com returned connection errors at fetch time; URL referenced by JetBrains/WebStorm docs and community posts) — https://youtrack.jetbrains.com/projects/WEB/issues/WEB-72048
36. zed-extensions/tsgo issue #56 "Support for TypeScript 7 Release" (PR #57 switches to `typescript` v7's native LSP) — https://github.com/zed-extensions/tsgo/issues/56
37. vuejs/language-tools issue #5381 (Vue/TS7 tracking; blockers: JS-plugin interop, no plugin API; Microsoft "resetting" language-service issues) — https://github.com/vuejs/language-tools/issues/5381
38. ishu.dev, "TypeScript 7.0: Go Compiler, 10x Faster Builds, and Everything That Just Broke", 10 Jul 2026 — https://ishu.dev/post/typescript-7-go-native-compiler-2026-07-10
39. byteiota.com, "TypeScript 7: Go Compiler, Breaking Changes, Migration Guide", 3 Jun 2026 (ts-morph/tsup/ts-jest/typescript-eslint breakage list) — https://byteiota.com/typescript-7-go-compiler-breaking-changes-migration-guide/
40. microsoft/typescript-go `CHANGES.md` (intentional Strada→Corsa differences: JS/JSDoc removals, `skipLibCheck` conflict reporting, template-literal Unicode) — https://raw.githubusercontent.com/microsoft/typescript-go/main/CHANGES.md
41. Daniel Rosenwasser, "Progress on TypeScript 7 — December 2025", 2 Dec 2025 (JS support rewritten; API gaps; ts5to6 tool) — https://devblogs.microsoft.com/typescript/progress-on-typescript-7-december-2025/
42. "Evaluating typescript-go (tsgo) for Our Monorepo" (Seed notes; gaps incl. `disableSourceOfProjectReferenceRedirect` bug #506, JS emit esnext-only at the time) — https://seed.hyper.media/hm/z6MkuBbsB1HbSNXLvJCRCrPhimY6g7tzhr4qvcYKPuSZzhno/notes/evaluating-typescript-go-tsgo-for-our-monorepo
43. var.gg, "TypeScript 7 tsgo: A Real Benchmark vs tsc 6.0.3", 12 Jul 2026 — https://var.gg/en/blog/typescript-native-tsgo
44. "TypeScript 7 Native Compiler Guide 2026: tsgo, Migration, Tooling, CI", ToolsMint, 7 Jun 2026 — https://www.toolsmint.com/learn/typescript-7-native-compiler-migration-guide-2026

### Could not verify / notable gaps

- **Minimum Node for the stable package:** only the npm `engines` field (`>=16.20.0`) was verifiable; the exact floor at which the native CLI shim breaks was not tested. [9]
- **JetBrains WEB-72048** page itself was unreachable at fetch time (connection errors); its existence and purpose are corroborated by WebStorm docs [33] and the 2026.2 news [34].
- **`@mkljczk/vite-tsgo-checker` npm page** returned HTTP 403 at fetch time; existence is corroborated by the maintainer's own comment in vite-plugin-checker#516 [17].
- **Hono-type-checks-under-tsgo** is inferred from Hono's CI devDependencies [19] and the absence of contrary issues [20], not from a published benchmark. No Hono maintainer statement specifically about TS 7 compatibility was found.
- All URLs were consulted between 2026-07 and 2026-08-02 unless otherwise noted; "current state" claims are accurate as of 2 Aug 2026.
