# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

MxLint Extension is a Mendix Studio Pro extension that surfaces lint results from [mxlint-cli](https://github.com/mxlint/mxlint-cli) directly inside the IDE. It consists of:
- A **C# .NET 8 extension** loaded by Mendix Studio Pro via MEF (Managed Extensibility Framework)
- A **React/TypeScript frontend** (Vite) served from `wwwroot/` inside the extension DLL's output directory

## Commands

### Full build (frontend + C# extension)
```bash
make           # clean → build frontend → dotnet build → copy to resources/App/extensions/
```

### Frontend only
```bash
cd frontend
npm install
npm run dev        # dev server for standalone development (uses /lint-results.json)
npm run build      # production build → outputs to frontend/dist/
npm run lint       # ESLint with auto-fix
npx tsc --noEmit   # type-check only
```

### C# tests
```bash
dotnet test tests/MxLintExtension.Tests/ --logger "console;verbosity=detailed"
```

### E2E tests (Windows only — requires Mendix Studio Pro installed)
```bash
pip install -r tests/e2e/requirements.txt
robot --loglevel DEBUG --console verbose --outputdir results tests/e2e/
```

### Windows development build (PowerShell)
```powershell
# Release build + deploy to test project
msbuild.exe .\MxLintExtension.sln /t:Rebuild /p:Configuration=Release /p:Platform="Any CPU"
Remove-Item -Recurse .\resources\App\extensions\MxLintExtension\* -Force
Copy-Item -Recurse .\bin\Release\net8.0\* .\resources\App\extensions\MxLintExtension\ -Force
```

### Frontend hot reload during development
```powershell
# Serve the built wwwroot locally (useful to preview without Studio Pro)
cd wwwroot
python -m http.server 8000
```

## Architecture

### C# Layer

The extension registers three MEF exports against the Mendix Studio Pro ExtensionsAPI:

| Class | Export | Responsibility |
|-------|--------|----------------|
| `MxLintMenuExtension` | `MenuExtension` | Adds "Open MxLint" and "About" menu items |
| `MxLintPaneExtension` | `DockablePaneExtension` | Opens the bottom-docked WebView pane, sets the `baseUri` using `WebServerBaseUrl` + `wwwroot/` |
| `MxLintWebServerExtension` | `WebServerExtension` | Serves static files and REST-like API routes |

**`Core/MxLint.cs`** is the workhorse: it manages the mxlint-cli binary (downloads on demand to `.mendix-cache/`), runs export + lint subprocesses, and reads/writes `mxlint.yaml` (the config file in the Mendix project root). YAML is parsed with YamlDotNet using camelCase naming conventions.

**Two messaging paths** exist for the same actions because the extension must work on both Windows (WebView2) and macOS (where WebView messaging has different capabilities):
- **WebView messages** (`MxLintPaneExtensionWebViewModel`): `postMessage` / `MessageReceived` events — primary path on Windows
- **HTTP API messages** (`MxLintWebServerExtension`): REST calls from the frontend to `api/message`, `api/runlint`, etc. — fallback path (used on macOS and for operations the WebView path doesn't cover)

### API Routes (served by `MxLintWebServerExtension`)

Each route is registered under both `<route>` and `wwwroot/<route>` for cross-platform URI compatibility:

| Route | Method | Purpose |
|-------|--------|---------|
| `api` | GET | Returns `lint-results.json` content |
| `api/theme` | GET | Returns Studio Pro theme (`light`/`dark`) |
| `api/noqa` | POST | Add or remove noqa skip rules in `mxlint.yaml` |
| `api/config` | GET/POST | Read or update `mxlint.yaml` raw content |
| `api/bookmarks` | GET/POST | Persist bookmark IDs in `mxlint.yaml` under `ui.bookmarks` |
| `api/runlint` | POST | Trigger a full lint run synchronously (mutex-locked) |
| `api/message` | POST | Dispatch frontend messages (`refreshData`, `runLintNow`, `openDocument`, `setAutoRefresh`) |
| `api/diag` | GET | Log diagnostic events from the frontend |

### Frontend

React 19 + TypeScript + Vite. All state lives in `App.tsx` — no state management library. Key patterns:

- **Data source**: In WebView mode fetches from `./api`; in standalone dev mode fetches from `/lint-results.json`
- **Virtual scrolling**: `useVirtualList` hook renders only visible rows (36px row height, 5 row overscan)
- **Auto-refresh**: Polls every 1 second in WebView mode; detects changes via DJB2 hash of the JSON response
- **Noqa suppression**: Frontend sends `POST api/noqa` with `{ action: "add"|"remove", entries: [{document, rules, reason}] }`
- **Messaging to C#**: Uses `window.chrome?.webview?.postMessage` when in WebView; falls back to `POST api/message`

### Build Pipeline

`make rebuild`:
1. `dotnet clean`
2. Build frontend (`npm install` + `npm run build` → `frontend/dist/`)
3. Copy `frontend/dist/*` → `wwwroot/` (replaces index.html, CSS, JS, assets)
4. `dotnet build` (embeds `wwwroot/` and `manifest.json` as `CopyToOutputDirectory: Always`)
5. Copy `bin/Debug/net8.0/*` → `resources/App/extensions/MxLintExtension/`

`resources/App/` is a bundled Mendix test project used for E2E tests. The extension DLL must be deployed there before running Robot Framework tests.

### CLI Version Management

The default CLI version is hardcoded in `Core/MxLint.cs` as `DefaultCliVersion`. It can be overridden per-project via `mxlint.yaml` under `cli.version`. The binary is cached at `.mendix-cache/<asset-name>` in the Mendix project directory and downloaded from GitHub Releases on first use.
