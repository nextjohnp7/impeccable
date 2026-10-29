# Fork Baseline — Impeccable

Status: approved 2026-10-29 — implementation plan: `docs/superpowers/plans/2026-10-29-fork-baseline.md`
Date: 2026-10-29
Repo: `nextjohnp7/impeccable` (fork of `pbakaus/impeccable`, currently identical to upstream `5e745ac`)

## 1. Intent and why

This fork exists to be maintained independently, not to send pull requests upstream. The goal
of this first phase is not to add features, but to make the fork **verifiable, reviewable, and
cheap to rebase**.

The reason is measurable: upstream moves at roughly 17 commits/day (521 commits in the last 30
days). Any divergence that touches a hot file will keep generating conflicts. This phase
therefore deliberately selects changes that **do not alter product behaviour** and **do not
touch files upstream is actively working on**.

## 2. Goals

1. **Documented local build & test.** One page (`docs/FORK.md`) with the prerequisites
   (Bun version from `.bun-version`, Rust stable + the `wasm32-unknown-unknown` target, Chrome
   for the detector tests) and the verification commands that actually work on Windows.
2. **A quality gate for new code.** Fork CI rejects `rustfmt`/`biome`/`clippy` failures
   **on changed lines**, without reformatting the existing 216k lines.
3. **Diffs a human can read.** The 23 generated skill directories (~45 MB of a 112 MB
   checkout) are marked as generated so they stop flooding every review.
4. **A written sync policy.** `main` mirrors upstream; all work happens on `fork/*` branches;
   the rebase procedure is documented.

## 3. Non-goals

- No splitting large files. No crate rewrites. No changes to detector rules, prompts, or
  command text.
- No touching the 23 generated directories beyond adding `.gitattributes` entries.
- No npm publish, no standalone engine release, no rebranding, no new providers in this phase.
- No touching `panic = "abort"` and no sweeping the 2,480 `unwrap()` calls.

## 4. Anti-goals

- **Giant PRs/commits** that cannot be rebased. This includes a repo-wide `cargo fmt`.
- **Undetected behaviour changes**: exit codes, JSON field order, stdout text.
  `docs/CLI-CONTRACT.md` and `tests/oracle/` are the contract; both must stay green.
- **Deleting the generated directories**, because upstream CI verifies them with
  `git diff --exit-code -- .agents .claude .cursor ...`.

## 5. Constraints (facts from the repo)

| Fact | Impact |
|---|---|
| 521 commits / 30 days | Routine rebases are mandatory; divergence must stay thin |
| `page_checks.rs` 68 commits/30 days, `element_checks.rs` 58, `quality.rs` 38, `driver.rs` 36 | Do not split or rewrite them now |
| `live_server.rs` 3 commits/30 days, `serve_question.rs` 15 | Safe candidates later if needed |
| 2,450 Rust `#[test]` cases + oracle goldens | Adequate safety net for small refactors |
| 2,480 `unwrap()` calls in `src`, `panic = "abort"` | `clippy::unwrap_used` is unrealistic; the gate has to be selective |
| 209 entries in `Cargo.lock` | Clippy in the gate can be slow; limit it to changed crates |
| Bun & Cargo are not installed on the working machine | Phase 0 is a prerequisite, not optional |

## 6. Approaches considered

**Quality gate**

- **A. Big-bang**: repo-wide `cargo fmt` + `clippy --workspace -- -D warnings`. Rejected: one
  commit reformatting 216k lines makes every subsequent rebase impossible, and the warning
  baseline has never been measured.
- **B. Based on changed files/crates** (chosen): `rustfmt --check` on changed `.rs` files,
  `cargo clippy -p <crate>` for touched crates, `biome check` on changed JS/TS. Zero churn on
  old code, immediate value, minimal conflict with upstream.
- **C. No gate** (status quo). Rejected: that is the current condition.

**Generated directories**

- **A. Remove them from the repo.** Rejected: violates the upstream CI contract.
- **B. Mark them via `.gitattributes`** (`-diff`, `linguist-generated`). Chosen: still tracked,
  but they no longer show up as text diffs.
- **C. Leave them alone.** Rejected: every `bun run build` means reviewing 45 MB of generated
  files.

**Large files**

- **A. Split now.** Rejected by the churn data in §5.
- **B. Defer** (chosen), with a threshold: split only when a file drops below ~5 commits/30
  days, or when there is a concrete need.

## 7. Decisions taken

1. **Branch model: `main` mirrors, `fork/main` integrates.** `main` only fast-forwards from
   `upstream/main` and never receives direct commits. All work lands on `fork/<topic>`, and its
   PRs target `fork/main` (the integration branch). Anything released later (if ever) is tagged
   from `fork/main`. Consequences: `git diff upstream/main fork/main` always shows exactly the
   fork's total divergence, and `main` can be reset to upstream at any time without losing
   anything.
2. **Change-based gate** (Approach B), running as a separate CI job (`fork-gates`) that does not
   modify any existing upstream job.
3. **Generated directories are marked, not deleted** (Approach B).
4. **No behaviour change** in this phase. Verification: `tests/oracle/run.mjs` and
   `cargo test --workspace` stay green, `git diff --exit-code` over the generated directories
   stays clean.
