# 02 - tsgo to Vite/Hono/React/Node integration today

Type: research
Status: resolved

Blocked by:

## Question

What is the current state of TypeScript 7 (the native Go compiler, `tsgo`) integration with the exact stack we've chosen: Vite + React SPA, Hono, and plain Node? What works today, what requires `tsc` fallback, and what are the concrete setup steps?

Concretely:

- Does Vite work with `tsgo` for type-checking and/or transpilation, or does the Vite build still rely on esbuild/swc/`tsc`? Is there a community plugin?
- Does Hono type-check cleanly under `tsgo`? Any known issues?
- Node runtime: does the current Node LTS run `tsgo`-compiled output, or do we need `--experimental-strip-types` / `tsx` for the poller? What Node version is required for `tsgo` itself?
- Editor/IDE type-checking: does `tsgo` plug into the language server, or does the editor still use `tsc`?
- The minimal working `tsconfig` for a `tsgo`-first project with `tsc` as fallback.
- Known rough edges / unsupported features in `tsgo` that would affect this project.

## Notes for the researcher

Use `/research` workflow. Primary sources: the TypeScript native-port announcement/docs, `tsgo` changelog, Vite/Hono issue trackers, and current tooling blog posts. Capture exact versions and commands.

## Answer

Full report in `research/02-tsgo-toolchain-integration.md` (286 lines). Key facts:

- **TS 7.0 shipped stable 8 Jul 2026** as the native Go compiler, installed as `npm install -D typescript`, invoked as **`tsc`** (no `tsgo` binary in stable — the name survives only in the nightly `@typescript/native-preview`). Engines `node >=16.20.0`.
- **Vite never used `tsc` for transpilation** (Oxc/Rolldown, no type checking). Recommended TS7 use: swap `tsc --noEmit` / `--watch` scripts for TS 7 = native checking. No official Vite plugin; `vite-plugin-checker` tsgo support is open issue #516.
- **Hono type-checks cleanly under tsgo** — Hono's own CI pins `@typescript/native-preview`. No breakage found.
- **Node backend: no build step needed.** Node runs `.ts` natively via type stripping (stable since v24.12/v25.2). Setup: `node --watch src/index.ts` for dev, `tsgo --noEmit` (i.e. `tsc`) for checking; `tsx` only for non-erasable syntax.
- **Editors:** native Go LSP (VS Code `TypeScriptTeam.native-preview` ext, WebStorm 2026.2 built-in). React `.tsx` fully supported; Vue/Astro/Svelte embedded tooling still on TS 6.
- **tsconfig:** base = `target:esnext, strict, skipLibCheck, isolatedModules, verbatimModuleSyntax, noEmit, types:[]`. Frontend adds `moduleResolution:bundler, jsx:react-jsx, types:["vite/client","react","react-dom"]`. Backend adds `module:nodenext, erasableSyntaxOnly, rewriteRelativeImportExtensions, types:["node"]`.
- **Biggest rough edge:** no programmatic API until 7.1 — typescript-eslint, ts-morph, ts-jest, tsup `--dts` must stay on TS 6 via `typescript@npm:@typescript/typescript6`. Run `npx tsc --noEmit` (TS7) + `npx tsc6 --noEmit` (fallback) in CI. Diagnostic drift between the two compilers is real — diff, don't silence.
