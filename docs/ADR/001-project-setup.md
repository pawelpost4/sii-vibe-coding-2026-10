# ADR-001: Project Initialisation and Tooling

**Date:** 2026-10-18
**Status:** Accepted
**Relates to:** `docs/ADR/000-main-architecture.md`

---

## 1. Scope

How the implementing agent creates the Next.js project from the empty `app/` scaffold: commands and their order, required flags, strict TypeScript settings, dependencies, UI component setup (shadcn/ui + AI Elements), test tooling, environment files, npm scripts and the verification that the skeleton works. Not covered: feature code (ADR-002, ADR-003).

---

## 2. Technology Documentation References

| Library | Context7 ID or official docs | Used for |
|---|---|---|
| Next.js / create-next-app | `/vercel/next.js` | CLI flags, empty-folder rule, App Router layout |
| AI SDK | `/vercel/ai` | Install (`ai`, `@ai-sdk/react`, `zod`), Node ≥ 22 requirement (AI SDK 7 migration guide) |
| OpenRouter provider | `/openrouterteam/ai-sdk-provider` | Package line compatible with `ai@^7` |
| shadcn/ui | `/shadcn-ui/ui` | `shadcn init` |
| AI Elements | `/vercel/ai-elements` | `npx ai-elements@latest add …` |
| sharp | `/lovell/sharp` | Native image library install |
| Vitest | `/vitest-dev/vitest` | Test runner config |
| React Testing Library / user-event | `/testing-library/react-testing-library`, `/testing-library/user-event` | Component tests |
| Playwright | `/microsoft/playwright` | `@playwright/test` config, `webServer` |

---

## 3. Component Design — Initialisation Procedure

Verified facts that shape the procedure (Context7, 2026-10-18):

- `create-next-app` refuses to scaffold into a folder containing files other than a short allow-list (`.git`, `.gitignore`, `.claude`, `docs`, `LICENSE`, …). `app/README.md` is **not** on the list, so the command fails while it exists.
- `create-next-app` defaults: TypeScript, Tailwind, App Router-ready, Turbopack, `@/*` alias, and it generates an `AGENTS.md` (`--agents-md`, default) to guide coding agents.
- AI SDK 7 and the current OpenRouter provider line require **Node ≥ 22** and are ESM-only. The machine has Node 24.
- AI Elements needs React 19, Tailwind 4 and shadcn/ui; its CLI initialises shadcn if missing.

Steps (run from the repository root unless stated otherwise; each step is a separate, reviewable change):

1. **Check prerequisites:** `node --version` ≥ 22, `npm --version`. Stop and report if lower.
2. **Free the folder:** move the course checklist out of the target: `git mv app/README.md docs/app-setup-checklist.md`. `app/` must then be empty.
3. **Scaffold:** `npx create-next-app@latest app --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --disable-git --agents-md --no-agent-feedback --yes`.
   - `--disable-git`: the repo already exists; no nested repository.
   - `--src-dir`: keeps `src/app`, `src/features`, `src/server`, `src/lib` apart from config files.
   - Keep Turbopack (default) for `dev`/`build`.
   - Do not pass `--empty` (we want the layout and Tailwind wiring) and do not enable `--react-compiler` (not needed; fewer moving parts).
