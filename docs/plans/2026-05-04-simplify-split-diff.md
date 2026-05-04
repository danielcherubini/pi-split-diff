# Simplify to Split Diff Only — Plan

**Goal:** Strip the pi-diff project down to only the split/unified diff rendering feature — remove the entire review system, CLI, prompts, and theme preset infrastructure.

**Architecture:** A single extension file (`src/index.ts`) that wraps pi's `write` and `edit` tools, computes a structured diff via the `diff` library, syntax-highlights via Shiki, and renders side-by-side (split) or stacked (unified) output based on terminal width. Auto-derives diff colors from the pi theme with hardcoded fallbacks.

**Tech Stack:** TypeScript, `diff` library, `@shikijs/cli`, pi SDK (`@mariozechner/pi-coding-agent`, `@mariozechner/pi-tui`)

**Dependencies:** Task 1 → Task 2 (hard dependency: build is broken between them). Task 3 and Task 4 can run in any order after Task 2. Task 5 must be last.

---

### Task 1: Delete all review-related files

**Context:**
The project has a large review feature (interactive TUI overlay, session management, git diff reading, markdown export, CLI tool, prompt templates, and tests) that is no longer needed. All 27 files must be deleted. This is a pure deletion task — no code changes.

**Files:**
- Delete: `src/review/command.ts`
- Delete: `src/review/command.test.ts` (if exists, or any test variant)
- Delete: `src/review/export.ts`
- Delete: `src/review/export.test.ts`
- Delete: `src/review/file-preview.ts`
- Delete: `src/review/file-preview.test.ts`
- Delete: `src/review/git.ts`
- Delete: `src/review/git.test.ts`
- Delete: `src/review/hunk-preview.ts`
- Delete: `src/review/hunk-preview.test.ts`
- Delete: `src/review/interactive.ts`
- Delete: `src/review/interactive.test.ts`
- Delete: `src/review/model.ts`
- Delete: `src/review/model.test.ts`
- Delete: `src/review/prompt.ts`
- Delete: `src/review/session.ts`
- Delete: `src/review/session.test.ts`
- Delete: `src/review/tui.ts`
- Delete: `src/review/tui.test.ts`
- Delete: `src/cli.ts`
- Delete: `src/index.review-command.test.ts`
- Delete: `src/index.review-tools.test.ts`
- Delete: `prompts/review-diff-agent.md`
- Delete: entire `src/review/` directory (catch-all for anything missed above)
- Delete: entire `prompts/` directory

**What to implement:**
Nothing to implement. Delete every file listed above. Use `rm -rf` for directories.

**Steps:**
- [ ] Run `rm -rf src/review/`
- [ ] Run `rm -f src/cli.ts`
- [ ] Run `rm -f src/index.review-command.test.ts`
- [ ] Run `rm -f src/index.review-tools.test.ts`
- [ ] Run `rm -rf prompts/`
- [ ] Verify deletion: `find src/ prompts/ media/ -type f` should show NO review files, NO cli.ts, NO prompts directory
- [ ] Run `npm run build` — expect compilation errors (index.ts still imports from deleted files). This is expected and will be fixed in Task 2.
- [ ] Commit with message: "chore: remove all review-related files"

**Acceptance criteria:**
- [ ] `src/review/` directory does not exist
- [ ] `src/cli.ts` does not exist
- [ ] `prompts/` directory does not exist
- [ ] Review test files do not exist

---

### Task 2: Strip index.ts — remove review code and simplify theming

**Context:**
After deleting the review files, `src/index.ts` still contains imports, tool registrations, and theming infrastructure that references the deleted code. This task removes all review-related code from index.ts and simplifies the theme system from 5 layers (env vars → per-color overrides → presets → auto-derive → hardcoded) down to 2 layers (auto-derive from pi theme → hardcoded fallbacks).

**Files:**
- Modify: `src/index.ts`

**What to implement:**

1. **Remove all imports** that reference deleted modules:
   - `registerReviewDiffCommand` from `./review/command.js`
   - `formatReviewMarkdown` from `./review/export.js`
   - `countReviewDiffLines`, `ReviewDiffMode`, `readGitDiff` from `./review/git.js`
   - `applyDiffPalette as applySharedDiffPalette`, `lang as detectDiffLanguage`, `renderSplit as renderSharedSplit`, `resolveDiffColors as resolveSharedDiffColors`, `themeCacheKey as sharedThemeCacheKey` from `./review/hunk-preview.js`
   - `createReviewComment`, `formatInteractiveReviewPanel`, `formatReviewComments`, `ReviewComment` from `./review/interactive.js`
   - **KEEP** `import { existsSync, readFileSync } from "node:fs"` — these are used by the write/edit tool wrappers. Do NOT remove this import.

