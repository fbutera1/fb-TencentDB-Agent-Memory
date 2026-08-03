# AGENTS.md

Guidance for AI coding agents working in this repository (TencentDB Agent Memory).

## What this project is

An open-source system that lets AI agents **remember** across sessions — chat memory,
skills, wiki, and code-graph assets accumulate and flow across agent teams. A single
repo containing four Node/TypeScript services, two SDKs, and deploy scripts.

See root `README.md` for product overview; `INSTALL.md` for deployment.

## Repository layout

```
MemoryCore/        Memory & metadata kernel — HTTP Gateway on :8420 (Hono, Node 22, TS)
MemoryPanel/       Team memory control panel — stateless backend (:8125) + React/Vite web/
MemoryKnowledge/   Knowledge service — Wiki + Code-Graph engine on :8421 (Hono, Drizzle/SQLite)
MemoryProxy/       Transparent LLM request proxy on :8096 (forwards OpenAI/Anthropic verbatim)
sdk/memory-core/   Official SDKs — typescript/ and python/ (v2 + v3 clients)
deploy/            Image build & local deploy scripts; global-images/start-all.sh runs all three
.github/workflows/ CI (pr-ci.yml)
```

## Tech stack & versions

- **Node.js ≥ 22.16.0** for all four TS services.
- **Python ≥ 3.9** for the Python SDK and v2→v3 migration scripts.
- Package managers vary per module: **npm** (MemoryCore, sdk/typescript, MemoryPanel/web),
  **pnpm** (MemoryPanel backend, MemoryKnowledge, MemoryProxy). Respect the existing
  lockfile in each dir — do not switch managers.
- Frontend: React 18 + Vite + Tailwind + Zustand, Tea (Tencent Design) component bridge.
- Tests: Vitest everywhere TS; pytest for the Python SDK.

## Ports (default)

| Service        | Port  |
|----------------|-------|
| memory-core    | 8420  |
| memory-hub (panel+knowledge combined image) | 8125 / 8424 |
| knowledge      | 8421  |
| proxy          | 8096  |

## Common commands (run inside the module directory)

| Module            | Dev                                   | Build                | Test / Lint                                  |
|-------------------|---------------------------------------|----------------------|----------------------------------------------|
| MemoryCore        | (see README)                          | `npm run build`      | `npm test` (vitest); `npm run test:oss`     |
| MemoryPanel (be)  | `npm run dev`                         | `npm run build`      | `npm test`; `npm run typecheck`             |
| MemoryPanel/web   | `npm run dev` (vite)                  | `npm run build`      | `npm run lint`; `npm run format:check`      |
| MemoryKnowledge   | `npm run dev`                         | `npm run build`      | `npm test`; `npm run db:generate`/`db:migrate` |
| MemoryProxy       | `npm run dev`                         | (runtime tsx)        | `npm test`                                   |
| sdk/typescript    | —                                     | `npm run build`      | `npm test`                                   |
| sdk/python        | —                                     | `python -m build`    | `pytest` (install `[dev]` extras)            |

### Run the full stack locally

```bash
cd deploy/global-images
cp .env.example .env  # then edit: two LLM param sets (memory group + proxy group)
./start-all.sh        # launches memory-core + memory-hub + proxy
./stop-all.sh         # stop; ./verify.sh to health-check
```

The simplest inner loop: run the stack in Docker, then patch the target module locally
and `npm run dev` inside it.

## Conventions

- **Commits**: Conventional Commits with a module scope and a **DCO sign-off** (required,
  CI-enforced). `git commit -s -m "feat(memory-core): ..."` — type(`scope`): subject.
  Scopes: `memory-core` / `panel` / `knowledge` / `proxy` / `sdk-ts` / `sdk-py` /
  `deploy` / `docs`.
- **Branches**: PR against `develop_server_team` or `master` per maintainer guidance; CI
  runs on PRs to `main`.
- **Code style**: TypeScript — match existing style, comment the *why*. Python — PEP 8
  with type hints. English naming.
- **Tests**: add tests with new features; fix bugs with a regression test first.

## Guarded / red-line areas (do not touch without checking)

- `MemoryCore/src/core/skill/**` — Skill extraction queue. CI runs
  `scripts/ci/check-skill-queue-isolation.sh` forbidding new direct `fs` imports here
  and changes to memory red-line files (state / pipeline-worker / integrations / redis).
  See `MemoryCore/docs/design/2026-06-16-skill-extract-queue.md`.
- `MemoryCore/openclaw.plugin.json` + `package.json#openclaw` — plugin manifest; CI
  validates required fields (`id`, `extensions`, `compat.pluginApi`, `build.openclawVersion`).
- MemoryCore published tarball has a **2 MB** size guard; avoid committing large files.

## Things to keep in mind

- MemoryProxy persists **no memory** — all memory/skill/knowledge reads & writes go
  through the MemoryCore Gateway (:8420). Proxy's own state (session init, injection
  cache, skill pins) uses ProxyStorage with 5 backend options (Redis/COS/SQLite/FS/Memory).
- MemoryCore stores knowledge **metadata**, not content — Wiki parsing & CodeGraph live in
  `MemoryKnowledge/`.
- MemoryPanel is **stateless** — no server-side login session, no local user DB. Auth is
  delegated; it validates credentials and forwards authorized requests.
- Memory model layers: L0 Conversation → L1 Atom → L2 Scenario → L3 Persona.
- Several docs are bilingual (`README_CN.md`, `INSTALL_CN.md`, `CONTRIBUTING_CN.md`);
  keep both in sync when editing docs.

## Security issues

Do **not** open a public issue — email agentmemory@tencent.com. See `CONTRIBUTING.md`.

## License

MIT. Contributions are accepted under the same license.
