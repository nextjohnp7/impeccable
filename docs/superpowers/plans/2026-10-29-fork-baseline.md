# Fork Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `nextjohnp7/impeccable` gets a provably green build, a written sync policy, a per-line quality gate in CI, and diffs that are not flooded with generated files.

**Architecture:** Three additions, all inside fork-owned files (`docs/FORK.md`, `scripts/fork/**`, `tests/fork/**`, `.github/workflows/fork-gates.yml`) plus one comment line in `.gitattributes`. Nothing touches `crates/`, `skill/`, `package.json`, `bun.lock`, `rust-toolchain.toml`, or `ci.yml`, so rebasing onto upstream stays cheap. The gate works by taking changed line ranges from `git diff -U0`, normalising `rustfmt`/`clippy`/`biome` findings into `{file, line}`, and failing only on findings that fall inside a changed line.

**Tech Stack:** Node ESM (`node:test`, `node:assert/strict`), `child_process`, GitHub Actions, rustfmt/clippy (`cargo`), Biome.

**Spec:** `docs/superpowers/specs/2026-10-29-fork-baseline-design.md`

## Global Constraints

- No behaviour change: `cargo test --workspace` and `tests/oracle/run.mjs` stay green; `docs/CLI-CONTRACT.md` is unchanged.
- `main` must stay identical to `upstream/main` (`git rev-list --count upstream/main..main` = `0`). All commits live on `fork/*`.
- Do not modify: `package.json`, `bun.lock`, `rust-toolchain.toml`, `.github/workflows/ci.yml`, anything under `crates/`, `skill/`, `cli/`, `extension/`, or the generated provider directories.
- No new npm dependencies. The gate uses `npx --yes @biomejs/biome@1.9.4`, not `devDependencies`.
- Style: ESM, semicolons, 2-space indent, kebab-case filenames, `node:test` + `node:assert/strict`, `ROOT` computed with `fileURLToPath(new URL('..', import.meta.url))`.
- Node floor is 18.19 (from `package.json` `engines`); CI uses Node 24.
- Pinned versions live only in fork-owned files: Rust `1.99.1`, Biome `1.9.4`, Bun from `.bun-version` (`1.3.13`).
- Commit subjects are imperative, matching the repo: `Add: ...`, `Fix: ...`.
- The gate's default base ref is `origin/fork/main`; it can be overridden with `--base` or `IMPECCABLE_FORK_BASE`.

## Review Focus

1. **Pre-existing findings on unchanged lines** (the most likely reason the gate ends up unused): a file that already had violations before the fork touched it must still pass. Test in Task 5.
2. **Windows checkouts**: a CRLF working tree and git-quoted paths (`"b/folder with space/f.rs"`) must not corrupt paths or line numbers. Test in Task 3.
3. **New files and renames**: ranges must attach to the new path; a new file means every line is changed. Test in Task 3.
4. **Missing tools** (no `cargo`/`rustfmt`/Biome): the gate must skip with a note and exit `0`, not block contributors. Test in Task 5.
5. **Binary files, deleted files, and mode-only changes** in the diff: must not crash and must not produce phantom ranges. Tests in Task 3 and Task 5.

---

### Task 1: Toolchain and a green baseline

**Files:** no files changed. The verification results are recorded in Task 2.

**Interfaces:**
- Consumes: nothing.
- Produces: a local environment with `bun 1.3.13`, `rustc 1.99.1`, the `wasm32-unknown-unknown` target, and Chrome for Puppeteer.

- [ ] **Step 1: Install the pinned Bun**

Run (PowerShell): `irm bun.sh/install.ps1 | iex`, then open a new shell.
Verify: `bun --version` → `1.3.13`. If it differs, run `bun upgrade --to 1.3.13`.

- [ ] **Step 2: Install Rust and the wasm target**

Run: `winget install Rustlang.Rustup` (or `rustup-init`), then `rustup toolchain install 1.99.1 --profile minimal --component rustfmt,clippy` and `rustup target add wasm32-unknown-unknown --toolchain 1.99.1`.
Verify: `RUSTUP_TOOLCHAIN=1.99.1 rustc --version` → `rustc 1.99.1`; `rustup target list --installed --toolchain 1.99.1` includes `wasm32-unknown-unknown`.

