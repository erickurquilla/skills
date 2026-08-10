# Monocromatic Isotropic Scattering Test

Use this skill to run and validate the EMU monochromatic isotropic scattering test from beginning to end.

The skill must:

1. Generate the initial particle distribution.
2. Execute the EMU simulation.
3. Run the plotting and validation script.
4. Determine whether the complete test passed or failed.
5. Investigate any failure and explain its likely root cause.
6. Show relevant progress, findings, warnings, errors, and diagnostic information directly in the Cursor chat whenever this helps the user understand what is happening.

The skill is intended for Cursor. It should communicate naturally through the Cursor chat rather than only producing a generic report at the end.

## Working Directory

Run all commands from the EMU `Exec` directory.

Before executing anything, confirm the current working directory with:

```bash
pwd
```

If the current directory is not the EMU `Exec` directory, locate it or explain clearly in the Cursor chat that the test cannot proceed from the current location.

Do not silently run the commands from an incorrect directory.

## Expected Files

Verify that the following files exist:

```text
../Scripts/initial_conditions/st12_monocromatic_isotropic_scattering_test.py
../Scripts/tests/coll_mono_isot_scat_test.py
../sample_inputs/inputs_monocromatic_isotropic_scattering_test
./main3d.gnu.TPROF.MPI.CUDA.ex
```

If a file is missing:

* Show the missing path in the Cursor chat.
* Search the repository for similarly named files.
* Check whether the problem is a spelling difference, moved file, missing build, or incorrect working directory.
* Explain the most likely cause.
* Do not continue blindly when a required file is missing.

## Cursor Chat Communication

Use the Cursor chat throughout the process whenever useful.

The messages shown in the chat should be specific to the current test and should not be generic status messages.

For example, the skill may show:

```text
Starting the monochromatic isotropic scattering test from the EMU Exec directory.
```

```text
The initial particle-generation script completed successfully and generated the expected particle file.
```

```text
The EMU executable failed during CUDA initialization. I am checking the runtime log and GPU environment.
```

```text
The simulation completed, but the validation script found a numerical difference larger than its allowed tolerance.
```

The skill does not need to print every command or every line of output in the Cursor chat. It should show information that helps the user follow the test, especially:

* The step currently running.
* Whether a step passed or failed.
* Important warnings.
* Relevant error messages.
* Diagnostic findings.
* Generated outputs and plots.
* The final PASS or FAIL result.

Long command output should normally be saved to log files. Show only the most relevant portions in the Cursor chat.

## Step 1: Record the Environment

Before running the test, collect:

* Current working directory.
* Current Git branch.
* Current Git commit hash.
* Whether the repository contains uncommitted changes.
* Python version.
* Executable permissions.
* GPU availability when relevant.

Use commands such as:

```bash
pwd
git branch --show-current
git rev-parse --short HEAD
git status --short
python --version
ls -l ./main3d.gnu.TPROF.MPI.CUDA.ex
```

When the executable uses CUDA, also check:

```bash
nvidia-smi
```

Show a brief environment summary in the Cursor chat. Do not dump the entire `nvidia-smi` output unless it contains information relevant to a failure.

## Step 2: Generate the Initial Particles

Run:

```bash
python ../Scripts/initial_conditions/st12_monocromatic_isotropic_scattering_test.py
```

Capture standard output, standard error, and the exit code.

A safe way to preserve the output is:

```bash
set -o pipefail
python ../Scripts/initial_conditions/st12_monocromatic_isotropic_scattering_test.py \
    2>&1 | tee monocromatic_isotropic_scattering_particles.log
particle_status=${PIPESTATUS[0]}
```

After the command finishes:

* Record the real Python exit code.
* Inspect the output for errors and warnings.
* Determine which files were created or modified.
* Confirm that the expected particle or initial-condition data was generated.
* Check that the generated files are not empty.
* Check that the plotting or simulation stages will look for the same generated filenames.

The particle-generation step passes only when:

* The script exits with code `0`.
* No fatal exception occurs.
* The required particle data is generated.
* The generated files are nonempty and readable.

