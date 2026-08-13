---
name: run-jenkins-tests
description: >-
  Run the Emu Jenkins CI test suite locally on a single GPU (no Docker, no
  mpirun -np 4), with local EOS/NuLib tables and CUDA arch overrides. Use when
  the user asks to run Jenkins tests, reproduce CI locally, run CI tests, or
  execute the run-jenkins-tests skill.
---

# TASK: Run Emu CI Test Suite Locally (Single-Device Adaptation)

## CONTEXT

Reproduce `Emu/Jenkinsfile` for `/home/erick/software/Emu` on this host.

**Jenkinsfile is authoritative** for stage order and script paths. This skill
adds local adaptations only. Always use full paths from Jenkinsfile
(`Scripts/data_reduction/...` vs `Scripts/initial_conditions/...`).

**Verified:** full Stages 0–15 PASS on this host (2026-08-13) with the rules below.
Typical wall time ~15–20 minutes when batched.

## ENVIRONMENT (every session)

```bash
cd /home/erick/software/Emu/Exec
source /home/erick/software/python_virtual_enviroments/cass_ve1/bin/activate
export OMPI_MCA_btl_vader_single_copy_mechanism=none
```

| Fact | Action |
|------|--------|
| GPU is Quadro GV100 (`sm_70`) | Pass `AMREX_CUDA_ARCH=70` on **every** CUDA `make -j`. Jenkins makefile defaults to `60` — that links but dies at runtime: CUDA "named symbol not found". |
| No Docker `/EOS` `/NuLib` | ParmParse CLI overrides for Fermi only (below). Never edit `sample_inputs/`. Do not create root `/EOS` `/NuLib` unless user approves sudo. |
| No multi-rank CI | Run `./main3d....ex ../sample_inputs/<inputs>` directly. Never `mpirun -np 4`. Fallback only: `mpirun -np 1`. |
| `Exec/GNUmakefile` drifts | Stage 0 always: `cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile` then rebuild. Do not use `GNUmakefile_cassowary` for this skill. |

## PREFLIGHT / FLAVOR MATRIX

Working trees get rebuilt between chats. **Do not assume** a prior stage’s exe remains.

```bash
grep -E '^NUM_FLAVORS|^USE_CUDA|^SET_EQUILIBRIUM|^AMREX_CUDA_ARCH' GNUmakefile | head -10
ls -l main3d.gnu.TPROF.MPI.CUDA.ex main3d.gnu.TPROF.MPI.ex 2>/dev/null
```

| Stages | Build needed |
|--------|----------------|
| 0–7 | CUDA 2F, `SET_EQUILIBRIUM=0`, `AMREX_CUDA_ARCH=70` |
| 8 | CUDA 2F, `SET_EQUILIBRIUM=1`, arch 70 |
| 9 | CUDA 2F, `SET_EQUILIBRIUM=0`, arch 70 (rebuild after 8) |
| 10 | CPU 3F HDF5 → `main3d.gnu.TPROF.MPI.ex` |
| 11–14 | CUDA 3F HDF5, arch 70 (**Stage 14 stays on 3F**) |
| 15 | `unit_test/` DEBUG MPI exe (separate makefile) |

If mismatch: `make realclean` + rebuild from cheat sheet **before** ICs/sim.

## TABLE PATHS (Stages 12–13 only)

```bash
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_fermi_dirac_test \
  nuceos_table_name=/home/erick/tables/LS220.h5 \
  nulib_table_name=/home/erick/tables/NuLib_SFHo.h5
```

Same overrides for `inputs_fermi_dirac_test_reflecting`.

Require in logs:

- `(ReadEosTable.cpp) Using table: /home/erick/tables/LS220.h5`
- `(ReadNuLibTable.cpp) Using table: /home/erick/tables/NuLib_SFHo.h5`

## GUARDRAILS (STRICT)

- No scratch scripts, report files, or README writes.
- No `git add` / `commit` / `push`.
- Do not modify `sample_inputs/`, `Scripts/`, `Jenkinsfile`, or Source for this skill.
- Report only in chat.
- On first failure: **STOP**, diagnose, propose fix, wait for confirmation.

## BUILD / EXE CHEAT SHEET

Always `make realclean` when switching makefile, `NUM_FLAVORS`, or `SET_EQUILIBRIUM`.