Installing a toolchain does not select it: `rust-toolchain.toml` pins `stable`, so a bare `cargo` keeps using stable. Export `RUSTUP_TOOLCHAIN=1.99.1` for every cargo and rustfmt command in this plan, including the baseline below, so local runs match `fork-gates`. Do not edit `rust-toolchain.toml`.

- [ ] **Step 3: Install Chrome for the detector tests**

Run: `bunx --bun puppeteer browsers install chrome`.
Verify: the command exits `0` and prints a binary path.

- [ ] **Step 4: Prove this tree is green before changing it**

With `RUSTUP_TOOLCHAIN=1.99.1` exported, run in order: `bun install`, then `bun run build`, then `cargo build --workspace --all-targets`, then `cargo test --workspace` (let it run in the background; ~10 minutes according to `AGENTS.md`), then `bun run fetch:engine`, then `bun run test`.
Expected: everything exits `0`. If something fails **before** any task below is done, stop and report it — that is the upstream baseline, not a mistake in this plan.

- [ ] **Step 5: Record the results**

Keep the output of `bun --version`, `rustc --version`, and the exit code of every command; they are used in Task 2 Step 3.

---

### Task 2: Fork policy and the integration branch

**Files:**
- Create: `docs/FORK.md`
- Create: `tests/fork-docs.test.mjs`
- Modify: nothing

**Interfaces:**
- Consumes: the results of Task 1 (the Bun/Rust versions actually installed).
- Produces: the document referenced by Tasks 6 and 7 (branch names, base ref, workflows that must be disabled); the `fork/main` branch on origin as the PR target.

- [ ] **Step 1: Write the failing test**

`tests/fork-docs.test.mjs`:

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import { fileURLToPath } from 'node:url';

const ROOT = fileURLToPath(new URL('..', import.meta.url));
const doc = () => readFileSync(join(ROOT, 'docs/FORK.md'), 'utf-8').replace(/\r\n?/g, '\n');

describe('docs/FORK.md', () => {
  it('names the three branch roles', () => {
    const text = doc();
    for (const needle of ['`main`', '`fork/main`', '`fork/<topic>`']) {
      assert.ok(text.includes(needle), `FORK.md should mention ${needle}`);
    }
  });

  it('records the upstream remote and the mirror invariant', () => {
    const text = doc();
    assert.match(text, /https:\/\/github\.com\/pbakaus\/impeccable\.git/);
    assert.match(text, /git rev-list --count upstream\/main\.\.main/);
  });

  it('names the two workflows to disable and why', () => {
    const text = doc();
    assert.ok(text.includes('sync-generated-output.yml'));
    assert.ok(text.includes('sheriff.yml'));
  });

  it('documents the gate base ref and the JS comment convention', () => {
    const text = doc();
    assert.ok(text.includes('IMPECCABLE_FORK_BASE'));
    assert.match(text, /\/\/! JS:/);
  });

  it('states the Bun version from .bun-version', () => {
    const pin = readFileSync(join(ROOT, '.bun-version'), 'utf-8').replace(/\r\n?/g, '\n').trim();
    assert.ok(doc().includes(pin), `FORK.md should state Bun ${pin}`);
  });
});
```

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork-docs.test.mjs`
Expected: FAIL — `ENOENT` for `docs/FORK.md`.

- [ ] **Step 3: Write `docs/FORK.md`**

Required content, any order: what this fork is and is not (it is not a request to PR upstream); the branch model (`main` mirrors, `fork/main` integrates, `fork/<topic>` is where work happens); the sync commands (`git fetch upstream`, `git switch main`, `git merge --ff-only upstream/main`, `git push origin main`) and the invariant `git rev-list --count upstream/main..main` → `0`; the rebase commands for a work branch; the two workflows disabled through the Actions tab and why; local prerequisites with the versions from Task 1; verification commands (`cargo test --workspace`, `bun run test`, `node --test tests/fork`); the gate base ref (`IMPECCABLE_FORK_BASE`, default `origin/fork/main`); the `RUSTUP_TOOLCHAIN=1.99.1` override and why it is needed (the repo pins `stable`, and installing a toolchain does not select it); a note that the 1,483 `//! JS:` comments point at upstream JS sources, not at files in this checkout; the "no behaviour change" rule and the list of files that must not be touched.