5. **Large files are deferred** (Approach B).
6. **Dangerous upstream workflows are disabled through repo settings, not by editing files.**
   `sync-generated-output.yml` writes to `main` on every sync and would break the mirror
   invariant; `sheriff.yml` auto-closes PRs after 14 days. Both are disabled from the Actions
   tab (zero diff, zero conflicts) and the reason is recorded in `docs/FORK.md`.
7. **The gate attributes findings per line, not per file.** `rustfmt`/`biome` have no per-line
   mode, so: `clippy --message-format=json` and `biome check --reporter=json` are filtered by
   the line ranges from `git diff -U0`; `rustfmt --check` output is parsed per hunk and sliced
   with the same ranges. A finding that cannot be attributed to a changed line **does not fail
   the gate** (fail-open), because a false failure on old code costs far more than one missed
   finding. Consequence: no baseline file is needed, and the `suspicious`/`style`/`perf` lint
   groups can be enabled from day one even though the old code is not clean.
8. **Upstream `ci.yml` is left untouched.** `fork-gates` is a separate, additive job that is
   made a required check. Trimming `ci.yml` (Node matrix, Windows, live-e2e) is deferred until
   it proves slow; it is also a file upstream touches often.
9. **Versions are pinned only in fork-owned files.** The Rust toolchain is pinned inside
   `.github/workflows/fork-gates.yml`, Biome runs as `bunx @biomejs/biome@<pinned-version>`.
   `rust-toolchain.toml`, `package.json`, and `bun.lock` are not touched so there is no conflict
   every time upstream bumps a dependency.

## 8. Phase plan

**Phase 0 — Toolchain (prerequisite, no product code)**
Install Bun (version from `.bun-version`), Rust stable + `wasm32-unknown-unknown`, and Chrome for
the detector. Prove `bun install`, `bun run build`, `cargo build --workspace --all-targets`, and
`cargo test --workspace` are green. Record the results in `docs/FORK.md`.

**Phase 1 — Fork identity (no conflict risk)**
`docs/FORK.md`: how the fork works, the `main`-mirrors policy, sync/rebase commands, local
prerequisites, and a note that the `//! JS: ...` comments (1,483 of them) point at JS sources in
the upstream repo, not at files in this checkout. Add the `fork-gates` skeleton.

**Phase 2 — CI gate**
Measure the baseline first (`cargo clippy --workspace --all-targets` → count warnings per group,
record it in a fork issue/PR). Implement `scripts/fork/check-changed.mjs`: take the changed line
ranges from `git diff -U0 upstream/main...HEAD`, run `rustfmt --check`, `cargo clippy
--message-format=json`, and `bunx @biomejs/biome@<pin> check --reporter=json`, then filter
findings to those ranges. Add `.github/workflows/fork-gates.yml` (pin Rust here). The gate must
pass on a commit that does not touch code, and fail on a test PR that deliberately breaks a rule
**on a changed line**.

**Phase 3 — `.gitattributes`**
25 paths are already marked `linguist-generated` (23 provider directories + `plugin/**` +
`cursor-plugin/**`), so the actual delta is only adding `-diff`. Verify with
`bun run build:release` (not `build:skills`, which uses `--skip-root-sync` and therefore never
dirties the generated directories), then confirm the result still produces no diff.

**Phase 4 (optional, separate) — Differentiation**
Only after the baseline is green and the first sync has succeeded: pick one capability upstream
does not have. Out of scope for this document.

## 9. Definition of done (phases 1–3)

- `docs/FORK.md` exists and its commands have been run locally, not copied from CI.
- `fork-gates` fails on a test PR that deliberately breaks formatting on a changed line, and
  passes on a PR that does not touch code.
- `main` is still identical to `upstream/main` (`git diff --quiet upstream/main main`).
- Diffs on `fork/*` branches no longer show the contents of generated directories.
- `cargo test --workspace` + `tests/oracle/run.mjs` are green on the working branch.

## 10. Left to the builder

- Script and CI job names (`check-changed.mjs`, `fork-gates`) — may be changed.
- The hunk/span parsing details for each tool, including behaviour when a tool is unavailable
  (the gate must skip, not fail, when `rustfmt`/`biome` is missing on a contributor's machine).
- Whether the gate is also installed as a local pre-commit hook (optional).
- How to map a changed crate path to the `cargo clippy -p` package name.
- The wording of `docs/FORK.md`.

## 11. Calls I made on my own (check these while reading)

| # | Builder question | Decision |
|---|---|---|
| 1 | Integration branch | `fork/main`, PRs target it (§7.1) |
| 2 | Dangerous upstream workflows | Disable via GitHub settings, record in `FORK.md` (§7.6) |
| 3 | Gate granularity | Per-line attribution from spans/hunks, fail-open (§7.7) |
| 4 | Is the full `ci.yml` still required? | Yes, unchanged; `fork-gates` is additive (§7.8) |
| 5 | Tool pins | Fork-owned files only; `bunx` for Biome (§7.9) |
| 6 | Is Clippy `correctness`-only pointless? | It is not used; per-line attribution makes the warn groups safe to enable (§7.7) |
| 7 | Phase 3 scope | `-diff` only; verify with `build:release` (Phase 3) |
