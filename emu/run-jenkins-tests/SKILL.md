# TASK: Run Emu CI Test Suite Locally (Single-Device Adaptation)

## CONTEXT
This reproduces the Jenkins CI pipeline for the Emu repo (`/home/erick/software/Emu`) on a
local server with a single GPU/device. The original Jenkinsfile runs every simulation with
`mpirun -np 4` inside a Docker container that mounts EOS and NuLib tables at `/EOS` and
`/NuLib`. Locally, we have neither multiple ranks nor those container mounts, so this run
needs two adaptations:

1. **No multi-rank MPI.** Do not add `mpirun -np 4` (or any `-np N > 1`) to any command.
   Run the compiled executable directly (e.g. `./main3d.gnu.TPROF.MPI.CUDA.ex <inputs_file>`).
   If the code hard-requires being launched under `mpirun`, use `mpirun -np 1`, but only as
   a fallback if direct execution fails — confirm which is needed during Discovery.

2. **Local EOS/NuLib tables, no input-file edits.** The tables now live at:
   - `/home/erick/tables/LS220.h5`
   - `/home/erham/tables/NuLib_SFHo.h5`   <!-- fix typo below -->
   - `/home/erick/tables/NuLib_SFHo.h5`

   Do NOT edit the text of any `sample_inputs/inputs_*` file to point at these paths.
   Instead, during Discovery, find out *how* the code currently resolves table paths
   (grep the source for how the `/EOS` and `/NuLib` mount points from the Dockerfile/Jenkins
   env are consumed — likely an environment variable, a hardcoded relative path, or a
   parameter read at runtime) and reproduce that mechanism without touching input text:
   - If it's an env var (e.g. something like `EOS_TABLE_PATH`, `NULIB_TABLE_PATH`), export it
     in the shell session before each run.
   - If it's a fixed relative/absolute path (e.g. `/EOS/...`, `/NuLib/...`), create symlinks
     at that exact path pointing to the real files in `/home/erick/tables/`, rather than
     editing any config or input file.
   Report which mechanism you found and which approach you used.

## GUARDRAILS (STRICT)
- Do not create any files beyond what's strictly necessary (e.g. symlinks, if that's the
  chosen table mechanism). No scratch scripts, no new test files, no README/report files
  written to disk.
- Do not `git add`, `git commit`, or `git push` anything, ever.
- Do not modify any file under `sample_inputs/`, `Scripts/`, or any input/test source file.
- Do not modify the Jenkinsfile.
- All output/reporting goes into the Cursor chat as plain text — not a file.
- If a stage fails, STOP immediately. Do not proceed to the next stage. Analyze and report
  the failure first (see REPORTING below).

## STAGES TO RUN (in order, from Exec/ unless noted)

0. **Prerequisites**
   - `mpicc -v`, `nvidia-smi`, `nvcc -V` (informational; note but don't fail the run if
     nvidia tools are unavailable — flag it and ask whether to continue CPU-only)
   - `git submodule update --init`
   - `cp makefiles/GNUmakefile_jenkins_HDF5_CUDA Exec/GNUmakefile`
   - In `Exec/`: `make generate; make -j`