4. **Pin Node:** add `"engines": {"node": ">=22"}` to `app/package.json`.
5. **Strict TypeScript:** in `app/tsconfig.json` keep `strict: true` and add `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `exactOptionalPropertyTypes`, `forceConsistentCasingInFileNames`. Add an npm script `typecheck` = `tsc --noEmit`. Generated shadcn/AI Elements files must also compile under these flags; fix generated code minimally if they don't (record any deviation in `app/AGENTS.md`).
6. **Runtime dependencies** (in `app/`): `npm install ai @ai-sdk/react @openrouter/ai-sdk-provider zod sharp server-only`. Use the provider's current line that declares `ai@^7` support; ensure `zod` ≥ 4.1.8.
7. **UI components:** `npx shadcn@latest init` (Tailwind v4, CSS variables), then `npx ai-elements@latest add conversation message prompt-input` (only these; add more only when needed). Generated files land in `src/components/ui` and `src/components/ai-elements` and are committed.
8. **Dev dependencies:** `npm install -D vitest @vitejs/plugin-react vite-tsconfig-paths jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom @playwright/test prettier tsx`; then `npx playwright install chromium`.
9. **Test config:** `vitest.config.ts` with two projects: `node` (files under `src/server`, `src/lib`, `src/app/api`) and `jsdom` (`*.test.tsx`), setup file registering jest-dom matchers and a network guard that fails any request to `openrouter.ai`. `playwright.config.ts` with `testDir: e2e`, Chromium, `webServer` running `npm run build && npm start` on port 3000, `reuseExistingServer` locally.
10. **Environment:** `cp .env.example app/.env.local`, fill the key; confirm `app/.gitignore` ignores `.env*` except `.env.example`. Document variables (ADR-000 §8) in `app/AGENTS.md`.
11. **Scripts** in `app/package.json`: `dev`, `build`, `start`, `lint`, `typecheck`, `test` (`vitest run`), `test:watch`, `test:e2e` (`playwright test`), `capture:llm` (`tsx scripts/capture-llm.ts`, live, needs key), `check:env` (`tsx scripts/check-env.ts`), `check:bundle` (build, then fail if `OPENROUTER_API_KEY` value or `LLM_MODEL` value appears in `.next/static`), `format` (`prettier --write .`).
12. **`app/AGENTS.md`:** keep the generated Next.js guidance; append: stack summary, commands above, folder rules (`src/server` server-only, Polish strings only in `src/lib/i18n/pl.ts`), link to `docs/ADR/`, verified `LLM_MODEL` ID + date (ADR-000 D3), and that `assets/example-images/` is the only source of test photos.
13. **Skeleton check (before features):** `npm run typecheck && npm run lint && npm test && npm run build` pass; `npm run dev` serves `http://localhost:3000`; manual QA screenshot of the default page; then commit `Frontend: scaffold Next.js app` (one logical commit), followed by separate commits for dependencies/UI setup.

---

## 4. Data Structures

Configuration artefacts produced:

| File | Contents |
|---|---|
| `app/package.json` | engines, scripts (§3.11), dependencies |
| `app/tsconfig.json` | strict flags (§3.5), `@/*` → `src/*` |
| `app/components.json` | shadcn config (Tailwind v4, aliases `@/components`, `@/lib/utils`) |
| `app/vitest.config.ts` | node + jsdom projects, setup file, network guard |
| `app/playwright.config.ts` | E2E config with `webServer` |
| `app/.env.local` | secrets (gitignored) |
| `app/AGENTS.md` | agent guidance (§3.12) |
| `docs/app-setup-checklist.md` | moved course checklist |

---

## 5. Interface Contracts

| Command (in `app/`) | Expected result |
|---|---|
| `npm run dev` | Next dev server on :3000 with Turbopack |
| `npm run build` / `npm start` | Production build / server on :3000 |
| `npm run typecheck` | `tsc --noEmit`, 0 errors |
| `npm run lint` | ESLint, 0 errors, 0 warnings |
| `npm test` | Vitest unit + component + integration, no network |
| `npm run test:e2e` | Playwright against built app and real OpenRouter |
| `npm run capture:llm` | Saves real model outputs to `tests/fixtures/captures/` |
| `npm run check:env` | Validates env vars and policy files; non-zero exit with a clear message on failure |

---

## 6. Technical Decisions

### S1 — Scaffold with create-next-app into `app/`
**Status:** Accepted
**Date:** 2026-10-18
**Context:** The course starts from an empty `app/`; the agent must create the template itself.
**Decision:** Use the official CLI with explicit flags (§3.3) after moving `app/README.md` out.
**Rejected alternatives:**
- Starting from a chatbot template: brings auth, DB and history, which the PRD excludes; more code to delete than to write.
- Hand-written config: conflicts with framework defaults (warned in `app/README.md`).
**Consequences:**
- (+) Reproducible, current defaults, generated `AGENTS.md` for agents.
- (-) The course checklist moves to `docs/app-setup-checklist.md`.
**Review trigger:** If create-next-app changes its flags or allow-list.

