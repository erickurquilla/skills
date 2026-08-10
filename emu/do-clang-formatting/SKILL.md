---
name: do-clang-formatting
description: >-
  Apply clang-format 18.1.8 to the Emu C++ codebase using the team's Sherwood/Debraj
  workflow: verify the formatter version, check merge state, format Source and
  unit_test with -style=file, and report diffs without committing. Use when the
  user asks to clang-format Emu, reformat the branch, run do-clang-formatting,
  or apply the team's clang-format workflow before merge/PR.
---

# Do Clang-Formatting (Emu)

Use this skill to apply `clang-format` consistently on the Emu branch, following
the team convention from Sherwood and Debraj.

## Team convention

When merging with `development`:

1. Keep **this branch's actual code changes**, not development's
   reformatted/whitespace-only diffs.
2. Reformat everything with clang-format **after** the merge content is correct.

Required formatter version: **exactly 18.1.8**.

Do **not** hand-format files. Only use the exact `find` + `clang-format`
invocation below so results match the rest of the team.

Do **not** commit automatically. Leave formatting changes unstaged/unstaged for
the user to review and commit.

Do **not** create summary/log/markdown report files. Report everything in the
Cursor chat.

## Working directory

Run commands from the Emu repository root:

```bash
/home/erick/software/Emu
```

Confirm with `pwd` before formatting. The repo must contain `.clang-format`
(required by `-style=file`).

## Step 1 — Verify environment

Check that `clang-format` is installed and is **exactly 18.1.8**:

```bash
which clang-format
clang-format --version
ls -la .clang-format
```

### If missing or wrong version (Linux)

Prefer the project Python virtualenv (e.g. `cass_ve1`) rather than replacing
the system Ubuntu package:

```bash
source /home/erick/software/python_virtual_enviroments/cass_ve1/bin/activate
pip install clang-format==18.1.8
hash -r
export PATH="$VIRTUAL_ENV/bin:$PATH"
which clang-format
clang-format --version   # must print 18.1.8
```

On Linux, `brew uninstall clang-format` is **not** needed. Ensure the venv
binary is ahead of `/usr/bin/clang-format` on `PATH`.

If the version is still wrong after install attempts, **stop** and tell the
user what is installed and what to fix. Do not format with a non-18.1.8 binary.

### Style file

Briefly confirm `.clang-format` exists at the repo root (Google-based C++,
`ColumnLimit: 80`, `IndentWidth: 4`, `PointerAlignment: Left`,
`BreakBeforeBraces: Attach`, `SortIncludes: true`, `UseTab: Never`).

## Step 2 — Check merge state first

Before formatting anything:

```bash
git status
git branch --show-current
test -f .git/MERGE_HEAD && echo 'MID-MERGE: yes' || echo 'MID-MERGE: no'
test -d .git/rebase-merge -o -d .git/rebase-apply && echo 'REBASE: yes' || echo 'REBASE: no'
git rev-list --count development..HEAD
git rev-list --count HEAD..development
git diff --stat development...HEAD | tail -40
```

### Stop conditions

* If the branch has **unresolved merge conflicts** or is **mid-merge** /
  **mid-rebase**: **STOP**. Report the state. Do not resolve conflicts
  automatically and do not format.
* If a merge with `development` was just completed: confirm with the user that
  **their real code changes** (not development's formatting-only diffs) are the
  ones present before reformatting. Reformatting on top of a bad merge makes
  real changes hard to distinguish from formatting noise.

Only proceed when the working tree is not mid-merge/rebase (or the user has
explicitly confirmed post-merge content is correct).

## Step 3 — Apply clang-format

From the Emu repo root, with clang-format **18.1.8** on `PATH`, run exactly:

```bash
find Source unit_test \( -name '*.cpp' -o -name '*.h' -o -name '*.H' \) | xargs clang-format -i -style=file
```

Notes:

* Use this Linux-compatible `find` form (`\( \)` with `-o`). Debraj found that
  some escaped-parenthesis variants fail on macOS's default regex; prefer this
  portable `-o` form on Linux as well for team consistency.
* Format only `Source` and `unit_test` with extensions `.cpp`, `.h`, `.H`.
* Do not hand-edit formatting.

## Step 4 — Report (no commit)

After formatting:

```bash
git diff --stat
git diff --numstat
git status --short
```

In the Cursor chat:

1. Show `git diff --stat` so the user can see which files changed and how much.
2. Flag any file whose formatting diff is unusually large (e.g. tens–hundreds of
   lines of churn). That may mean the file was never formatted before or has a
   style-config issue; recommend a quick review before commit.
3. Optionally sample a small portion of a large diff to confirm it is wrapping /
   whitespace only, not logic changes.
4. **Do not commit.** Leave changes for the user to review and commit.

## Communication

Show progress in chat when useful, for example:

```text
clang-format is 18.1.8; .clang-format is present. Checking merge state next.
```

```text
Not mid-merge; branch is N commits ahead of development. Applying clang-format.
```

```text
Formatting done. Evolve.cpp has a large formatting-only churn — worth a skim before commit. Nothing was committed.
```

## Checklist

```text
- [ ] clang-format --version is exactly 18.1.8
- [ ] .clang-format exists at repo root
- [ ] Not mid-merge / mid-rebase (or user confirmed post-merge content)
- [ ] Ran the exact find | xargs clang-format -i -style=file command
- [ ] Reported git diff --stat and flagged large diffs
- [ ] Did not commit
```