1. **MSW**: `st0_msw_test.py` → run `inputs_msw_test` (no mpirun -np 4) → `msw_test.py` → `rm -rf plt*`
2. **MSW Reflecting**: same init script → run `inputs_msw_test_reflecting` → `msw_test.py` → cleanup
3. **MSW Relativistic**: same init script → run `inputs_relativistic_msw_test` → `relativistic_msw_test.py` → cleanup
4. **Bipolar**: `st1_bipolar_test.py` → run `inputs_bipolar_test` → cleanup
5. **Fast Flavor**: `st2_2beam_fast_flavor.py` → run `inputs_fast_flavor` → `fast_flavor_test.py` → cleanup
6. **Fast Flavor k**: `st3_2beam_fast_flavor_nonzerok.py` → run `inputs_fast_flavor_nonzerok` → `fast_flavor_k_test.py` → cleanup
7. **Fiducial 2F GPU Binary**: `st4_linear_moment_ffi.py` → run `inputs_1d_fiducial` →
   `reduce_data_fft.py`, `reduce_data.py`, `combine_files.py plt _reduced_data.h5`,
   `combine_files.py plt _reduced_data_fft_power.h5`, `avgfee.py`, `power_spectrum.py`,
   `convertToHDF5.py`, `gnuplot avgfee_gnuplot.plt` → note any produced `*.pdf` in the report
   (don't archive, just point to the path) → cleanup
8. **Collisional flavor instability**: `make realclean; make generate NUM_FLAVORS=2 SET_EQUILIBRIUM=1; make -j NUM_FLAVORS=2 SET_EQUILIBRIUM=1` →
   `st8_coll_inst_test.py` → run `inputs_collisional_instability_test` → `reduce_data.py` →
   `coll_inst_test.py` → note `*.pdf` → cleanup
9. **Outflow Vphase Conservation**: `make realclean; make generate NUM_FLAVORS=2; make -j NUM_FLAVORS=2` →
   `st11_periodic_empty_bc.py` → run `inputs_outflow_bh_test` → `write_particles_all_domain.py` →
   `bc_empty_init_test.py` → note `*.pdf` → cleanup
10. **Fiducial 3F CPU HDF5**: `cp ../makefiles/GNUmakefile_jenkins_HDF5 GNUmakefile` →
    `make realclean; make generate; make -j` → `st4_linear_moment_ffi_3F.py` → run
    `inputs_1d_fiducial` (CPU/MPI executable variant, no GPU suffix) → cleanup
11. **Collisions to equilibrium**: `cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile` →
    `make realclean; make generate NUM_FLAVORS=3; make -j NUM_FLAVORS=3` →
    `st7_empty_particles.py` → run `inputs_coll_equi_test` → `coll_equi_test.py` → cleanup
12. **Fermi-Dirac test**: `st9_empty_particles_multi_energy.py`,
    `nsm_constant_background_rho_Ye_T_writer.py` → run `inputs_fermi_dirac_test` →
    `write_particles_all_domain.py` → `fermi_dirac_test.py` → cleanup (`plt*`, `*pdf`, `rho_Ye_T.hdf5`)
13. **Fermi-Dirac Reflecting**: same init scripts → run `inputs_fermi_dirac_test_reflecting` →
    `write_particles_all_domain.py` → `fermi_dirac_test.py` → cleanup
14. **Monochromatic Isotropic-Kernel Scattering**: `st12_monocromatic_isotropic_scattering_test.py` →
    run `inputs_monocromatic_isotropic_scattering_test` → `coll_mono_isot_scat_test.py` → note `*.pdf` → cleanup
15. **Metric test**: in `unit_test/`: `make` → `./main3d.gnu.DEBUG.MPI.ex`

## EXECUTION RULES
- Every simulation run command in the original Jenkinsfile used `mpirun -np 4 ./main3d... inputs_file`.
  Replace each with a direct call: `./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_file`
  (or the CPU/HDF5/DEBUG variant named in that stage). Do not reintroduce multi-rank mpirun.
- Rebuild steps (`make realclean; make generate ...; make -j ...`) must be run exactly as
  specified per stage — flavor counts and flags differ between stages and matter for correctness.
- Clean up `plt*`/`*pdf`/other generated artifacts after each stage as in the original,
  same as before, so stages don't interfere with each other.

## REPORTING (in Cursor chat, plain text — not a file)
For each stage, report:
- Stage name, pass/fail
- Key output (e.g. test script's pass/fail message, not full raw logs)

**On first failure:**
1. Stop the run — do not execute remaining stages.
2. Show the actual error (compile error, runtime crash, or test-script assertion failure).
3. Do root-cause analysis: is it a build issue, a missing/misconfigured table path, a
   physics/tolerance mismatch in the test script, a leftover artifact from a prior stage, or
   an environment issue (missing GPU, missing dependency)?
4. Propose a concrete fix, but do not apply any fix automatically — wait for confirmation.

At the very end (only if all stages pass), give a one-paragraph summary: total stages run,
table-path mechanism used, any warnings/deviations from the original Jenkins pipeline.