- [ ] **Step 4: Run the test, confirm it passes**

Run: `node --test tests/fork-docs.test.mjs`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add docs/FORK.md tests/fork-docs.test.mjs
git commit -m "Add: fork policy doc and its consistency test"
```

- [ ] **Step 6: Create the integration branch**

```bash
git branch fork/main main
git push -u origin fork/main
```
Expected: `* [new branch] fork/main -> fork/main`. From here on, work-branch PRs target `fork/main`, not `main`.

- [ ] **Step 7: Disable the two upstream workflows that break the branch model**

If `gh` is available and authenticated for the fork:

```bash
gh workflow disable sync-generated-output.yml
gh workflow disable sheriff.yml
```

If not: GitHub → fork repo → **Actions** tab → select the workflow → **...** menu → **Disable workflow**. Only repo settings change; no file is edited, so there is no conflict with upstream.

Verify: `gh workflow list --all` shows both as `disabled_manually`.

---

### Task 3: Changed-line range library

**Files:**
- Create: `scripts/fork/lib/changed-ranges.mjs`
- Create: `tests/fork/changed-ranges.test.mjs`

**Interfaces:**
- Consumes: nothing.
- Produces (used by Task 5):
  - `parseUnifiedDiff(diffText: string) -> Map<string, Array<[number, number]>>` — keys are root-relative posix paths, values are inclusive line ranges on the new side, sorted ascending.
  - `isChanged(ranges: Map<string, Array<[number, number]>>, file: string, line: number) -> boolean`
  - `toPosix(p: string) -> string` — `\` → `/`.
  - `unescapeGitPath(raw: string) -> string` — strips surrounding quotes and C-style escapes (`\"`, `\\`, `\t`, `\n`, octal `\ooo`).

- [ ] **Step 1: Write the failing test**

`tests/fork/changed-ranges.test.mjs` — include these cases (exact assertions):