| Config | `Exec/GNUmakefile` | Build | Executable |
|--------|--------------------|-------|------------|
| CUDA 2F | `cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile` | `make generate NUM_FLAVORS=2; make -j NUM_FLAVORS=2 AMREX_CUDA_ARCH=70` | `main3d.gnu.TPROF.MPI.CUDA.ex` |
| CUDA 2F + equil | same | `make realclean; make generate NUM_FLAVORS=2 SET_EQUILIBRIUM=1; make -j NUM_FLAVORS=2 SET_EQUILIBRIUM=1 AMREX_CUDA_ARCH=70` | same |
| CUDA 2F reset | same | `make realclean; make generate NUM_FLAVORS=2; make -j NUM_FLAVORS=2 AMREX_CUDA_ARCH=70` | same |
| CPU 3F HDF5 | `cp ../makefiles/GNUmakefile_jenkins_HDF5 GNUmakefile` | `make realclean; make generate; make -j` | `main3d.gnu.TPROF.MPI.ex` |
| CUDA 3F HDF5 | `cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile` | `make realclean; make generate NUM_FLAVORS=3; make -j NUM_FLAVORS=3 AMREX_CUDA_ARCH=70` | `main3d.gnu.TPROF.MPI.CUDA.ex` |

## CRITICAL GOTCHAS

1. **`write_particles_all_domain.py` → `../Scripts/data_reduction/` only** (not `initial_conditions/`). Wrong path → no `plt*.h5` → `fermi_dirac_test.py` numpy crash on empty glob.
2. **Plot I/O is binary even with HDF5 makefile** (`IO.cpp` `#undef AMREX_USE_HDF5`). Sims write `pltNNNNN/` dirs. Stages **9, 12, 13** must run `write_particles_all_domain.py` after sim, before test (`yt` required in venv).
3. **Fermi order:** IC → rhoYeT writer → sim (+ table CLI) → `write_particles` → `fermi_dirac_test.py` → cleanup. Expect `completed ---> plt00000/06000` and `plt*.h5`.
4. **Stage 14 needs CUDA 3F** left from Stage 11. Clear `rm -rf plt*` first (stale `plt*.old.*` false-fails the test).
5. **Cleanup is last** in each stage. Never delete `plt*` before that stage’s tests finish.
6. **Script roots from `Exec/`:** `initial_conditions/` · `tests/` · `data_reduction/` · `babysitting/` under `../Scripts/`.
7. **Batching tip:** shell batches `0–6`, `7–9`, `10–15` with `set -o pipefail` and stop-on-first-fail work well; keep `/tmp/ci_s*_*.log` for diagnosis only (not reports).

## VERIFIED PASS BASELINES (this host)

Use as sanity checks; stop if far off or exit ≠ 0.

| Stage | Pass signal (approx) |
|-------|----------------------|
| 1–2 MSW | `f_xx error` ~`7.5e-7`; conservation ~`1e-15` |
| 3 MSW rel | `f_xx error` ~`3e-12` |
| 4 Bipolar | `Done.` (no test script) |
| 5 Fast flavor | growth/theory ~`1.00013` |
| 6 Fast flavor k | growth/theory ~`0.993` |
| 7 Fiducial 2F | `Done.` + PDFs e.g. `avgfee.pdf` |
| 8 Coll inst | ImΩ relative err ~`0.031` vs Johns LSA |
| 9 Outflow | boundary relative errs ~`1e-11`–`1e-9`; PDFs `electron_occupation*.pdf` |
| 10 Fiducial 3F CPU | `Done.` |
| 11 Coll equi | averages ~`9.93e32` (theory `1e33`) |
| 12–13 Fermi | `max_error_fee`/`feebar` ~`5e-5`; `fuu`/`ftt` ~`0.01` |
| 14 Mono-iso | `Isotropic scattering test passed` + conservation passed; max dens err ~`6e-5` |
| 15 Metric | prints final cartesian / cylindrical / spherical values (no FAIL) |

## STAGES (in order)

### 0. Prerequisites
```bash
cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile
make realclean
make generate NUM_FLAVORS=2
make -j NUM_FLAVORS=2 AMREX_CUDA_ARCH=70
```

### 1. MSW
```bash
python ../Scripts/initial_conditions/st0_msw_test.py
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_msw_test
python ../Scripts/tests/msw_test.py
rm -rf plt*
```

### 2. MSW Reflecting
Same IC → `inputs_msw_test_reflecting` → `msw_test.py` → `rm -rf plt*`

### 3. MSW Relativistic
Same IC → `inputs_relativistic_msw_test` → `relativistic_msw_test.py` → `rm -rf plt*`

### 4. Bipolar
`st1_bipolar_test.py` → `inputs_bipolar_test` → `rm -rf plt*`

### 5. Fast Flavor
`st2_2beam_fast_flavor.py` → `inputs_fast_flavor` → `fast_flavor_test.py` → `rm -rf plt*`

