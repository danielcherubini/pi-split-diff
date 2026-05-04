# pi-diff

[![npm version](https://img.shields.io/npm/v/@heyhuynhgiabuu/pi-diff)](https://www.npmjs.com/package/@heyhuynhgiabuu/pi-diff)

A [pi](https://pi.dev) extension that replaces the default `write` and `edit` tool output with **Shiki-powered, syntax-highlighted diffs** — side-by-side split view, unified stacked view, and word-level change emphasis, all rendered directly in your terminal.

> **Status:** Early release.

### Unified view — stacked single-column diff

<img width="700" alt="pi-diff unified view" src="https://github.com/buddingnewinsights/pi-diff/raw/main/media/unified.png" />

### Split view — side-by-side comparison

<img width="700" alt="pi-diff split view" src="https://github.com/buddingnewinsights/pi-diff/raw/main/media/split.png" />

## Features

- **Syntax-highlighted diffs** — full Shiki grammar highlighting (190+ languages) composited with diff background colors
- **Split view** — side-by-side comparison for `edit` tool, auto-falls back to unified on narrow terminals
- **Unified view** — stacked single-column layout for `write` tool overwrites
- **Word-level emphasis** — changed characters get brighter backgrounds so you see exactly what changed
- **Adaptive layout** — auto-detects terminal width; wraps intelligently on wide terminals, truncates on narrow ones
- **LRU cache** — singleton Shiki highlighter with 192-entry cache for fast re-renders
- **Large diff fallback** — gracefully degrades (skips highlighting, still shows diff structure) for files > 80k chars
- **Auto-derive colors** — automatically derives diff colors from your pi theme, no configuration needed

## Install

```bash
pi install npm:@heyhuynhgiabuu/pi-diff
```

Latest release: https://github.com/buddingnewinsights/pi-diff/releases/latest

Or load directly for development:

```bash
pi -e ./src/index.ts
```

## How It Works

pi-diff wraps the built-in `write` and `edit` tools from the pi SDK, including single-edit and multi-edit `edit` calls. When the agent writes or edits a file:

1. **Before the write** — reads the existing file content
2. **Delegates** to the original SDK tool (file is actually written)
3. **After the write** — computes a structured diff between old and new content
4. **Renders** the diff with syntax highlighting and word-level emphasis

The rendering pipeline:

```
Old content ──┐
              ├── diff (structuredPatch) ── parse ── highlight (Shiki → ANSI)
New content ──┘                                          │
                                                         ├── inject diff bg
                                                         ├── inject word-level bg
                                                         └── wrap/fit to terminal
```

### Views

| View | Used by | Description |
|------|---------|-------------|
| **Split** | `edit` tool | Side-by-side with old on left, new on right. Diagonal stripes fill empty slots. Auto-falls back to unified when terminal < 150 cols or > 20% of lines would wrap. |
| **Unified** | `write` tool | Single column with `+`/`-` gutter. Compact, works at any terminal width. |

Both views show:
- Colored border bars (`▌`) for changed lines
- Line numbers in the gutter
- Hunk separators (`··· N unmodified lines ···`)
- Word-level emphasis on paired add/del lines

## Configuration

### Auto-Derive (Default Behavior)

pi-diff **automatically derives** diff colors from your pi theme's diff foreground colors and tool-state backgrounds. Added/context surfaces use `toolSuccessBg`; removed surfaces use `toolErrorBg`. This ensures diffs look good with any pi theme and terminal background — no configuration needed.

The auto-derive uses different intensity levels:
- **Line backgrounds**: 8–10% of the theme's diff fg color mixed into the matching tool-state background (subtle tint)
- **Word highlights**: 20–22% (more visible for changed characters)
- **Gutters**: 5–6% (subtler than line backgrounds)

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DIFF_THEME` | `github-dark` | Shiki theme name (e.g., `dracula`, `one-dark-pro`, `catppuccin-mocha`) |
| `DIFF_SPLIT_MIN_WIDTH` | `150` | Minimum terminal columns to use split view |
| `DIFF_SPLIT_MIN_CODE_WIDTH` | `60` | Minimum code columns per side in split view |

### Example

```bash
# Use a different Shiki theme
export DIFF_THEME="catppuccin-mocha"

# Allow split view on narrower terminals
export DIFF_SPLIT_MIN_WIDTH=120
```

## Architecture

```
src/
├── index.ts    # Extension entry point — wraps write/edit tools with diff rendering
└── core/
    └── diff.ts # Diff parsing — converts old/new content to structured DiffLine[]
```

### Key internals

| Component | Purpose |
|-----------|---------|
| `parseDiff()` | Converts old/new content to structured `DiffLine[]` using `diff.structuredPatch` |
| `hlBlock()` | Shiki ANSI highlighting with LRU cache (192 entries) |
| `injectBg()` | Composites diff backgrounds into Shiki ANSI output (fg + bg layering) |
| `wordDiffAnalysis()` | Single-pass word diff → similarity score + character ranges |
| `renderSplit()` | Side-by-side renderer with diagonal stripe fillers |
| `renderUnified()` | Stacked single-column renderer |
| `wrapAnsi()` | ANSI-aware line wrapping with state carry-forward |
| `shouldUseSplit()` | Heuristic: split vs unified based on terminal width and wrap ratio |

### Rendering constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_PREVIEW_LINES` | 60 | Max lines in edit preview (split view) |
| `MAX_RENDER_LINES` | 150 | Max lines in write result (unified view) |
| `MAX_HL_CHARS` | 80,000 | Skip syntax highlighting above this |
| `CACHE_LIMIT` | 192 | LRU cache entries for highlighted blocks |
| `WORD_DIFF_MIN_SIM` | 0.15 | Minimum similarity for word-level emphasis |

## Exports

The extension exports a `__testing` object for unit testing:

```typescript
import { __testing } from "@heyhuynhgiabuu/pi-diff";

const { parseDiff, renderSplit, renderUnified, normalizeShikiContrast } = __testing;
```

## Development

```bash
git clone https://github.com/buddingnewinsights/pi-diff.git
cd pi-diff
npm install
npm run typecheck   # TypeScript validation
npm run lint        # Biome linting
npm test            # Run tests
```

### Load in pi for testing

```bash
# From the pi-diff directory
pi -e ./src/index.ts

# Or install globally
pi install .
```

## How pi Extensions Work

pi-diff is a **pi extension** — a TypeScript file that exports a default function receiving the pi API:

```typescript
export default function piDiffExtension(pi: any): void {
  // Get SDK tools
  const origWrite = createWriteTool(cwd);
  const origEdit = createEditTool(cwd);

  // Register enhanced versions
  pi.registerTool({
    ...origWrite,
    name: "write",
    execute: async (...) => { /* wrap + diff */ },
    renderCall: (...) => { /* preview */ },
    renderResult: (...) => { /* render diff */ },
  });
}
```

Extensions can:
- **Register tools** — `pi.registerTool(definition)`
- **Listen to events** — `pi.on("session_start" | "input" | "before_tool_call" | ...)`
- **Register commands** — `pi.registerCommand("/name", handler)`
- **Register providers** — `pi.registerProvider("name", config)`

See the [pi docs](https://pi.dev) for the full extension API.

## License

MIT — [huynhgiabuu](https://github.com/buddingnewinsights)
