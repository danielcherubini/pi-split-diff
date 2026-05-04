# pi-diff

[![npm version](https://img.shields.io/npm/v/@heyhuynhgiabuu/pi-diff)](https://www.npmjs.com/package/@heyhuynhgiabuu/pi-diff)

A [pi](https://pi.dev) extension that replaces the default `write` and `edit` tool output with **Shiki-powered, syntax-highlighted diffs** rendered directly in your terminal.

## Screenshots

### Split view (side-by-side)

![pi-diff split view](media/split.png)

### Unified view (stacked)

![pi-diff unified view](media/unified.png)

## Features

- **Syntax-highlighted diffs** — Shiki grammar highlighting (190+ languages) composited with diff background colors
- **Split view** — side-by-side comparison for `edit` tool, auto-falls back to unified on narrow terminals
- **Unified view** — stacked single-column layout for `write` tool overwrites
- **Word-level emphasis** — brighter backgrounds on changed characters so you see exactly what changed
- **Adaptive layout** — auto-detects terminal width; wraps on wide terminals, truncates on narrow ones
- **LRU cache** — singleton Shiki highlighter with 192-entry cache for fast re-renders
- **Large diff fallback** — gracefully degrades (skips highlighting, still shows diff structure) for files > 80k chars
- **Auto-derive colors** — derives diff colors from your pi theme automatically, no configuration needed

## Installation

```bash
pi install npm:@heyhuynhgiabuu/pi-diff
```

### Development

Load directly from source:

```bash
pi -e ./src/index.ts
```

## How It Works

pi-diff wraps the built-in `write` and `edit` tools from the pi SDK. When the agent writes or edits a file:

1. **Before the write** — reads the existing file content
2. **Delegates** to the original SDK tool (file is actually written)
3. **After the write** — computes a structured diff between old and new content
4. **Renders** the diff with syntax highlighting and word-level emphasis

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
| **Split** | `edit` tool | Side-by-side with old on left, new on right. Diagonal stripes fill empty slots. Auto-falls back to unified when terminal is < 150 cols or > 20% of lines would wrap. |
| **Unified** | `write` tool | Single column with `+`/`-` gutter. Compact, works at any terminal width. |

Both views show colored border bars (`▌`) for changed lines, line numbers in the gutter, hunk separators, and word-level emphasis on paired add/del lines.

## Configuration

### Auto-Derive (Default)

pi-diff **automatically derives** diff colors from your pi theme's diff foreground colors and tool-state backgrounds. Added/context surfaces use `toolSuccessBg`; removed surfaces use `toolErrorBg`. This ensures diffs look good with any pi theme and terminal background — no configuration needed.

The auto-derive uses different intensity levels:

- **Line backgrounds**: 8–10% of the theme's diff fg color (subtle tint)
- **Word highlights**: 20–22% (more visible for changed characters)
- **Gutters**: 5–6% (subtler than line backgrounds)

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DIFF_THEME` | `github-dark` | Shiki theme name (e.g. `dracula`, `one-dark-pro`, `catppuccin-mocha`) |
| `DIFF_SPLIT_MIN_WIDTH` | `150` | Minimum terminal columns to use split view |
| `DIFF_SPLIT_MIN_CODE_WIDTH` | `60` | Minimum code columns per side in split view |

```bash
# Use a different Shiki theme
export DIFF_THEME="catppuccin-mocha"

# Allow split view on narrower terminals
export DIFF_SPLIT_MIN_WIDTH=120
```

## Development

```bash
git clone https://github.com/buddingnewinsights/pi-diff.git
cd pi-diff
npm install
```

| Script | Description |
|--------|-------------|
| `npm run build` | Compile TypeScript |
| `npm run typecheck` | TypeScript validation |
| `npm run lint` | Biome linting |
| `npm test` | Run tests (vitest) |

Load in pi for testing:

```bash
pi -e ./src/index.ts
```