```js
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { parseUnifiedDiff, isChanged, toPosix, unescapeGitPath } from '../../scripts/fork/lib/changed-ranges.mjs';

describe('parseUnifiedDiff', () => {
  it('maps a simple hunk to inclusive new-side ranges', () => {
    const map = parseUnifiedDiff([
      'diff --git a/src/a.rs b/src/a.rs',
      '--- a/src/a.rs',
      '+++ b/src/a.rs',
      '@@ -10,2 +10,3 @@',
    ].join('\n'));
    assert.deepEqual(map.get('src/a.rs'), [[10, 12]]);
  });

  it('ignores deletion-only hunks', () => {
    const map = parseUnifiedDiff(['--- a/src/a.rs', '+++ b/src/a.rs', '@@ -5,3 +5,0 @@'].join('\n'));
    assert.equal(map.has('src/a.rs'), false);
  });

  it('keeps the new path of a rename', () => {
    const map = parseUnifiedDiff([
      'diff --git a/src/old.rs b/src/new.rs',
      'rename from src/old.rs',
      'rename to src/new.rs',
      '--- a/src/old.rs',
      '+++ b/src/new.rs',
      '@@ -1,0 +1,4 @@',
    ].join('\n'));
    assert.deepEqual([...map.keys()], ['src/new.rs']);
    assert.deepEqual(map.get('src/new.rs'), [[1, 4]]);
  });

  it('treats a newly added file as fully changed', () => {
    const map = parseUnifiedDiff(['--- /dev/null', '+++ b/src/new.rs', '@@ -0,0 +1,7 @@'].join('\n'));
    assert.deepEqual(map.get('src/new.rs'), [[1, 7]]);
  });

  it('skips deletions and binary files', () => {
    const map = parseUnifiedDiff([
      'diff --git a/gone.rs b/gone.rs',
      '--- a/gone.rs',
      '+++ /dev/null',
      '@@ -1,5 +0,0 @@',
      'diff --git a/logo.png b/logo.png',
      'Binary files a/logo.png and b/logo.png differ',
    ].join('\n'));
    assert.equal(map.size, 0);
  });

  it('unescapes quoted paths', () => {
    const map = parseUnifiedDiff(['--- "a/dir with space/f.rs"', '+++ "b/dir with space/f.rs"', '@@ -1,1 +1,2 @@'].join('\n'));
    assert.deepEqual(map.get('dir with space/f.rs'), [[1, 2]]);
  });

  it('parses a CRLF diff', () => {
    const map = parseUnifiedDiff([
      'diff --git a/src/a.rs b/src/a.rs',
      '--- a/src/a.rs',
      '+++ b/src/a.rs',
      '@@ -1,1 +1,2 @@',
    ].join('\r\n'));
    assert.deepEqual([...map.keys()], ['src/a.rs']);
    assert.deepEqual(map.get('src/a.rs'), [[1, 2]]);
  });
});

describe('isChanged', () => {
  const map = new Map([['src/a.rs', [[10, 12], [40, 41]]]]);
  it('is true inside a range and false outside it', () => {
    assert.equal(isChanged(map, 'src/a.rs', 10), true);
    assert.equal(isChanged(map, 'src/a.rs', 12), true);
    assert.equal(isChanged(map, 'src/a.rs', 13), false);
    assert.equal(isChanged(map, 'src/a.rs', 39), false);
  });
  it('is false for unknown files', () => {
    assert.equal(isChanged(map, 'src/b.rs', 10), false);
  });
});

describe('path helpers', () => {
  it('normalises windows separators', () => {
    assert.equal(toPosix('crates\\core\\src\\lib.rs'), 'crates/core/src/lib.rs');
  });
  it('unescapes git quoting without stripping the b/ prefix', () => {
    assert.equal(unescapeGitPath('"b/a\\tb\\303\\251.rs"'), 'b/a\tb\u00e9.rs');
  });
});
```

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork/changed-ranges.test.mjs`
Expected: FAIL — `Cannot find module .../scripts/fork/lib/changed-ranges.mjs`.

- [ ] **Step 3: Implement `scripts/fork/lib/changed-ranges.mjs`**

Binding rules: ignore `+++ /dev/null` lines; do not create an entry for a hunk whose `+count` is `0`; take the path from `+++ b/...` (not from `diff --git`); strip the `b/` prefix only if present; support multiple hunks per file; octal escapes in `unescapeGitPath` are read as bytes and then UTF-8 decoded (`Buffer.from(bytes)`). Strip a trailing `\r` from every header line so a CRLF diff parses exactly like an LF one. `unescapeGitPath` decodes quotes and escapes only; the `b/` prefix is stripped afterwards, inside `parseUnifiedDiff`.

- [ ] **Step 4: Run the test, confirm it passes**

Run: `node --test tests/fork/changed-ranges.test.mjs`
Expected: PASS. On Windows, the quoted-path test is the one that catches CRLF/backslash bugs — do not skip it.

- [ ] **Step 5: Commit**

```bash
git add scripts/fork/lib/changed-ranges.mjs tests/fork/changed-ranges.test.mjs
git commit -m "Add: changed-line range parser for fork gate"
```

---

### Task 4: Normalise tool output into findings

**Files:**
- Create: `scripts/fork/lib/tool-output.mjs`
- Create: `tests/fork/tool-output.test.mjs`

**Interfaces:**
- Consumes: `toPosix` and `unescapeGitPath` from Task 3.
- Produces (used by Task 5): `Finding = { file: string, line: number, tool: 'rustfmt'|'clippy'|'biome', message: string }`
  - `parseRustfmtCheck(stdout: string, repoRoot: string) -> Finding[]`
  - `parseClippyJson(stdout: string, repoRoot: string) -> Finding[]`
  - `parseBiomeJson(stdout: string, repoRoot: string) -> Finding[]`
  - `byteOffsetToLine(text: string, byteOffset: number) -> number` (1-based)
  - `normalizePath(p: string, repoRoot: string) -> string | null` — `null` when the path is outside the root, a root-relative posix path when inside.

- [ ] **Step 1: Write the failing test**

Cases and exact assertions:

```js
// rustfmt: repeated "Diff in <path> at line <n>:" headers, absolute paths, multiple hunks
parseRustfmtCheck('Diff in /repo/src/a.rs at line 12:\n-fn x(){}\n+fn x() {}\nDiff in /repo/src/a.rs at line 40:\n', '/repo')
// => [{file:'src/a.rs',line:12,tool:'rustfmt'},{file:'src/a.rs',line:40,tool:'rustfmt'}]

