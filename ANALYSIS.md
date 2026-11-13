# Analysis objective

Complete the LAVA-Unstructured *Multielement Airfoil RANS* tutorial
(https://www.nas.nasa.gov/LAVA/unst_docs/tutorials/multielement_airfoil/multielement_airfoil/)
using CAPE. The geometry is a 2D, three-element high-lift airfoil
(slat, main, flap) extracted from the CRM-HL configuration. Run steady RANS
(Spalart-Allmaras) at Mach 0.2, Reynolds number 5e6 per meter, at two angles
of attack (16 deg, the tutorial condition, plus 8 deg for comparison), and
collect the integrated loads (CL, CD, CM) for the full assembly and for each
element.

# Procedure

The run matrix is in `matrix.csv` (2 cases, config `mea2d-lava01`), and the
main CAPE file is `pyLava.json`. The basic flow is:

1. **Mesh.** The volume mesh is not in the repo; generate it with the LAVA
   Voronoi mesher in `inputs/mesh/` (the input `voro_mea2d.yml` and surface
   triangulation `crmhl-2dcut-triangulation.cgns` are already there):

       cd inputs/mesh
       export OMP_NUM_THREADS=10
       lavavoro.AVX2 ./voro_mea2d.yml

   The executable name depends on the installation/architecture; check what
   is available (`compgen -c lavavoro` or consult the LAVA docs). The mesher
   creates `voro_mea2d.cgns`, which is what each case's `run.inputs`
   references via the symlink `Mesh.LinkFiles` in `pyLava.json`.

2. **Verify setup.** Run

       cape -I 0 --no-start

   and confirm that `mea2d-lava01/m0.20a16.0b0.0/` contains `case.json`,
   `conditions.json`, `run.00.inputs`, `run_lava.pbs`, and a
   `voro_mea2d.cgns` link. Inspect `run.00.inputs`: the reference
   conditions (mach, alpha, Re, temperature) must match the run matrix.

3. **Adapt resources.** Before starting cases, make sure the settings fit
   this machine: `RunControl.lavaunst.executable` (name of the
   LAVA-Unstructured binary), `RunControl.mpi.executable` and `nProc`,
   `RunControl.Environ.OMP_NUM_THREADS`, and the `PBS`/`ShellCmds` sections
   if running on a PBS system (set `RunControl.qsub: true` in that case).

4. **Run.** Start both cases and enter the standard CAPE loop: `cape wait`,
   act on cases (`cape report`, check subfigures), extend if needed with
   `cape perform extend -I {CASE}`, approve with
   `cape perform approve -I {CASE}`.

5. **Extract.** The DataBook components are `all`, `slat`, `main`, `flap`
   (the loads groups in the input file). Extraction happens as part of
   `cape perform approve`.

# Completion criteria

This is a steady RANS case. Per the LAVA tutorial, the continuity residual
(`l2resr0`) should drop about 8 orders of magnitude within ~1500 iterations;
the SA turbulence residual (`l2turbresr0`) stalls after ~3 orders.

- Approve a case when the force/moment histories (CL, CD, CLM for `all`)
  are visually flat and the residuals are not growing.
- Extend a case (one 2000-iteration phase at a time) if forces are still
  trending or `l2resr0` is larger than about 1e-6 at the end of a phase.
- Hard cutoff: do not run more than 6000 total iterations for a case; if
  still not converged, mark it for human review rather than continuing.
