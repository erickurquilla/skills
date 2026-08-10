SKILL: MSW Test

PURPOSE:
Run the MSW (Mikheyev-Smirnov-Wolfenstein) oscillation test for the Emu code 
(2-flavor build) end-to-end, and verify correctness. Use this skill whenever 
the user asks to "run the MSW test" or "check MSW."

WORKING DIRECTORY:
/home/erick/software/Emu  (adjust to wherever the Emu build directory is, 
run commands from the Exec/ or equivalent build directory used for this test)

STEPS TO EXECUTE, IN ORDER:

1. Clean the build:
   make realclean

2. Generate the 2-flavor build files:
   make generate NUM_FLAVORS=2

3. Compile with 2 flavors, capturing compiler output:
   make -j NUM_FLAVORS=2 2>&1 | tee comp_output.txt

4. Generate initial conditions for the MSW test:
   python ../Scripts/initial_conditions/st0_msw_test.py

5. Run the simulation executable with the MSW test inputs:
   ./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_msw_test

6. Run the MSW test validation script:
   python ../Scripts/tests/msw_test.py

EXECUTION RULES:
- Run each step in sequence. If a step fails (non-zero exit code, compiler 
  error, runtime crash, or Python exception), STOP and do not proceed to 
  subsequent steps.
- Do NOT write any summary files, logs, markdown reports, or text files 
  with the results. comp_output.txt from step 3 is the only file allowed 
  to be written, since it's part of the normal command — do not create 
  any additional files beyond what the commands themselves produce.
- All reporting must be done as a normal chat/text response back to me, 
  not saved to disk.

REPORTING (only if the test fails, at any step):
Give me a plain-text report in your response, structured as:
- Which step failed (build / IC generation / run / validation)
- The exact error message or relevant excerpt (compiler error, traceback, 
  or failed assertion/tolerance check from msw_test.py)
- Your best diagnosis of the likely cause, based on the error and any 
  relevant code you inspect
- Whether this looks like a code regression, a stale build artifact issue, 
  an environment/dependency issue, or a test-script issue — and why
- Suggested next step to fix it (but do not attempt fixes automatically 
  unless I ask you to)

If the test PASSES:
Just tell me clearly it passed, and briefly note any relevant output 
(e.g., final error/tolerance value from msw_test.py) so I can sanity-check 
it, without dumping full raw output.

DO NOT:
- Write .md, .txt, .log files summarizing results
- Modify any source files as part of running this test unless I explicitly ask
- Skip steps or run them out of order
- Silently continue past a failed step