### 6. Fast Flavor k
`st3_2beam_fast_flavor_nonzerok.py` → `inputs_fast_flavor_nonzerok` → `fast_flavor_k_test.py` → `rm -rf plt*`

### 7. Fiducial 2F GPU Binary
```bash
python ../Scripts/initial_conditions/st4_linear_moment_ffi.py
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_1d_fiducial
python ../Scripts/data_reduction/reduce_data_fft.py
python ../Scripts/data_reduction/reduce_data.py
python ../Scripts/data_reduction/combine_files.py plt _reduced_data.h5
python ../Scripts/data_reduction/combine_files.py plt _reduced_data_fft_power.h5
python ../Scripts/babysitting/avgfee.py
python ../Scripts/babysitting/power_spectrum.py
python ../Scripts/data_reduction/convertToHDF5.py
gnuplot ../Scripts/babysitting/avgfee_gnuplot.plt
# note *.pdf in chat; do not archive
rm -rf plt*
```

### 8. Collisional flavor instability
Rebuild CUDA 2F + `SET_EQUILIBRIUM=1` →  
`st8_coll_inst_test.py` → `inputs_collisional_instability_test` →  
`python ../Scripts/data_reduction/reduce_data.py` → `coll_inst_test.py` → note `*.pdf` → `rm -rf plt* *pdf`

### 9. Outflow Vphase Conservation
Rebuild CUDA 2F (no equil) →  
`st11_periodic_empty_bc.py` → `inputs_outflow_bh_test` →  
`python ../Scripts/data_reduction/write_particles_all_domain.py` →  
`bc_empty_init_test.py` → note `*.pdf` → `rm -rf plt* *pdf`

### 10. Fiducial 3F CPU HDF5
```bash
cp ../makefiles/GNUmakefile_jenkins_HDF5 GNUmakefile
make realclean; make generate; make -j
python ../Scripts/initial_conditions/st4_linear_moment_ffi_3F.py
./main3d.gnu.TPROF.MPI.ex ../sample_inputs/inputs_1d_fiducial
rm -rf plt*
```

### 11. Collisions to equilibrium
```bash
cp ../makefiles/GNUmakefile_jenkins_HDF5_CUDA GNUmakefile
make realclean; make generate NUM_FLAVORS=3; make -j NUM_FLAVORS=3 AMREX_CUDA_ARCH=70
python ../Scripts/initial_conditions/st7_empty_particles.py
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_coll_equi_test
python ../Scripts/tests/coll_equi_test.py
rm -rf plt*
```

### 12. Fermi-Dirac test
Keep Stage-11 CUDA 3F build (rebuild if unsure).
```bash
python ../Scripts/initial_conditions/st9_empty_particles_multi_energy.py
python ../Scripts/initial_conditions/nsm_constant_background_rho_Ye_T_writer.py
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_fermi_dirac_test \
  nuceos_table_name=/home/erick/tables/LS220.h5 \
  nulib_table_name=/home/erick/tables/NuLib_SFHo.h5
python ../Scripts/data_reduction/write_particles_all_domain.py
python ../Scripts/tests/fermi_dirac_test.py
rm -rf plt* *pdf rho_Ye_T.hdf5
```

### 13. Fermi-Dirac Reflecting
Same as 12 with `inputs_fermi_dirac_test_reflecting` + same table CLI overrides.

### 14. Monochromatic Isotropic-Kernel Scattering
**Still CUDA 3F.** Do not rebuild as 2F.
```bash
rm -rf plt*
python ../Scripts/initial_conditions/st12_monocromatic_isotropic_scattering_test.py
./main3d.gnu.TPROF.MPI.CUDA.ex ../sample_inputs/inputs_monocromatic_isotropic_scattering_test
python ../Scripts/tests/coll_mono_isot_scat_test.py
# note n_total.pdf, n_x_and_n_others_*.pdf
rm -rf plt*
```

### 15. Metric test
```bash
cd /home/erick/software/Emu/unit_test
make
./main3d.gnu.DEBUG.MPI.ex
```

## REPORTING (chat only)

Per stage: name, pass/fail, one-line key output (prefer numbers from the baseline table).

**On first failure:** stop; quote error; classify (build / CUDA arch / table path /
wrong script path / leftover artifacts / physics / missing dep); propose fix;
do **not** apply until confirmed.

**If all pass:** short summary — Stages 0–15, table mechanism (CLI overrides),
`AMREX_CUDA_ARCH=70`, deviations (no `-np 4`, no Docker mounts, no artifact archive).