2. **Remove the entire preset/config system** (~300 lines):
   - `DiffPreset` interface
   - `DiffUserConfig` interface
   - `DIFF_PRESETS` record (default, midnight, subtle, neon)
   - `parseAnsiRgb()` — KEEP this (used by auto-derive)
   - `hexToBgAnsi()` — REMOVE (only used by preset config)
   - `hexToFgAnsi()` — REMOVE (only used by preset config)
   - `deriveBgFromFg()` — REMOVE (not used by auto-derive path)
   - `mixBg()` — KEEP this (used by auto-derive)
   - `loadDiffConfig()` — REMOVE entirely
   - `applyDiffPalette()` — REPLACE with a simple inline auto-derive call during extension init

3. **Simplify the theme initialization.** Replace the complex `applyDiffPalette()` + `_autoDerivePending` + `_hasExplicitBgConfig` system with:
   - A single function called once during extension registration that tries to auto-derive colors from the theme, falling back to hardcoded defaults
   - Remove `_autoDerivePending` and `_hasExplicitBgConfig` flags
   - The `resolveDiffColors()` function should still exist but be simpler — just read theme fg colors or return defaults

4. **Remove env var color override helpers:**
   - Remove `envFg()` and `envBg()` functions
   - Keep `envInt()` (used for `SPLIT_MIN_WIDTH`, etc.)
   - Keep `DIFF_THEME` env var for Shiki theme selection
   - Remove all `DIFF_BG_*` and `DIFF_FG_*` env var references from the ANSI color declarations — use hardcoded defaults directly

5. **Remove all three review tool registrations:**
   - `review_git_diff` tool (~200 lines)
   - `review_git_comment` tool (~100 lines)
   - `review_git_comments` tool (~80 lines)
   - Remove all associated interfaces: `ReviewGitDiffParams`, `ReviewGitCommentParams`, `ReviewGitCommentsParams`
   - Remove helper functions: `reviewGitDiffMode()`, `reviewGitDiffMaxLines()`, `normalizeOptionalPositiveInteger()`

6. **Remove `registerReviewDiffCommand(pi, cwd)` call** from the extension function