// clippy: non-JSON lines are ignored; only reason 'compiler-message', level warning/error, primary span
// clippy: both an absolute file_name and one relative to repoRoot must yield 'src/a.rs'
// clippy: a span outside repoRoot -> the Finding is dropped (no throw)

// biome: location.path.file may be a plain path; span [startByte,endByte] is mapped to a line via the file contents
// biome: byteOffsetToLine('a\nb\nc', 3) === 2 and byteOffsetToLine('é\nx', 3) === 2 (bytes, not chars)

// normalizePath: 'crates\\core\\src\\lib.rs' -> 'crates/core/src/lib.rs'; '../outside.rs' -> null
```

Write these as separate `it(...)` blocks with inline string fixtures; do not download real tool output in the test.

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork/tool-output.test.mjs`
Expected: FAIL — the module does not exist yet.

- [ ] **Step 3: Implement `scripts/fork/lib/tool-output.mjs`**

Rules: `byteOffsetToLine` slices `Buffer.from(text, 'utf8')` at `byteOffset` and counts `\n`; `normalizePath` uses `path.resolve(repoRoot, p)` then `path.relative`, and returns `null` for paths that escape the root or cannot be resolved; no parser may throw on an unrecognised line (ignore it).

- [ ] **Step 4: Match against real tool output**

Run `rustfmt --check` on a Rust file whose formatting you deliberately broke (in `$TEMP`, not in the repo) and `npx --yes @biomejs/biome@1.9.4 check --reporter=json` on a deliberately broken JS file (also in `$TEMP`, with `biome.json` copied over). Compare the output shape with the fixture. If the shape differs from the fixture, fix the parser **and** the fixture in the same commit; do not weaken the assertion.

- [ ] **Step 5: Run the test, confirm it passes**

Run: `node --test tests/fork/tool-output.test.mjs`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add scripts/fork/lib/tool-output.mjs tests/fork/tool-output.test.mjs
git commit -m "Add: tool output normalisers for fork gate"
```

---

### Task 5: Per-line gate CLI

**Files:**
- Create: `scripts/fork/check-changed.mjs`
- Create: `tests/fork/check-changed.test.mjs`

**Interfaces:**
- Consumes: `parseUnifiedDiff`, `isChanged` (Task 3); `parseRustfmtCheck`, `parseClippyJson`, `parseBiomeJson` (Task 4).
- Produces (used by Task 6): CLI `node scripts/fork/check-changed.mjs [--base <ref>] [--tools rustfmt,clippy,biome] [--json]`; exit `0` clean/skipped, `1` findings on changed lines, `2` base ref not found. The module exports `runCheck({ cwd, baseRef, tools, exec, log })`, returning `{ findings, exitCode, skipped }`, with an injectable `exec(cmd, args, opts) -> { status, stdout, stderr }` for tests.

- [ ] **Step 1: Write the failing test**

Five cases, one `it` each, using `runCheck` with a fake `exec` and `cwd` = a temporary repo directory created with `git init`:

```js
// helper: temporary repo with one base commit, then a committed/staged change
// 1. rustfmt violation on a changed line -> exitCode 1, findings[0].file === 'src/a.rs'
// 2. rustfmt violation on an UNCHANGED line (hunk lands on line 1, the change is on line 50) -> exitCode 0
// 3. exec throws ENOENT for rustfmt/cargo/npx -> exitCode 0 and skipped contains 'rustfmt','clippy','biome'
// 4. no changed files -> exitCode 0 and exec is never called (count the calls)
// 5. only non-code files (docs/x.md) changed -> clippy and rustfmt are not called
```

Case 2 is the core of the product: write its diff fixture explicitly; do not rely on the real repo.

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork/check-changed.test.mjs`
Expected: FAIL — `runCheck` does not exist yet.