### S2 — Strict TypeScript beyond `strict`
**Status:** Accepted
**Date:** 2026-10-18
**Context:** User asked for strict mode; LLM outputs and request bodies are untrusted.
**Decision:** Extra flags in §3.5; all external data parsed with zod before use; no `any` (ESLint `@typescript-eslint/no-explicit-any` as error).
**Rejected alternatives:**
- Default `strict` only: misses unchecked index access on arrays from model outputs.
**Consequences:**
- (+) Fewer runtime surprises.
- (-) Generated component code may need small fixes.
**Review trigger:** If generated libraries cannot compile under the flags without large patches.

### S3 — npm, Node ≥ 22
**Status:** Accepted
**Date:** 2026-10-18
**Context:** User choice; pnpm is not installed; AI SDK 7 requires Node 22+.
**Decision:** npm with committed `package-lock.json`; `engines.node >= 22`.
**Rejected alternatives:**
- pnpm: extra setup for participants.
**Consequences:**
- (+) Works on every participant laptop with Node 22+.
- (-) Slower installs.
**Review trigger:** If install times become a problem in CI.

---

## 7. Diagrams

### Initialisation sequence

```mermaid
sequenceDiagram
    actor A as Implementing agent
    participant R as Repo root
    participant N as create-next-app
    participant P as app/
    A->>R: node --version (≥ 22?)
    A->>R: git mv app/README.md docs/app-setup-checklist.md
    A->>N: npx create-next-app@latest app --ts --tailwind --eslint --app --src-dir ... --disable-git --yes
    N->>P: scaffold + npm install + AGENTS.md
    A->>P: strict tsconfig, engines
    A->>P: npm install ai @ai-sdk/react @openrouter/ai-sdk-provider zod sharp server-only
    A->>P: npx shadcn init; npx ai-elements add conversation message prompt-input
    A->>P: dev deps, vitest + playwright config, scripts
    A->>P: cp ../.env.example .env.local
    A->>P: typecheck, lint, test, build, dev → screenshot
    A->>R: commit per logical step
```

### Resulting dependency graph

```mermaid
flowchart LR
    APP[app/ Next.js 16] --> AI[ai v7]
    APP --> REACT[@ai-sdk/react]
    APP --> ORP[@openrouter/ai-sdk-provider]
    APP --> ZOD[zod ≥4.1.8]
    APP --> SHARP[sharp]
    APP --> SO[server-only]
    APP --> SHAD[shadcn/ui + AI Elements]
    APP -. dev .-> VT[vitest + RTL]
    APP -. dev .-> PW[@playwright/test]
    ORP --> AI
    REACT --> AI
```

---

## 8. Testing Strategy

### Test scenarios for this area

| Scenario | Type | Input | Expected output | Edge cases |
|---|---|---|---|---|
| Skeleton builds | Smoke | Fresh scaffold | typecheck, lint, test (0 tests allowed only at this step), build pass | — |
| Strict flags active | Unit | A test file accessing `array[0]` without a check | `tsc` error (verified once, then removed) | — |
| Network guard | Unit | Test calling `fetch('https://openrouter.ai/...')` | Test fails with a clear message | — |
| Env check | Unit | Missing key / missing policy file | `check:env` exits non-zero naming the problem | Empty policy file |
| Server-only boundary | Build | Client component importing `src/server/*` | `npm run build` fails | — |
| Dev server | Manual | `npm run dev` | Page renders at :3000; screenshot saved | — |

### Technical acceptance criteria

- **TAC-001-01** `app/` contains a Next.js App Router project with `src/` layout, `package-lock.json`, `engines.node >= 22`, and no nested `.git`.
- **TAC-001-02** `tsconfig.json` has `strict`, `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch`, `exactOptionalPropertyTypes` set to true.
- **TAC-001-03** `npm run typecheck && npm run lint && npm test && npm run build` succeed in `app/` with 0 errors and 0 warnings.
- **TAC-001-04** `src/components/ai-elements/` contains `conversation`, `message` and `prompt-input` components; they compile under the strict flags.
- **TAC-001-05** `app/.env.local` is ignored by Git (`git check-ignore app/.env.local` succeeds).
