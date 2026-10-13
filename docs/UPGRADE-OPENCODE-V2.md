# Upgrade plan: OpenCode V2 plugin API

Tracking branch: `feat/opencode-v2-plugin` (created from `main` @ `0.1.19`).

## 1. Problem

OpenCode **v2.x** changed the plugin module contract. A plugin's default export must now be an
**object** shaped `{ id: string, effect: fn }` or `{ id: string, setup: fn }`. `oc-sync` currently
exports a **function**, which V2 rejects before the plugin code ever runs:

```
PluginModule.LoadError: Plugin must export a default definition with an id and an effect or setup
function. (cause: SchemaError(Expected object at ["default"]))
```

Symptoms:

- `~/.config/opencode/opencode.jsonc` still declares `"plugin": ["oc-sync"]`.
- OpenCode installs `oc-sync@latest` into its own cache and fails to load it, so **sync silently
  stops working** after the V2 upgrade. It is not a token/config/network problem.

Current state: `main` (and published `0.1.19`) still use the V1 shape:

```ts
// src/plugin/plugin.ts
export const OpencodeSyncPlugin = async (_ctx: unknown): Promise<Record<string, unknown>> => { ... };
// src/index.ts
export default OpencodeSyncPlugin;
```

## 2. V1 → V2 API delta

The V2 API is shipped in `@opencode-ai/plugin` (already a dependency) via subpath exports:

| Entry point | Shape | Cleanup |
|---|---|---|
| `@opencode-ai/plugin/v2/promise` | `{ id: string; setup: (ctx: PluginContext) => Promise<void> \| void }` | none (return value ignored) |
| `@opencode-ai/plugin/v2/effect` | `{ id: string; effect: (ctx: PluginContext) => Effect<void, never, Scope> }` | `Scope` finalizers |

Both entry points export `define(plugin)` (currently an identity helper for typing).

The V2 `PluginContext` exposes `options`, `plugin` (add/remove domain), and hook domains
(`agent`, `aisdk`, `catalog`, `command`, `integration`, `reference`, `skill`). It does **not**
expose the V1 `PluginInput` (`client`, `project`, `directory`, `worktree`, `serverUrl`, `$`).

**Good news:** `oc-sync` ignores its plugin context entirely and uses `node:os`/`homedir()` and its
own GitHub client, so it uses **no V1-only context or hooks**. The port is almost entirely a
wrapper change.

## 3. Migration steps

### 3.1 Wrap the plugin definition (primary change)

`src/plugin/plugin.ts` — convert the exported function into a V2 definition:

```ts
import { define } from '@opencode-ai/plugin/v2/promise';
import type { PluginContext } from '@opencode-ai/plugin/v2/promise';

export const OpencodeSyncPlugin = define({
  id: 'oc-sync',
  setup: async (_context: PluginContext): Promise<void> => {
    log('Plugin starting...');
    try {
      const pathConfig = getPathConfig(homedir());
      await initializeState(pathConfig);
      const isValid = validateAndLogConfig(getPluginState());
      if (isValid) {
        await ensureStorageExists(pathConfig);
        performInitialSync(pathConfig);
        startFileWatcher(pathConfig);
        startIntervalSync(pathConfig);
        log('Plugin ready');
      }
    } catch (error) {
      log(`ERROR: Plugin initialization failed: ${getErrorMessage(error)}`);
      const stack = getErrorStack(error);
      if (stack) console.error(stack);
    }
  },
});
```

`src/index.ts` stays `export default OpencodeSyncPlugin;` — now exporting the object.

Notes:

- `setup` returns `void`, so drop `return { cleanup: stopBackgroundSync }` and the `return {}`.
  This matches current reality: the V1 `{ cleanup }` was never invoked by OpenCode (see the old
  comment referencing a missing shutdown hook).
- `_context` is intentionally unused; prefixed `_` to satisfy `noUnusedParameters`.
- Keep the existing `try/catch` so a bad config/network never crashes plugin load.

### 3.2 Decide lifecycle/cleanup strategy

The V1 code registered a non-functional cleanup. In V2:

- `v2/promise` `setup()` has no cleanup channel. Timers/watchers started here live for the process.
- `v2/effect` can register a `Scope` finalizer to call `stopBackgroundSync()` on unload/reload.