- [ ] **Step 3: Implement `scripts/fork/check-changed.mjs`**

Rules: `baseRef` defaults to `process.env.IMPECCABLE_FORK_BASE || 'origin/fork/main'`; resolve the base with `git merge-base <baseRef> HEAD` (exit `2` on failure); take the diff with `git diff -U0 --no-color <mergeBase>`; add untracked files (`git ls-files --others --exclude-standard`) as `[[1, Number.MAX_SAFE_INTEGER]]`; run `rustfmt --check` per changed `.rs` file, `cargo clippy -p <crate> --all-targets --message-format=json` per changed crate (the package name comes from `[package] name` in `crates/<dir>/Cargo.toml`), and `npx --yes @biomejs/biome@1.9.4 check --reporter=json` per changed JS/TS/JSON file; filter every finding through `isChanged`; print `path:line tool: message` followed by the summary `N finding(s) in changed lines`. External commands that do not exist (`status === null` or `err.code === 'ENOENT'`) go into `skipped`, not into a failure.

- [ ] **Step 4: Run the test, confirm it passes**

Run: `node --test tests/fork/check-changed.test.mjs`
Expected: PASS (5 tests).

- [ ] **Step 5: Run the gate on this repo**

Run: `node scripts/fork/check-changed.mjs --base upstream/main`
Expected: exit `0` — the changes so far are documents and new, already-formatted files. If it exits `1`, the finding is real: fix it, do not weaken the gate.

- [ ] **Step 6: Commit**

```bash
git add scripts/fork/check-changed.mjs tests/fork/check-changed.test.mjs
git commit -m "Add: per-line quality gate for fork"
```

---

### Task 6: Fork CI workflow

**Files:**
- Create: `.github/workflows/fork-gates.yml`
- Create: `tests/fork-gates-workflow.test.mjs`

**Interfaces:**
- Consumes: the Task 5 CLI; base ref `origin/fork/main`.
- Produces: two required checks, `gates` and `suite`, for PRs into `fork/main`.

- [ ] **Step 1: Write the failing test**

```js
// exact assertions:
// - every `uses:` in the file matches /@[a-f0-9]{40}$/  (the same rule as tests/workflow-security.test.js)
// - the text contains 'node-version: 24'
// - the text contains 'rustup toolchain install 1.99.1 --profile minimal --component rustfmt,clippy'
// - the text contains 'RUSTUP_TOOLCHAIN: 1.99.1'
// - the text contains 'node scripts/fork/check-changed.mjs'
// - the text contains 'permissions:' followed by 'contents: read'
// - the text contains 'branches: [fork/main]' and 'pull_request:'
// - the text contains 'fetch-depth: 0'
// - the text contains 'suite:' and 'cargo test --workspace'
```

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork-gates-workflow.test.mjs`
Expected: FAIL — `ENOENT` for the workflow.

- [ ] **Step 3: Write `.github/workflows/fork-gates.yml`**

Structure: `name: fork-gates`; triggers `pull_request` and `push` on `fork/main`; `permissions: { contents: read }`; two jobs on `ubuntu-latest`; a checkout step with `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1` and `fetch-depth: 0`; Node setup with `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1 # v7.1.0` and `node-version: 24`; `rustup toolchain install 1.99.1 --profile minimal --component rustfmt,clippy`; job-level `env: RUSTUP_TOOLCHAIN: 1.99.1`.

