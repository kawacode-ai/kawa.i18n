# CLAUDE.md — kawa.i18n

Guidance for Claude Code when working in this repository.

**kawa.i18n** is a Kawa Code extension that translates code identifiers (variables, functions, classes) and comments between human languages — so an English codebase can be read and edited in Japanese, Romanian, etc., while the canonical source stays in one language.

It runs as a **standalone binary** spawned by Kawa Code (Muninn) over IPC, plus a web component that mounts inside Muninn's UI. The translation pipeline uses an AST transformer to extract identifiers, a token mapper to maintain stable cross-language IDs, and a backend (local dictionary or remote API) to perform the actual translation.

---

## Workflow — run on every non-trivial turn

Follow the standard Kawa Code workflow defined in the parent [Odin CLAUDE.md](../CLAUDE.md):

1. `check_active_intent` — resume if one exists.
2. `get_relevant_context` with a task description.
3. Now explore code, read files, plan.
4. Create an intent when transitioning to actual code changes.
5. Before commit — `record_decision` for significant decisions.
6. After commit — `complete_intent` with the commit SHA and `status: "committed"`.

Use `repoOrigin: git@github.com:kawacode-ai/kawa.i18n.git` and `repoPath: /Users/markvasile/Code/CodeAwareness/Odin/kawa.i18n`.

---

## Build & Development

```bash
yarn dev               # tsx src/index.ts (run service from source)
yarn dev:watch         # hot reload
yarn build             # tsc + UI build (cd ui && yarn build)
yarn build:macos       # build + pkg → binaries/i18n-service
yarn build:windows     # → binaries/i18n-service.exe
yarn build:linux       # → binaries/i18n-service
./setup-dev-config.sh  # configure Kawa Code to spawn ./dev.sh instead of the binary
./dev.sh               # the dev entry point Kawa Code spawns in dev mode
yarn test              # jest
yarn test:roundtrip    # roundtrip translation correctness
```

`postinstall` runs `cd ui && yarn install` — UI dependencies are installed automatically.

To iterate on the extension while it's running inside Kawa Code:
1. Run `./setup-dev-config.sh` once to point Muninn at `dev.sh`.
2. Edit source.
3. Reload the extension from Kawa Code's UI (or restart Muninn).

---

## Architecture

### Service entry
- `src/index.ts` — boots the IPC server, registers handlers, opens the socket Muninn spawns it on.

### Translation pipeline (`src/core/`)
- `astTransformer.ts` — language-aware AST walking, extracts identifiers and ranges.
- `identifierExtractor.ts` — pulls user-defined names (skipping keywords/built-ins).
- `commentExtractor.ts` — pulls comments separately (translated as natural language).
- `markdownExtractor.ts` — handles `.md` files.
- `tokenMapper.ts` — assigns stable token IDs so translation is reversible (round-trippable).
- `translator.ts` — orchestrates extraction → backend call → re-injection.
- `unifiedTranslator.ts` — entry point that wraps the full pipeline.
- `types.ts` — shared types.

### Translation backends (`src/translation/`)
- `local-backend.ts` — dictionary lookup from `src/dictionary/` (no network).
- `api-backend.ts` — remote translation service.
- `backend.ts` — backend interface + selection logic.

### IPC (`src/ipc/`)
- `server.ts` — socket server (Unix sockets / named pipes).
- `protocol.ts` — message framing and types.
- `handlers.ts` — domain action handlers (subscribed to `i18n`, `auth`, `intent`).
- `muninn-socket.ts` — outbound calls back to Muninn.
- `stream-buffer.ts` — chunked-message reassembly.

### Other
- `src/dictionary/` — bundled translation dictionaries.
- `src/intent/` — intent-aware translation (translate only the lines touched by an active intent).
- `src/auth/` — token handling for the API backend.
- `src/claude/` — Claude API helpers (LLM-assisted translation).
- `ui/` — separate Vite project that builds `i18n-ui.js`, the web component embedded in Muninn.

---

## Extension manifest (`extension.json`)

Kawa Code reads this to wire the extension up:
- `binary.path` → production binary; `binary.devPath` → `./dev.sh`; `binary.devMode: "spawn"` (Muninn spawns the process).
- `domains.subscribe` — IPC domains this extension listens on (`i18n`, `auth`, `intent`).
- `ui.webComponent` — bundled UI module (`./ui/dist/i18n-ui.js`), routes, screens.
- `ui.settings` — settings panel registered in Muninn's settings UI.

When changing handler domains, update `domains.subscribe` — Muninn won't route messages to domains the extension hasn't declared.

---

## Round-trip correctness

The translator must be reversible: translating `en → ja → en` should produce identical source. The `tokenMapper` keeps stable IDs to make this work. **Run `yarn test:roundtrip` and `yarn test:comprehensive` before committing changes to anything in `src/core/`.**

---

## Naming conventions

Outside `kawa.muninn` and `kawa.api`, refer to the desktop app as **"Kawa Code"**. `Muninn` is acceptable in source comments because they're repo-local, but not in user-facing strings or the UI.

---

## Gotchas

- **Binary spawn semantics**: Muninn spawns this service as a child process. It must speak the IPC protocol on stdin/stdout (or the socket Muninn passes). Don't write debug output to stdout — use stderr or a file logger.
- **Postinstall coupling**: `yarn install` triggers a UI install. If `ui/` is missing or its `package.json` is broken, the top-level install fails. Don't delete `ui/` without removing the `postinstall` hook.
- **Dictionary updates** (under `src/dictionary/`) require a rebuild and a binary repackage — they're bundled into the executable via `pkg`, not loaded at runtime.
- **AST coverage**: the AST transformer is language-specific. When adding support for a new source language, add identifier-extraction tests to `src/__tests__/` and a roundtrip example in `examples/`.