**Recommendation for the first cut:** ship the `v2/promise` port (smallest, lowest risk). Track
graceful shutdown as a follow-up using `v2/effect` + `Scope` finalizers, and make
`startFileWatcher`/`startIntervalSync` idempotent (dispose any existing instances before starting)
so plugin reloads do not stack timers/watchers.

### 3.3 Dependency + packaging

- Keep `@opencode-ai/plugin` but **pin** to a v2-capable range (e.g. `^1.19.0`) instead of
  `"latest"`, so the `./v2/*` exports are guaranteed.
- Confirm the `exports` map still exposes `"."` (it does) — the loader resolves `dist/index.js`.
- Bump `version` to a **breaking** release: `0.2.0` (or `1.0.0`). V1 and V2 module shapes are
  mutually exclusive — one build cannot satisfy both, because V1 expects a function and V2 expects
  an object.

### 3.4 Version/compatibility strategy

- `oc-sync@0.2.x` → requires OpenCode **v2.x**.
- Keep `0.1.x` (and a `v1`/`maintenance` branch) for OpenCode **1.x** users.
- Publish `0.2.0` under a `next` dist-tag first, then promote to `latest` once verified.
- Document the minimum OpenCode version prominently in README/docs.

## 4. Files to change

| File | Change |
|---|---|
| `src/plugin/plugin.ts` | Wrap in `define({ id, setup })`, drop cleanup return |
| `src/index.ts` | No structural change (default export is now the V2 object) |
| `package.json` | Pin `@opencode-ai/plugin` to v2 range; bump version; add `peerDependencies`/engines metadata |
| `src/plugin/background-sync.ts` | Make `startFileWatcher`/`startIntervalSync` idempotent (follow-up for reload safety) |
| `README.md` | Installation/compatibility section for V2 |
| `docs/LLM-INSTALL.md` | Update agent install prompts / required version |
| `docs/CONFIG.md` | Note supported OpenCode version |
| `docs/DEV-SETUP.md` | How to load a local build into OpenCode V2 for testing |
| `docs/ARCHITECTURE.md` | Update `plugin/` description (V2 definition) |
| `docs/TODO.md` | Record the V2 migration + remove stale `@opencode-ai/plugin` "verify" item |

## 5. Testing plan

1. `npm run check` (typecheck + lint + format) against the v2 types.
2. `npm run build`; inspect `dist/index.js` default export is an object with `id`/`setup`.
3. **Load test against OpenCode v2.x** using a local entrypoint (absolute path) so no publish is
   needed:
   - Register the built file (absolute path) in `opencode.json` and confirm the log shows
     `loading plugin` followed by no `PluginModule.LoadError`, and `opencode-sync.log` shows
     `Plugin starting...` / `Plugin ready`.
   - Confirm `GET`-level behavior: initial sync, file-watcher sync, interval sync.
4. Regression: verify config/token/env behavior unchanged (token from config file or
   `GITHUB_TOKEN`), and that missing config still logs "sync is disabled" without failing load.
5. Reload safety: toggle plugin config / restart and confirm no duplicate timers/watchers leak
   (relevant once 3.2 follow-up lands).

## 6. Risks / open questions

- **Cleanup semantics:** exact promise-API shutdown behavior isn't documented in the type package;
  confirm empirically whether `setup` is re-invoked or `PluginDomain.remove` disposes anything.
  If graceful shutdown is required, prefer `v2/effect`.
- **OpenCode version floor:** confirm the first OpenCode release that ships the `./v2/*` exports
  and the `{ id, setup }` schema; pin docs/deps to it.
- **Breaking change for users on OpenCode 1.x:** must be released as a major/minor bump with clear
  migration notes; keep 0.1.x installable.
- **Env/runtime:** V2 plugins run inside the OpenCode runtime (Bun/Effect); confirm `setInterval`,
  `fs.watch`, and `pako` behave the same. The plugin already runs successfully in V1, so risk is low.
- **No event hooks used:** V2 `PluginContext` has no generic `event` hook; fine for `oc-sync` today,
  but any future session/event integration must use the V2 domains.

## 7. Suggested commits on this branch

1. `docs: add OpenCode V2 upgrade plan`
2. `feat(plugin)!: migrate to @opencode-ai/plugin v2 promise API` (wrap with `define`)
3. `chore(deps): pin @opencode-ai/plugin to v2 range; bump to 0.2.0`
4. `test: verify V2 load + sync paths`
5. `docs: update install/compat docs for OpenCode V2`