Show the result in the Cursor chat.

For example:

```text
Particle generation: PASS

The script exited with code 0 and generated the initial-condition data required by the simulation.
```

If it fails, immediately show the important error in the Cursor chat and begin diagnosing it.

## Step 3: Execute the EMU Simulation

Run:

```bash
./main3d.gnu.TPROF.MPI.CUDA.ex \
    ../sample_inputs/inputs_monocromatic_isotropic_scattering_test
```

Capture the complete output in a log file while also displaying it in the terminal:

```bash
set -o pipefail
./main3d.gnu.TPROF.MPI.CUDA.ex \
    ../sample_inputs/inputs_monocromatic_isotropic_scattering_test \
    2>&1 | tee monocromatic_isotropic_scattering_run.log
run_status=${PIPESTATUS[0]}
```

Use `${PIPESTATUS[0]}` to obtain the executable's exit code. Do not use only `$?`, because that may report the exit code from `tee` instead of the EMU executable.

After execution:

* Record the executable exit code.
* Inspect the log for fatal errors and warnings.
* Confirm that expected simulation outputs or plotfiles were created.
* Confirm that output files are nonempty.
* Identify the final timestep or simulation time reached.
* Determine whether the intended stopping condition was reached.
* Check for stale output from previous runs.

Search the log for indicators such as:

```bash
grep -inE \
"error|fatal|abort|assert|nan|inf|cuda|mpi|segmentation|illegal|failed" \
monocromatic_isotropic_scattering_run.log
```

Do not treat every occurrence of words such as `inf` or `error` as a real failure without inspecting its context.

The simulation step passes only when:

* The executable exits with code `0`.
* No fatal error appears in the log.
* The expected simulation output is generated.
* The intended final timestep or stopping condition is reached.
* No unexpected NaNs or infinities are detected.

Show a concise result in the Cursor chat, including important output names and the final simulation time or timestep when available.

## Step 4: Plot and Validate the Results

Run:

```bash
python ../Scripts/tests/coll_mono_isot_scat_test.py
```

Capture the output and exit code:

```bash
set -o pipefail
python ../Scripts/tests/coll_mono_isot_scat_test.py \
    2>&1 | tee monocromatic_isotropic_scattering_plot.log
plot_status=${PIPESTATUS[0]}
```

After the command finishes:

* Record the exit code.
* Confirm that the script found and read the simulation data.
* Identify generated plots.
* Confirm that plot files are nonempty.
* Report their names and locations.
* Inspect numerical comparisons printed by the script.
* Record expected values, actual values, differences, and tolerances when available.
* Determine whether the script explicitly reports PASS or FAIL.

The plotting and validation step passes only when:

* The script exits with code `0`.
* The required simulation data is found.
* Expected plots or validation outputs are generated.
* Numerical checks satisfy their required tolerances.
* No relevant traceback or fatal warning occurs.

Show the validation result directly in the Cursor chat.

## Failure Investigation

If any step fails, do not stop after repeating the final error message.

Investigate the underlying cause and explain the findings in the Cursor chat.

The investigation should be proportional to the failure. Run relevant diagnostics, but avoid unrelated commands.

### Python Failures

For Python-script failures, check:

* Missing Python modules.
* Incorrect Python environment.
* Missing input files.
* Incorrect relative paths.
* Permission errors.
* Syntax errors.
* Array shape mismatches.
* File-format mismatches.
* Empty or corrupt data.
* Plotting backend problems.
* Expected simulation output not being generated.
* The script reading stale data from a previous run.

Read the complete traceback.

Identify:

1. The first meaningful exception.
2. The script and line where it originates.
3. Whether the error comes from EMU code or an external library.
4. The likely correction.

Do not report only the last line of the traceback.

### Executable Failures

For executable failures, check:

