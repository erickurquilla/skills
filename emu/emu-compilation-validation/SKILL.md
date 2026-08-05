---
name: emu-compilation-validation
description: >-
  Validate that the EMU codebase compiles for NUM_FLAVORS=2 and NUM_FLAVORS=3
  with clean independent builds, analyze warnings/errors, and produce a structured
  compilation report. Use when the user asks to validate EMU compilation, test
  NUM_FLAVORS builds, check flavor-dependent compile failures, or run the EMU
  compilation validation skill.
---

# EMU Compilation Validation Skill

## Objective

Validate that the EMU codebase compiles successfully for both supported flavor configurations:

* `NUM_FLAVORS=2`
* `NUM_FLAVORS=3`

The EMU executable source directory is:

```bash
/home/erick/software/Emu/Exec
```

For each configuration, perform a clean and independent compilation, analyze all warnings and errors, and produce a detailed report.

## Instructions

### 1. Enter the EMU executable directory

```bash
cd /home/erick/software/Emu/Exec
```

Before running any compilation commands, confirm:

* The directory exists.
* A valid `Makefile` is present.
* The current Git branch and commit hash.
* Whether the working tree contains modified or untracked files.

Do not modify, delete, reset, or overwrite existing source files.

### 2. Compile with `NUM_FLAVORS=2`

Run the following commands sequentially:

```bash
make realclean
make generate NUM_FLAVORS=2
make -j NUM_FLAVORS=2
```

Capture the complete standard output and standard error from each command.

For every command, record:

* The exact command executed.
* The exit status.
* Whether it succeeded or failed.
* Relevant warnings.
* Relevant error messages.
* The source file and line number associated with each issue, when available.

If `make realclean` appears to be an invalid target, verify whether the intended target is `realclean`, `clean`, or another cleanup target defined in the `Makefile`. Do not silently replace the command; document the discrepancy first.

### 3. Analyze the `NUM_FLAVORS=2` result

Determine whether the failure, warning, or unexpected behavior is caused by:

* C or C++ compilation errors.
* Preprocessor macros.
* Flavor-dependent enum or array definitions.
* Missing generated files.
* Incorrect include paths.
* Missing libraries or environment modules.
* Linker errors.
* Stale object files.
* Parallel-build race conditions.
* Inconsistent compiler flags.
* Problems in the `make generate` step.
* Unsupported assumptions about the number of flavors.

For each issue:

1. Explain the likely root cause.
2. Identify the relevant file, function, macro, or build rule.
3. Propose a concrete solution.
4. Include a minimal code or `Makefile` change when appropriate.
5. Do not apply the change unless explicitly instructed.

### 4. Compile with `NUM_FLAVORS=3`

After completing the two-flavor analysis, begin from a clean build state and run:

```bash
make realclean
make generate NUM_FLAVORS=3
make -j NUM_FLAVORS=3
```

Capture and analyze the output using the same procedure applied to `NUM_FLAVORS=2`.

Do not reuse object files or generated files from the two-flavor build unless the build system explicitly guarantees that they are configuration-independent.

### 5. Compare both configurations

Compare the results for `NUM_FLAVORS=2` and `NUM_FLAVORS=3`.

Pay particular attention to:

* Preprocessor branches using `NUM_FLAVORS`.
* Flavor-index enums.
* Array dimensions.
* Generated headers or source files.
* Neutrino and antineutrino component counts.
* Upper-triangular flavor combinations.
* Initialization code.
* Serialization or output routines.
* Loops with hard-coded flavor bounds.
* Build dependencies that do not account for changes in `NUM_FLAVORS`.

Determine whether:

* Both configurations compile successfully.
* Only one configuration fails.
* The same issue appears in both configurations.
* Generated files from one configuration contaminate the other.
* A full clean build is required whenever `NUM_FLAVORS` changes.

## Required Report

Write the final report using the following structure.

### 1. Environment

Include:

* Repository path.
* Current Git branch.
* Git commit hash.
* Working-tree status.
* Compiler name and version.
* `make` version.
* Relevant environment variables.
* Available CPU core count.
* Date and time of the test.

### 2. `NUM_FLAVORS=2` Compilation

Include a table with:

| Step     | Command                       | Exit Status | Result    |
| -------- | ----------------------------- | ----------: | --------- |
| Clean    | `make realclean`              |       value | Pass/Fail |
| Generate | `make generate NUM_FLAVORS=2` |       value | Pass/Fail |
| Compile  | `make -j NUM_FLAVORS=2`       |       value | Pass/Fail |

Then summarize:

* Errors.
* Warnings.
* Suspected root causes.
* Proposed solutions.

### 3. `NUM_FLAVORS=3` Compilation

Include the same table and analysis for:

```bash
NUM_FLAVORS=3
```

### 4. Comparison

Explain the important differences between the two builds, including any flavor-dependent failures.

### 5. Recommended Fixes

For every recommended change, provide:

* File path.
* Relevant line or build rule.
* Explanation.
* Proposed patch or code snippet.
* Expected effect.
* Possible risks or side effects.

### 6. Final Status

End with one of the following conclusions:

* Both configurations compile successfully.
* `NUM_FLAVORS=2` compiles, but `NUM_FLAVORS=3` fails.
* `NUM_FLAVORS=3` compiles, but `NUM_FLAVORS=2` fails.
* Both configurations fail.
* The build could not be completed because of an environmental or dependency issue.

Also clearly state whether the repository is currently safe to use with both supported values of `NUM_FLAVORS`.

## Safety Constraints

* Do not modify source files automatically.
* Do not run `git reset`, `git clean`, or commands that remove untracked files.
* Do not install packages without permission.
* Do not suppress warnings or errors.
* Do not report success based only on partial compilation.
* A build is successful only when the final `make` command exits with status `0`.
* Preserve complete build logs for both configurations.
