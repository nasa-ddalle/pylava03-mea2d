# Demo: Agentic LAVA-Unstructured on the Multielement Airfoil

This repo is a CAPE tutorial
([pyLava](../..) example 03) for running the
[LAVA-Unstructured multielement airfoil RANS tutorial](https://www.nas.nasa.gov/LAVA/unst_docs/tutorials/multielement_airfoil/multielement_airfoil/)
under the control of an *agent*: an AI assistant (for example
[OpenCode](https://opencode.ai/) connected to an OpenAI-compatible model
endpoint) that drives CAPE instead of a human.

The case is a Reynolds-Averaged Navier-Stokes verification case from the
NASA Langley
[Turbulence Modeling Resource](https://turbmodels.larc.nasa.gov/multielementverif.html):
a 2D, three-element high-lift airfoil (slat, main, flap) at
$M = 0.2$, $Re = 5\times10^6$, and $\alpha = 16^\circ$.

## What is in the repo

- `inputs/unst_mea2d.inputs`: solver input file from the LAVA tutorial,
  used verbatim as the CAPE template (the `CartInputFile` option). CAPE
  rewrites the `reference conditions` from the run matrix for each case and
  writes the result as `run.00.inputs` in each case folder.
- `inputs/mesh/crmhl-2dcut-triangulation.cgns`: surface triangulation from
  the tutorial (input to the mesh generator).
- `inputs/mesh/voro_mea2d.yml`: LAVA Voronoi mesher input from the tutorial.
- `pyLava.json`: master CAPE JSON file. It selects
  `"LAVASolver": "unstructured"`, one 2000-iteration phase, 2 MPI ranks
  with 32 OpenMP threads, and databook/report definitions for the four
  loads groups (`all`, `slat`, `main`, `flap`).
- `matrix.csv`: the run matrix; two cases sharing config `mea2d-lava01`
  (the tutorial condition $\alpha=16^\circ$, plus $\alpha=8^\circ$).
- `AGENTS.md` / `ANALYSIS.md`: instructions to the agent. `ANALYSIS.md` is
  where the analysis objective, procedure, and completion criteria are
  described; edit it to change what the agent tries to do.

The volume mesh `inputs/mesh/voro_mea2d.cgns` is intentionally *not* in the
repo: it is produced by running the mesh generator (below), which is part
of the tutorial.

## Prerequisites

- CAPE (this repo/stage targets `cape.pylava` with LAVA-Unstructured
  support, i.e. the `LAVASolver: "unstructured"` option).
- The LAVA suite: the unstructured solver executable (e.g.
  `lavaunst.3D.NS1`; exact name depends on build/architecture) and the
  Voronoi mesh generator (e.g. `lavavoro.AVX2`). See the
  [LAVA installation guide](https://www.nas.nasa.gov/LAVA/setup_docs/installation/).
  These are needed on the machine that runs CFD, not necessarily where you
  drive CAPE from.
- An agent harness (OpenCode) and an approved model endpoint. As with any
  export-controlled or internal code, do not point public cloud endpoints
  at ITAR/EAR-restricted material.

## Step 1: Generate the volume mesh

The unstructured solver needs a 3D mesh with one cell in the depth
direction (LAVA-Unstructured does not support true 2D meshes). Run the
Voronoi mesher where the LAVA tools are installed:

```bash
cd inputs/mesh
export OMP_NUM_THREADS=10
lavavoro.AVX2 ./voro_mea2d.yml
cd ../..
```

On Pleiades, you can instead generate the mesh as a batch job on a Rome
node (interactive Rome access is only available from the Pleiades front
ends, and not at all from other hosts):

```bash
cd inputs/mesh
qsub run_mesh.pbs
```

`run_mesh.pbs` requests one full `rom_ait` node in the `sls_aero1`
queue (the resources of `iqsub rom`); adjust the `-W group_list` if you
charge to a different group.

This creates `inputs/mesh/voro_mea2d.cgns`. (The provided
`voro_mea2d.yml` produces a mesh coarser than required for mesh-insensitive
results; reduce `reference_length` there if you want a finer mesh.)

If you prefer to experiment in a sandbox instead of modifying this repo,
first run `./copy-files.py` and work in the `work/` folder it creates.

## Step 2: Check the CAPE setup

From this folder:

```bash
cape -I 0 --no-start
```

This prepares case 0 without starting it. You should see:

```
Using pylava JSON file: pyLava.json
Case Config/Run Directory        Status  Iterations  Que CPU Time
---- --------------------------- ------- ----------- --- --------
0    mea2d-lava01/m0.20a16.0b0.0 ---     /           .

Set up 1 job(s) but did not start.
```

Inspect `mea2d-lava01/m0.20a16.0b0.0/`. It should contain:

- `case.json` — run-control settings for this case (`LAVASolver:
  "unstructured"`, phase iters, `lavaunst` executable, `mpi` options)
- `conditions.json` — the run-matrix values for this case
- `run.00.inputs` — the LAVA-Unstructured input file for phase 0. Check
  that `reference conditions` (mach, alpha, Re, temperature) match the run
  matrix, that `unst.implicit.number of steps` is 2000, and that
  `unst.grid.file` is `voro_mea2d.cgns`.
- `run_lava.pbs` — the job script (runs `python3 -m cape.pylava run`)
- `voro_mea2d.cgns` — symlink to the mesh in `inputs/mesh/`

Delete the `mea2d-lava01/` folder when done; the agent will recreate it.

### Settings you may have to change in `pyLava.json`

- `RunControl.lavaunst.executable`: name/path of the unstructured solver
  binary on your system (currently `lavaunst.3D.NS1.AVX2`, the Rome
  build; Rome has no AVX-512).
- `RunControl.mpi.executable` and `RunControl.nProc`: MPI launcher and
  rank count; the tutorial uses `mpirun -np 2`. The `mpi.args` include
  `--bind-to none`: without it OpenMPI pins each rank to a single core
  and the 32 OpenMP threads of each rank fight over that core.
- `RunControl.Environ.OMP_NUM_THREADS`: threads per rank (32 in the
  tutorial, 64 cores total).
- `PBS` and `ShellCmds`: queue resources and environment modules. The
  defaults target Pleiades Rome nodes: one `rom_ait` node (128 cores)
  in queue `sls_aero1` with 8 h walltime and `group_list=e0847`
  (compare `iqsub rom --dry-run`). `"qsub"` is `true`; set it to
  `false` to run cases directly on the local host instead. `JumpHosts`
  routes `qsub`/`qstat`/`qdel` over SSH (via `cfe02` to `pfe25`) when
  working from a non-Pleiades host such as `cb10n08`; delete it when
  working on a Pleiades front end.
- `RunInputs`: anything here is copied into each case's `run.inputs`
  (currently used to write restart files every 400 steps instead of the
  template's 20000).

## Step 3: Run with the agent

Start your agent (e.g. OpenCode) *in this folder*. It will read
`AGENTS.md`, which points it at CAPE's general instructions and at
`ANALYSIS.md`, whose filled-in objective reads roughly as:

> Mesh the multielement airfoil if needed, run the two-case matrix
> (`cape -I 0 --no-start` first to check the setup), and drive both cases
> to converged steady RANS, extending once per 2000-iteration phase up to
> at most 6000 iterations, then approve and extract forces and moments.

A typical agentic session then proceeds as in the companion
[pyfun05-agentic_arrow](../pyfun05-agentic_arrow/README.rst) demo:

1. The agent checks the run matrix and prepares a case with
   `cape -I 0 --no-start` for inspection.
2. It adapts resource settings (`lavaunst.executable`, MPI, PBS) to the
   local machine, then starts the cases (locally or, with `qsub: true`, on
   PBS).
3. It monitors with `cape wait` and `cape check`, generating convergence
   reports (`cape report`) with
   `cape list-report-subfigs` / `cape open subfig ...` to look at
   force/moment and residual histories.
4. It extends cases that have not converged
   (`cape perform extend -I {CASE}`) and approves finished ones
   (`cape perform approve -I {CASE}`), which also extracts the DataBook.

The `case` report includes two `Tecplot` subfigures rendered from
`inputs/p-xz.lay` and `inputs/p-xz-mesh.lay` (pressure contours and mesh
on the $x$-$z$ plane, read straight from `volume/Unst.volLatest.cgns`).
Building the PDF on a Cabeus host therefore needs, in addition to the
CAPE modules:

    module load pkgsrc          # PDFLaTeX (fancyhdr & friends)
    module load tecplot/2025r2  # batch tec360; 2023r2's CGNS loader is
                                # too old for LAVA's CGNS 4.1.2 output

## Expected results

Following the LAVA tutorial: the continuity residual (`l2resr0`) should
drop by about 8 orders of magnitude within about 1500 iterations; the SA
turbulence residual (`l2turbresr0`) stalls after roughly 3 orders. CAPE's
`L2` report subfigure reads the same `monitor/Unst.data.iter` file you
would otherwise view with `plotHist.py ns l2resr0 l2turbresr0 -doLogy`.
Useful force-history checks for the agent include `cl_all`, `cd_all` and
`cmy_all` (mapped to CAPE's `CL`, `CD`, `CLM`), all of which are always
written to `Unst.data.iter` along with the per-element loads.

## References

- LAVA-Unstructured tutorial: [Multielement Airfoil RANS](https://www.nas.nasa.gov/LAVA/unst_docs/tutorials/multielement_airfoil/multielement_airfoil/)
- LAVA-Unstructured input file: [Solver Inputs](https://www.nas.nasa.gov/LAVA/unst_docs/input_file_unstructured/)
- NASA Langley Research Center, *2D Multielement Airfoil — Turbulence
  Modeling Resource*, <https://turbmodels.larc.nasa.gov/multielementverif.html>