Job `gates` runs `node scripts/fork/check-changed.mjs --base origin/fork/main`. Job `suite` exists because `ci.yml` only triggers on `main`, so a PR into `fork/main` would otherwise get no build or test coverage at all: it runs `bun install`, `bun run build`, `cargo test --workspace`, `bun run fetch:engine`, `bun run test`, in that order. Do not copy the upstream Node matrix, the Windows legs, or live-e2e into this file; those stay in `ci.yml` and apply to `main` only. Do not touch `ci.yml`.

- [ ] **Step 4: Run the new test and the existing workflow security test**

Run: `node --test tests/fork-gates-workflow.test.mjs` → PASS.
Run: `bun test tests/workflow-security.test.js` → PASS (confirms our SHA pins satisfy the repo rule).

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/fork-gates.yml tests/fork-gates-workflow.test.mjs
git commit -m "Add: fork-gates workflow"
```

- [ ] **Step 6: Prove the gate fails a violation**

Create a temporary branch from `fork/main`, add one badly formatted Rust line to an existing `.rs` file, push it, and open a PR into `fork/main`. Expected: the `gates` check is red with a `path:line` message. Then delete that branch (do not merge). This is the only way to prove the gate is not vacuous.

---

### Task 7: Hide generated directories from diffs

**Files:**
- Modify: `.gitattributes` (the comment block on lines 38–40, plus 25 lines on 41–65)
- Create: `tests/fork-gitattributes.test.mjs`

**Interfaces:**
- Consumes: nothing.
- Produces: `git check-attr diff <path>` returns `unset` for all provider output.

- [ ] **Step 1: Write the failing test**

```js
// for every path in the list of 25 below, run:
//   git check-attr diff <path>  -> stdout ends with ': diff: unset'
// and assert precision:
//   git check-attr diff crates/live/assets/antipatterns.json -> ': diff: unspecified'
```

The list of 25: `.agent`, `.agents`, `.codex`, `.claude`, `.cursor`, `.devin`, `.dsh`, `.gemini`, `.github/agents`, `.github/hooks`, `.github/skills`, `.grok`, `.hermes`, `.kiro`, `.opencode`, `.pi`, `.omp`, `.qoder`, `.rovodev`, `.trae`, `.trae-cn`, `.veto`, `.vibe`, `plugin`, `cursor-plugin` (all `/**`).

- [ ] **Step 2: Run the test, confirm it fails**

Run: `node --test tests/fork-gitattributes.test.mjs`
Expected: FAIL — still `unspecified` for every path.

- [ ] **Step 3: Change `.gitattributes`**

Append ` -diff` to those 25 lines, and update the comment block on lines 38–40 so it states that provider output is now also `-diff` (not "Collapsed on GitHub only"). Do not change other lines and do not reorder them.

- [ ] **Step 4: Run the test, confirm it passes**

Run: `node --test tests/fork-gitattributes.test.mjs`
Expected: PASS.

- [ ] **Step 5: Make sure the release build stays clean**

Run: `bun run build:release`, then `git status --porcelain`.
Expected: empty. If something changed, those are artifacts that genuinely belong (`build:release` syncs the provider folders) — review them, then commit them separately with a clear `Add:`/`Update:`.

- [ ] **Step 6: Commit**

```bash
git add .gitattributes tests/fork-gitattributes.test.mjs
git commit -m "Fix: hide generated provider diffs"
```

---

## Final verification (before the PR into `fork/main`)

- [ ] `node --test tests/fork` → all PASS
- [ ] `node scripts/fork/check-changed.mjs --base upstream/main` → exit `0`
- [ ] `bun test tests/workflow-security.test.js tests/readme-gitignore.test.mjs` → PASS (no regressions)
- [ ] `cargo test --workspace` → exit `0`
- [ ] `git rev-list --count upstream/main..main` → `0`
- [ ] A PR into `fork/main` shows both `gates` and `suite` green
- [ ] `git diff --stat upstream/main fork/main` touches only: `docs/FORK.md`, `docs/superpowers/**`, `scripts/fork/**`, `tests/fork*.test.mjs`, `tests/fork/**`, `.github/workflows/fork-gates.yml`, `.gitattributes`

Do not open a PR against `pbakaus/impeccable`. PRs target this fork's `fork/main` only.
