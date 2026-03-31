# Get Claude Code working from Source

Here's how i got it working. 
Download it, put it in a folder and paste this as a prompt in Claude code to get it working from source.
markdown
Get this codebase up and running. Move files if required. 
clone https://github.com/anthropics/claude-code into a temp folder if you need help. `anthropics/claude-code` is the outside scaffolding as far as i know. Fix any version errors if you encounter them.

Stop only when you have got it working.
Below is the Run Summary.

Claude Code — Building from Source
The `src/` directory contains the raw TypeScript/React source code for Claude Code, but has zero build infrastructure — no `package.json`, no `tsconfig.json`, no build scripts. The published npm package (`@anthropic-ai/claude-code`) ships a single 13MB bundled `cli.js` built with Bun, so the build system isn't public. This documents how it was reconstructed.
Steps
1. Cloned the public repo for reference
- `https://github.com/anthropics/claude-code` → `/tmp/claude-code-ref`
- It only contains plugins, scripts, and README — no source code or build config
2. Analyzed the source code
- Entry point: `src/entrypoints/cli.tsx`
- Uses Bun's bundler (`bun:bundle` for feature flags, `MACRO.*` for build-time constants)
- ~1,884 `.ts`/`.tsx` files, vendored Ink (terminal React framework) in `src/ink/`
- 85 external npm dependencies identified by scanning all imports
3. Created `package.json`
- All 85 npm dependencies with versions
- Local file stubs for internal packages (`@ant/*`, `color-diff-napi`)
- Dev dependencies for types (`@types/react`, `bun-types`, `typescript`, etc.)
4. Created `tsconfig.json`
- Bun + React JSX, ES2022 modules, bundler resolution
5. Created `build.ts` (Bun bundler script)
- Bundles `src/entrypoints/cli.tsx` → `dist/cli.js`
- Defines `MACRO.VERSION`, `MACRO.BUILD_TIME`, `MACRO.PACKAGE_URL`, etc.
- Plugin to shim `bun:bundle`'s `feature()` → always returns `false`
- Defines `process.env.NODE_ENV` = `"production"` for dead code elimination
- Marks native modules as external (`sharp`, `audio-capture-napi`, etc.)
6. Created stub packages for internal Anthropic modules
- `stubs/@ant/claude-for-chrome-mcp` — browser integration
- `stubs/@ant/computer-use-mcp` — computer use
- `stubs/@ant/computer-use-swift` — macOS Swift bridge
- `stubs/@ant/computer-use-input` — input handling
- `stubs/color-diff-napi` — native syntax highlighting
7. Created ~20 stub source files for missing internal modules
Feature-gated code that references files not in the open-source extract:
**Types and tools:**
- `src/types/connectorText.ts`
- `src/tools/TungstenTool/`
- `src/tools/REPLTool/`
- `src/tools/SuggestBackgroundPRTool/`
- `src/tools/VerifyPlanExecutionTool/`
- `src/tools/WorkflowTool/constants.ts`
**Assistant / agents:**
- `src/assistant/` (index, gate, sessionDiscovery, AssistantSessionChooser)
- `src/commands/assistant/`
- `src/commands/agents-platform/`
- `src/components/agents/SnapshotUpdateDialog.ts`
**Services and utils:**
- `src/services/compact/snipCompact.ts`
- `src/services/compact/cachedMicrocompact.ts`
- `src/services/contextCollapse/index.ts`
- `src/utils/protectedNamespace.ts`
- `src/utils/filePersistence/types.ts`
- `src/entrypoints/sdk/` generated types
- `src/ink/devtools.ts`
- `src/ink/global.d.ts`
**Content stubs:**
- 26 markdown/text files for bundled skills (`claude-api/`, `verify/`, classifier prompts)
8. Fixed two runtime issues
- **Commander.js**: rejected `-d2e` short flag (not valid in v14) → changed to `--d2e`
- **Auto-updater**: version `0.0.0-dev` was below the enforced minimum → set to `2.1.88`
9. Fixed React version mismatch
- `react@19.2.4` (stable) doesn't have `useEffectEvent`
- `react-reconciler` and the vendored Ink code require it
- Switched to matching canary builds: `react@19.3.0-canary-ed69815c-20260323` + `react-reconciler@0.34.0-canary-ed69815c-20260323`
Usage
bash
# Build
bun run build.ts # builds dist/cli.js (27MB)
# Run
node dist/cli.js # interactive REPL
node dist/cli.js -p # non-interactive pipe mode
node dist/cli.js --help # shows all options
node dist/cli.js --version