7. **Keep everything else intact:**
   - `parseDiff` import from `./core/diff.js`
   - Shiki: `codeToANSI` import, `hlBlock()`, LRU cache, pre-warm
   - ANSI utilities: `strip()`, `tabs()`, `termW()`, `fit()`, `ansiState()`, `wrapAnsi()`, `lnum()`, `shortPath()`, `summarize()`, `rule()`, `stripes()`
   - Word diff: `wordDiffAnalysis()`, `injectBg()`, `plainWordDiff()`
   - Renderers: `renderSplit()`, `renderUnified()`, `shouldUseSplit()`, `adaptiveWrapRows()`
   - Color system: `autoDeriveBgFromTheme()`, `resolveDiffColors()`, `themeCacheKey()`, `isLowContrastShikiFg()`, `normalizeShikiContrast()`
   - Constants: `SPLIT_MIN_WIDTH`, `MAX_PREVIEW_LINES`, `MAX_HL_CHARS`, `CACHE_LIMIT`, `WORD_DIFF_MIN_SIM`, `MAX_WRAP_ROWS_*`
   - Language detection: `EXT_LANG`, `lang()`
   - All ANSI escape constants: `RST`, `BOLD`, `DIM`, `BG_ADD`, `BG_DEL`, etc.
   - Write/edit tool wrapping (the extension's main purpose)
   - `__testing` exports (keep `normalizeShikiContrast`, `parseDiff`, `renderSplit`, `renderUnified`)

**Steps:**
- [ ] Remove all review-related imports from the top of `src/index.ts`
- [ ] Remove `existsSync`, `readFileSync` from `node:fs` import (remove the entire import line if those were the only imports from node:fs)
- [ ] Delete `DiffPreset` interface, `DiffUserConfig` interface
- [ ] Delete `DIFF_PRESETS` record
- [ ] Delete `hexToBgAnsi()`, `hexToFgAnsi()`, `deriveBgFromFg()`
- [ ] Delete `loadDiffConfig()` function
- [ ] Delete `applyDiffPalette()` function entirely
- [ ] Delete `_autoDerivePending` and `_hasExplicitBgConfig` variables
- [ ] Remove `envFg()` and `envBg()` functions
- [ ] Simplify ANSI color declarations — replace `envBg("DIFF_BG_ADD", "...")` with just the hardcoded ANSI string directly
- [ ] **Update aliased call sites.** After removing the hunk-preview imports, update all call sites that used the old aliases to use the local function names:
  - `detectDiffLanguage(fp)` → `lang(fp)` (appears ~4 times in write/edit tool wrappers)
  - `sharedThemeCacheKey(theme)` → `themeCacheKey(theme)` (~4 times)
  - `renderSharedSplit(...)` → `renderSplit(...)` (~3 times)
  - `resolveSharedDiffColors(theme)` → `resolveDiffColors(theme)` (~2 times)
- [ ] Remove all three review tool registrations (`review_git_diff`, `review_git_comment`, `review_git_comments`)
- [ ] Remove `ReviewGitDiffParams`, `ReviewGitCommentParams`, `ReviewGitCommentsParams` interfaces
- [ ] Remove `reviewGitDiffMode()`, `reviewGitDiffMaxLines()`, `normalizeOptionalPositiveInteger()` functions
- [ ] Remove `registerReviewDiffCommand(pi, cwd)` call from the extension function
- [ ] Simplify `resolveDiffColors()`. Remove `_autoDerivePending` and `_hasExplicitBgConfig` checks entirely. Replace with a simple one-time auto-derive flag:
  ```typescript
  let _didAutoDerive = false;
  function resolveDiffColors(theme?: any): DiffColors {
    if (!_didAutoDerive && theme?.getFgAnsi) {
      autoDeriveBgFromTheme(theme);
      _didAutoDerive = true;
    }
    if (!theme?.getFgAnsi) return DEFAULT_DIFF_COLORS;
    try {
      return {
        fgAdd: theme.getFgAnsi("toolDiffAdded") || FG_ADD,
        fgDel: theme.getFgAnsi("toolDiffRemoved") || FG_DEL,
        fgCtx: theme.getFgAnsi("toolDiffContext") || FG_DIM,
      };
    } catch {
      return DEFAULT_DIFF_COLORS;
    }
  }
  ```
- [ ] In the extension's `diffRendererExtension()` function, remove `applySharedDiffPalette()` call. Auto-derive happens lazily on first `resolveDiffColors()` call.
- [ ] Run `npm run build` — must compile without errors
- [ ] Run `npm run typecheck` — must pass
- [ ] Run `npm test` — the remaining test (`src/core/diff.test.ts`) must pass
- [ ] Run `npm run lint` — must pass (or fix any lint errors)
- [ ] Commit with message: "refactor: strip review code and simplify theming in index.ts"

**Acceptance criteria:**
- [ ] `npm run build` succeeds with zero errors
- [ ] `npm run typecheck` passes
- [ ] `npm test` passes (core diff tests only)
- [ ] No references to `review/`, `cli.ts`, or `prompts/` remain anywhere in the codebase
- [ ] No preset system code remains (`DIFF_PRESETS`, `loadDiffConfig`, `applyDiffPalette`)
- [ ] No env var color overrides remain (`envFg`, `envBg`, `DIFF_BG_*`, `DIFF_FG_*`)
- [ ] Auto-derive from theme still works (colors derived from `toolDiffAdded`/`toolDiffRemoved` theme keys)
- [ ] `renderSplit` and `renderUnified` are still exported via `__testing`
- [ ] The extension still registers write/edit tool wrappers

---

### Task 3: Update package.json

**Context:**
The package.json still references the CLI binary, prompts directory, and all review dist files. These need to be cleaned up so the published package only contains the diff rendering extension.

**Files:**
- Modify: `package.json`

**What to implement:**

1. Remove `"bin"` field entirely (the `pi-diff-review` CLI is gone)
2. Remove `"prompts"` from the `"pi"` section (leave only `"extensions"`)
3. Simplify `"files"` array — keep only:
   - `dist/index.d.ts`, `dist/index.d.ts.map`, `dist/index.js`, `dist/index.js.map`
   - `dist/core/diff.d.ts`, `dist/core/diff.d.ts.map`, `dist/core/diff.js`, `dist/core/diff.js.map`
   - `media/`, `README.md`, `LICENSE`
   - Remove ALL `dist/review/*` entries, `dist/cli.*` entries, and `prompts/` entry
4. Update `"description"` to: "Shiki-powered terminal diff renderer for pi — syntax-highlighted split and unified views with auto width detection."
5. Keep dependencies unchanged (`diff`, `@shikijs/cli`)
6. Keep peerDependencies unchanged
7. **Update build script:** Change `"build": "tsc && chmod +x dist/cli.js"` to just `"build": "tsc"`. The `chmod` will fail after cli.ts is deleted.

**Steps:**
- [ ] Edit `package.json` — remove `"bin"` field
- [ ] Edit `package.json` — remove `"prompts"` from `"pi"` section
- [ ] Edit `package.json` — simplify `"files"` array (remove all `dist/review/*`, `dist/cli.*`, and `prompts/` entries)
- [ ] Edit `package.json` — update `"description"`
- [ ] Edit `package.json` — change `"build"` script from `"tsc && chmod +x dist/cli.js"` to just `"tsc"`
- [ ] Run `npm run build` — verify dist files match the new `"files"` list
- [ ] Verify: `ls dist/` should contain only `index.*` and `core/diff.*` (no `review/` or `cli.*`)
- [ ] Commit with message: "chore: clean up package.json for split-diff-only release"

**Acceptance criteria:**
- [ ] `package.json` has no `"bin"` field
- [ ] `package.json` `"pi"` section has only `"extensions"` (no `"prompts"`)
- [ ] `package.json` `"files"` contains no `dist/review/*` or `prompts/` paths
- [ ] `npm run build` produces only `dist/index.*` and `dist/core/diff.*`

---

### Task 4: Rewrite README.md

**Context:**
The README documents the full project including review features, CLI, prompts, and theme presets. It needs to be rewritten to document only the split/unified diff rendering extension.

**Files:**
- Modify: `README.md`

**What to implement:**

Write a new README.md with these sections only:

1. **Title + badge line** — keep npm version badge, update description
2. **Screenshots** — keep `media/split.png` and `media/unified.png` (remove review-diff.png reference)
3. **Features** bullet list — syntax-highlighted diffs, split view with auto-fallback, unified view, word-level emphasis, adaptive layout, LRU cache, large diff fallback, auto-derive colors from theme
4. **Install** section — `pi install npm:@heyhuynhgiabuu/pi-diff` and dev loading
5. **How It Works** section — the rendering pipeline diagram, views table (split vs unified)
6. **Configuration** section — simplified to:
   - `DIFF_THEME` env var for Shiki theme
   - `DIFF_SPLIT_MIN_WIDTH` and `DIFF_SPLIT_MIN_CODE_WIDTH` env vars for layout thresholds
   - Auto-derive explanation (no presets, no per-color overrides)
7. **Architecture** section — simplified file tree (`src/index.ts`, `src/core/diff.ts`), key internals table, rendering constants table
8. **Exports** section — `__testing` object
9. **Development** section — unchanged
10. **How pi Extensions Work** section — keep as-is (useful context)
11. **License** — unchanged

Remove entirely: all `/review-diff`, `review_git_diff`, `review_git_comment`, `review_git_comments`, `pi-diff-review` CLI, prompt template, key bindings, session persistence documentation.

**Steps:**
- [ ] Delete `media/review-diff.png` (do this alongside the README rewrite so the repo isn't in an inconsistent state with a missing image reference)
- [ ] Write the new README.md with sections listed above
- [ ] Verify no references to review features remain
- [ ] Verify screenshot references only point to `media/split.png` and `media/unified.png`
- [ ] Commit with message: "docs: rewrite README for split-diff-only project"

**Acceptance criteria:**
- [ ] README has no mention of `/review-diff`, `review_git_diff`, CLI, or prompts
- [ ] README documents split view, unified view, auto-fallback, and configuration
- [ ] README screenshots reference only existing files

---

### Task 5: Final verification

**Context:**
After all changes, verify the project builds cleanly, tests pass, and the file structure matches expectations.

**Files:**
- No new files. Verification only.

**Steps:**
- [ ] Run `npm run build` — must succeed
- [ ] Run `npm run typecheck` — must pass
- [ ] Run `npm test` — must pass
- [ ] Run `npm run lint` — must pass
- [ ] Verify file structure:
  ```
  src/
  ├── index.ts
  └── core/
      ├── diff.ts
      └── diff.test.ts
  media/
  ├── split.png
  └── unified.png
  ```
- [ ] Verify `dist/` contains only `index.*` and `core/diff.*`
- [ ] Run `grep -rn "from.*review\\|registerReviewDiffCommand\\|ReviewComment\\|ReviewDiff\\|review_git" src/` — should find zero matches. A broad `grep -r "review"` is too noisy (will match comments).
- [ ] Commit with message: "verify: final check after simplification"

**Acceptance criteria:**
- [ ] All build, typecheck, test, and lint commands pass
- [ ] File structure matches the expected layout above
- [ ] No stray references to deleted code remain