* Missing executable.
* Missing execute permissions.
* Missing shared libraries.
* CUDA runtime or driver incompatibility.
* GPU visibility.
* MPI initialization errors.
* Invalid input parameters.
* Missing particle files.
* Incorrect file paths.
* AMReX assertions.
* Memory allocation failures.
* Segmentation faults.
* Illegal CUDA memory accesses.
* NaNs or infinities.
* Output-directory conflicts.
* Failure to reach the intended final timestep.

Relevant commands may include:

```bash
file ./main3d.gnu.TPROF.MPI.CUDA.ex
ldd ./main3d.gnu.TPROF.MPI.CUDA.ex
nvidia-smi
```

Use only commands relevant to the observed error.

### Missing Output

If the simulation or plotting script cannot find required data:

* Determine the exact filename expected by the script.
* Compare it with the files actually generated.
* Inspect the simulation input file for output prefixes and directories.
* Inspect the Python scripts for hard-coded paths or filenames.
* Check file timestamps.
* Determine whether the simulation stopped before writing output.
* Determine whether an old output directory caused the run to stop or reuse stale results.

Explain the mismatch clearly in the Cursor chat.

### Numerical Validation Failure

If all commands exit successfully but a numerical test fails:

* Do not report the complete test as successful.
* Report the expected value.
* Report the computed value.
* Report the absolute difference.
* Report the relative difference when meaningful.
* Report the allowed tolerance.
* Identify which physical or numerical quantity failed.
* Check for normalization errors.
* Check whether the initial distribution is isotropic.
* Check whether the scattering parameters correspond to the intended test.
* Check for conservation violations.
* Check for NaNs or infinities.
* Distinguish a plotting failure from a physics or numerical failure.

A zero exit code alone is not enough to declare PASS.

## Source Inspection

When diagnosing a problem, inspect relevant source files or input parameters when necessary.

Possible files include:

```text
../Scripts/initial_conditions/st12_monocromatic_isotropic_scattering_test.py
../Scripts/tests/coll_mono_isot_scat_test.py
../sample_inputs/inputs_monocromatic_isotropic_scattering_test
```

When explaining a source-level issue:

* Mention the relevant file.
* Mention the function, variable, or input parameter involved.
* Include the relevant line number when available.
* Explain why it produces the observed behavior.
* Propose a concrete correction.

Do not modify files automatically unless the user explicitly requests a fix.

## Final Cursor Chat Report

At the end, provide a clear report directly in the Cursor chat.

Use the following structure, adapting it to the actual results:

```text
Monocromatic Isotropic Scattering Test
======================================

Overall status: PASS or FAIL

Environment
-----------
Working directory:
Git branch:
Git commit:
Uncommitted changes:
Python version:
Executable:
GPU:

Particle generation
-------------------
Command:
Exit code:
Status:
Log:
Generated files:
Important findings:

EMU simulation
--------------
Command:
Exit code:
Status:
Log:
Generated outputs:
Final timestep or simulation time:
Important warnings or errors:

Plotting and validation
-----------------------
Command:
Exit code:
Status:
Log:
Generated plots:
Numerical comparisons:
Important warnings or errors:

Diagnosis
---------
Root cause:
Evidence:
Affected file or command:
Suggested solution:

Final conclusion
----------------
Clearly state whether the entire monochromatic isotropic scattering test passed.
```

Do not include empty diagnosis sections when the test passes and there is nothing meaningful to diagnose.

## Success Criteria

Return `PASS` only when all of the following are true:

1. The initial-condition script exits successfully.
2. The expected particle data is generated.
3. The EMU executable exits successfully.
4. The simulation reaches its intended stopping condition.
5. The expected simulation output is generated.
6. The plotting and validation script exits successfully.
7. Expected plots or numerical validation outputs are generated.
8. All numerical checks satisfy their tolerances.
9. No fatal errors, unexpected NaNs, or unexpected infinities are detected.

Otherwise, return `FAIL`.

When returning `FAIL`, explain in the Cursor chat:

* What failed.
* Why it likely failed.
* What evidence supports the diagnosis.
* Which file, parameter, or command is involved.
* What concrete action should be taken to fix or further investigate the problem.

