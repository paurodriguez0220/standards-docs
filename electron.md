# Electron Desktop Apps

## Purpose

Standards for building Electron desktop apps. Distilled from building [claude_orchestrator-desktop](https://github.com/paurodriguez0220/claude_orchestrator-desktop) — a Windows desktop app wrapping CLI tools, PTYs, and git.

Platform-scoped: applies only when a project is an Electron app.

---

## Stack

| Concern | Choice |
| --- | --- |
| Bundler / dev server | `electron-vite` (one config, three targets: main, preload, renderer) |
| Packaging | `electron-builder` (NSIS target on Windows) |
| Renderer | React + TypeScript strict mode (see [Web Components](web-components.md)) |
| Terminal UI | `@xterm/xterm` + `@xterm/addon-fit` |
| PTY | `node-pty` |
| Tests | Vitest — two configs, one per process (see below) |

---

## Process Model & IPC Boundary

The renderer never touches the filesystem, spawns processes, or imports Node APIs. All system access lives in the main process. This is the single most important architectural rule.

- **Main process** — a service layer (`src/main/services/`) for domain logic, plus thin per-domain IPC handler modules (`src/main/ipc/`) registered at startup. Handlers validate and delegate; services do the work.
- **Shared contract** (`src/shared/`) — IPC channel names as constants and request/response types in one place. Both sides import from here; no stringly-typed channels.
- **Preload** — a typed facade over `ipcRenderer`, exposed via `contextBridge.exposeInMainWorld`. Export the interface so renderer code and tests type against it:

```typescript
export interface MyAppApi {
  createTask(request: TaskCreateRequest): Promise<TaskRecord>;
  onPtyOutput(listener: (event: PtyOutputEvent) => void): () => void; // returns unsubscribe
}
contextBridge.exposeInMainWorld('myApp', api);
```

- Use `ipcRenderer.invoke`/`ipcMain.handle` for request/response; `send`/`on` only for fire-and-forget streams (PTY input, resize events).
- Event subscriptions exposed from preload must return an unsubscribe function — React effects need the cleanup.

---

## Native Modules (node-pty and friends)

Native modules need three things or they break silently at runtime:

1. **`allowScripts`** in `package.json` — npm blocks postinstall build scripts by default; allowlist the exact native packages (`node-pty`, `electron`, `esbuild`).
2. **`asarUnpack`** in the `build` block — native `.node` binaries cannot load from inside the asar archive:

```json
"asarUnpack": ["**/node_modules/node-pty/**/*"]
```

3. **`npmRebuild: false`** when prebuilt binaries already match the Electron ABI — avoids a broken rebuild step on corporate machines without build tools.

---

## Spawning External Processes

- Never build a shell command by string-interpolating user input (URLs, branch names, titles). Always `execFile`/`spawn` with argument arrays.
- Always set `cwd` explicitly when the spawned tool is directory-sensitive (git, claude). Don't inherit the app's working directory — it differs between dev mode and the installed app.
- Under strict `tsc`, `execFile` stdout may type as `string | Buffer` depending on options — coerce explicitly (`String(stdout)`) rather than casting.
- For one-shot AI synthesis, `claude -p "<prompt>"` (headless print mode) works well as a child process — treat it like any other CLI: arg arrays, explicit `cwd`, timeout.

---

## Renderer Gotchas

- **Clipboard images:** the DOM `paste` event is unreliable inside `xterm.js`/Electron — use the Async Clipboard API (`navigator.clipboard.read()`) to pull image blobs instead.
- **External links:** deny window opens and route to the OS browser:

```typescript
mainWindow.webContents.setWindowOpenHandler((details) => {
  shell.openExternal(details.url);
  return { action: 'deny' };
});
```

- **Dev vs packaged loading:** dev mode loads `process.env['ELECTRON_RENDERER_URL']` (set by electron-vite); packaged loads `loadFile(...)`. Gate on `app.isPackaged`, not `NODE_ENV`.

---

## Testing

Two Vitest configs, because the two processes have different runtimes:

| Config | Environment | Covers |
| --- | --- | --- |
| `vitest.main.config.ts` | `node` | services, IPC handlers, preload (with `electron` mocked) |
| `vitest.renderer.config.ts` | `jsdom` | React components via Testing Library |

- Mock the `electron` module in main-process tests — never boot Electron in unit tests.
- In renderer tests, stub the preload API (`window.myApp`) — the typed facade interface makes this trivial.
- Every component folder co-locates `component.tsx`, `component.test.tsx`, `component.stories.tsx` (see [Web Components](web-components.md)).

---

## Packaging & Distribution

- `electron-builder --win` with the NSIS target; config lives in `package.json`'s `build` block; output to `dist/`.
- `files: ["out/**/*"]` — ship only the bundled output, not `src/`.
- **Code-signing is intentionally skipped for single-user personal tools.** The SmartScreen "unknown publisher" warning on first run is expected and accepted ("More info" → "Run anyway"). Signing certificates are not worth it for an audience of one.
- `nsis.oneClick: false` + `allowToChangeInstallationDirectory: true` for a normal installer experience.
- After installing, launch from the Start Menu entry — don't keep dev-mode launcher scripts around once a packaged build exists.

---

## Related

- [Web Components & Storybook](web-components.md)
- [Code Style & Conventions](code-style.md)
- [Security Practices](security.md)
- [Testing](testing.md)

---
*Maintained by paurodriguez0220 · Last updated: 2026-07-09*
*Standards: https://github.com/paurodriguez0220/standards-docs*
