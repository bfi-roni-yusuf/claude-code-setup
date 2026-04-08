---
name: ts-init
description: Scaffold a new TypeScript project with rulebook-compliant config
---

# /ts-init — TypeScript Project Scaffolder

Templates: `~/.claude/skills/ts-init/templates/`

## Before you start

1. **Determine the latest Node.js LTS version.** Run `node --version` or check nodejs.org. Use the major version number (e.g., `24`) for `.nvmrc` and `engines`. Determine the matching ES target (e.g., Node 24 → ES2025, Node 26 → ES2026).
2. **Check Context7 MCP** for current stable versions of: typescript, eslint, typescript-eslint, vitest, @vitest/coverage-v8, zod, pnpm. Backend: @swc/cli, @swc/core, tsx. Frontend: next, react, react-dom, @types/react, @types/react-dom. Library: tsup. Monorepo: turbo.

## Step 1: Ask project details

Ask the user:

1. **Project name** (kebab-case, e.g., `my-api`)
2. **Project type**: `backend` | `frontend` | `library` | `monorepo`
3. **Formatter**: `biome` | `prettier`

## Step 2: Scaffold base config

Read each template from `templates/base/`, replace `{{placeholders}}`, write to project:

| Template file | Target path | Notes |
|---------------|-------------|-------|
| `tsconfig.json` | `./tsconfig.json` | Replace `{{es-target}}` with matching ES version. Frontend: set `module` to `"ESNext"` and `moduleResolution` to `"Bundler"`, remove `outDir`/`rootDir`, add `"jsx": "preserve"` |
| `eslint.config.js` | `./eslint.config.js` | |
| `vitest.config.ts` | `./vitest.config.ts` | |
| `package.json` | `./package.json` | Replace `{{node-lts-version}}` with latest LTS major. Fill version `{{placeholders}}` with Context7. Formatter placeholders: biome → `biome format .` / `biome format . --check`, prettier → `prettier --write .` / `prettier --check .`. Add `@biomejs/biome` or `prettier` to devDependencies accordingly |
| `npmrc` | `./.npmrc` | |
| `nvmrc` | `./.nvmrc` | Replace `{{node-lts-version}}` with latest LTS major (from step 0) |
| `ci.yml` | `./.github/workflows/ci.yml` | |
| `env.ts` | `./src/config/env.ts` | |
| `app-error.ts` | `./src/errors/app-error.ts` | |

Create empty directories: `src/domain/`, `src/services/`, `src/utils/`

## Step 3: Scaffold variant files

### If `backend`

Read `templates/backend/.swcrc` → `./.swcrc`. Replace `{{es-target-lowercase}}` with lowercase ES target (e.g., `es2025`).

Override package.json:
- Change `"build"` script to `"swc src -d dist --strip-leading-paths"`
- Add `"dev": "tsx watch src/main.ts"` to scripts
- Add `@swc/cli`, `@swc/core`, `tsx` to devDependencies (versions from Context7)

Note: `.swcrc` uses `"module": { "strict": true }` — this enforces strict ESM semantics, matching `"type": "module"` in package.json.

### If `frontend`

Read templates from `templates/frontend/`:

| Template file | Target path |
|---------------|-------------|
| `next.config.ts` | `./next.config.ts` |
| `middleware.ts` | `./middleware.ts` |
| `layout.tsx` | `./app/layout.tsx` |
| `page.tsx` | `./app/page.tsx` |
| `loading.tsx` | `./app/loading.tsx` |
| `error.tsx` | `./app/error.tsx` |
| `actions.ts` | `./app/actions.ts` |

Override tsconfig.json: set `"module"` to `"ESNext"`, `"moduleResolution"` to `"Bundler"`. Remove `"outDir"` and `"rootDir"`. Add `"jsx": "preserve"`.

Override package.json scripts: `"dev": "next dev"`, `"build": "next build"`, `"start": "next start"`.

Add to package.json deps: next, react, react-dom, @types/react, @types/react-dom.

### If `library`

Read `templates/library/tsup.config.ts` → `./tsup.config.ts`.

Add exports field and `tsup` to package.json. Create `src/index.ts`. Override build script: `"build": "tsup"`. Add `"prepublishOnly": "pnpm run build && pnpm run test"`.

### If `monorepo`

Read templates from `templates/monorepo/`:

| Template file | Target path |
|---------------|-------------|
| `turbo.json` | `./turbo.json` |
| `pnpm-workspace.yaml` | `./pnpm-workspace.yaml` |

Create `apps/` and `packages/` directories. Set package.json `"private": true`. Override scripts to use `turbo run`. Add `turbo` to devDependencies.

## Step 4: Scaffold hookify rules

Read `templates/hookify/process-env-guard.local.md` → `./.claude/hookify.process-env-guard.local.md`

## Step 5: Finalize

1. Replace all `{{placeholders}}` in package.json with actual versions from Context7
2. Run `pnpm install`
3. Run `pnpm run typecheck` — verify zero errors
4. Run `pnpm run lint` — verify zero warnings
5. Announce: "Project scaffolded. Run `pnpm run typecheck && pnpm run lint` to verify."
