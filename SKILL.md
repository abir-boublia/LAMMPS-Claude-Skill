---
name: "lammps"
description: "When the user asks for help writing, debugging, or understanding LAMMPS molecular dynamics input scripts, data files, force field setup, or simulation workflows (Release 30Sep2026)."
---

# LAMMPS Expert Skill (Release 30 Sep 2026)

You are an expert in LAMMPS (Large-scale Atomic/Molecular Massively Parallel Simulator), the classical molecular dynamics code developed at Sandia National Laboratories. This skill covers Release 30Sep2026, distributed under GPLv2, documented at https://docs.lammps.org/.

## Core Principles

- Always produce syntactically valid LAMMPS input scripts.
- Specify version-sensitive commands with the correct syntax for the 30Sep2026 release.
- When a feature requires an optional package, state which package and how to enable it.
- Prefer established, well-tested approaches over exotic or undocumented tricks.
- Cite the relevant manual section (e.g., "See `pair_style lj/cut` command reference") when introducing non-trivial commands.

---

## 1. Input Script Structure

A LAMMPS input script is a plain-text file processed line by line. The canonical order is:

```
# 1. Initialization
units           real          # or metal, lj, si, cgs, electron, micro, nano
dimension       3
boundary        p p p         # p=periodic, f=fixed, s=shrink-wrap, m=shrink-wrap+min
atom_style      full          # or atomic, charge, molecular, bond, angle, etc.

# 2. System definition
read_data       system.data   # or: lattice + region + create_box + create_atoms
# Alternative: read_restart restart.file

# 3. Force field / interaction settings
pair_style      lj/cut/coul/long 12.0
pair_coeff      * * ...
bond_style      harmonic
bond_coeff      ...
angle_style     harmonic
angle_coeff     ...
kspace_style    pppm 1.0e-4
special_bonds   lj/coul 0 0 0.5   # for OPLS-style 1-4 scaling

# 4. Simulation settings
neighbor        2.0 bin
neigh_modify    every 1 delay 0 check yes
group           mobile type 1 2
velocity        all create 300.0 12345 dist gaussian

# 5. Fixes (time integration, thermostats, barostats)
fix             1 all nve
fix             2 all langevin 300.0 300.0 100.0 12345
# or: fix 1 all nvt temp 300 300 100
# or: fix 1 all npt temp 300 300 100 iso 1.0 1.0 1000

# 6. Output
thermo          1000
thermo_style    custom step temp pe ke etotal press vol density
dump            1 all custom 5000 dump.lammpstrj id type x y z vx vy vz
restart         100000 restart.a restart.b

# 7. Run
timestep        1.0    # in units of the chosen unit system
run             500000
write_data      final.data
```

### Key syntax rules
- Lines starting with `#` are comments; `#` after a command comments the rest of the line.
- Long lines can be continued with `&` at the end.
- Variables: `variable name style args` — styles include `equal`, `atom`, `loop`, `index`, `string`, `getenv`, `file`, `format`, `python`, `timer`, `internal`, `vector`, `uloop`.
- Substitution: `${name}` or `$x` (single-char) or `$(expression)` for immediate evaluation.
- `include filename` — read another input file.
- `jump filename label` — jump to a label in another (or the same) file.
- `label name` — target for `jump`.
- `if "condition" then "command" else "command"` — conditional execution.
- `loop`, `next`, `jump` enable iteration patterns.

---

## 2. Units Systems

| Style | Distance | Time | Energy | Mass | Charge |
|-------|----------|------|--------|------|--------|
| **real** | Angstrom | fs | kcal/mol | g/mol | e |
| **metal** | Angstrom | ps | eV | g/mol | e |
| **si** | m | s | J | kg | C |
| **cgs** | cm | s | erg | g | statcoul |
| **electron** | Bohr | fs | Hartree | amu | e |
| **micro** | um | us | pg*um^2/us^2 | pg | pC |
| **nano** | nm | ns | ag*nm^2/ns^2 | ag | e |
| **lj** | sigma | tau | epsilon | m | e |

All commands must use values consistent with the chosen `units` style. The `units` command must appear before any other command that uses dimensional quantities.

---

## 3. Atom Styles

The `atom_style` determines what per-atom attributes are stored:

- **atomic** — only position, velocity, type (metals, LJ fluids)
- **charge** — adds per-atom charge (ionic systems)
- **molecular** — adds bonds, molecule ID (uncharged polymers)
- **full** — adds bonds + charge (most biomolecular / organic FFs)
- **bond** — bonds only, no angles
- **angle** — bonds + angles
- **dipole** — adds dipole moment
- **electron** — for eFF
- **ellipsoid** — aspherical particles
- **sphere** — finite-size spherical, adds radius + rmass
- **peri** — Peridynamics
- **smd** — Smooth Mach Dynamics
- **spin** — magnetic spin
- **hybrid** — combine multiple styles

The atom style must match the data file header and the force field requirements.

---

## 4. Force Fields — Pair Styles

### 4.1 Simple Pair Potentials
- `lj/cut`, `lj/cut/coul/cut`, `lj/cut/coul/long`, `lj/cut/coul/dsf`
- `lj/cut/coul/long/soft` — for free energy perturbation with softcore
- `buck`, `buck/coul/long` — Buckingham
- `morse` — Morse potential
- `born/coul/long` — Born-Mayer-Huggins
- `lj96/cut` — 9-6 LJ
- `mie/cut` — generalized Mie
- `lj/expand` — shifted LJ for finite-size particles

### 4.2 Many-Body Potentials
- `eam`, `eam/alloy`, `eam/fs` — Embedded Atom Method (metals)
- `meam`, `meam/spline` — Modified EAM (requires MEAM package)
- `tersoff`, `tersoff/zbl` — covalent semiconductors (Si, C, SiC, BN)
- `sw` — Stillinger-Weber (Si, GaN)
- `airebo`, `airebo/morse`, `rebo` — carbon/hydrocarbon (requires MANYBODY package)
- `reaxff` — ReaxFF reactive force field (requires REAXFF package)
- `comb`, `comb3` — charge-optimized many-body
- `bop` — bond-order potential
- `adp` — angular-dependent potential
- `edip` — environment-dependent interatomic potential
- `eim` — embedded ion method
- `vashishta` — for SiO2, SiC
- `polymorphic` — general tabulated many-body

### 4.3 Charged / Long-Range
- `coul/long` — long-range Coulomb (needs kspace_style)
- `coul/dsf`, `coul/wolf` — damped-shifted force / Wolf summation (no kspace needed)
- `lj/long/coul/long` — long-range LJ + Coulomb
- kspace solvers: `ewald`, `pppm`, `pppm/disp`, `msm`
  - `pppm` is fastest for most systems (FFT-based)
  - `pppm/disp` handles long-range LJ dispersion
  - `msm` works with non-periodic boundaries

### 4.4 Coarse-Grained
- `dpd` — dissipative particle dynamics
- `gayberne` — Gay-Berne ellipsoids
- `resquared` — RE-squared
- `brownian` — Brownian dynamics
- `colloid` — colloidal particles
- `oxdna/*/` and `oxrna2/*/` — coarse-grained DNA/RNA (requires CG-DNA package)
- `spica` — SPICA coarse-grained

### 4.5 Machine-Learning Potentials
- `pace`, `pace/extrapolation` — Atomic Cluster Expansion (ML-PACE package)
- `snap` — Spectral Neighbor Analysis Potential (ML-SNAP package)
- `pod` — Proper Orthogonal Descriptors (ML-POD package)
- `quip` — GAP/SOAP via QUIP library (ML-QUIP package)
- `hdnnp` — Behler-Parrinello neural network (ML-HDNNP package)
- `runner` — RuNNer neural network (ML-RUNNER package)
- `mliap` — unified ML-IAP interface supporting multiple descriptor types
- External interfaces: ANI, ChIMES, DeepPot, HIPNN, MTP via `pair_style mliap`

### 4.6 Reactive
- `reaxff` — ReaxFF (requires REAXFF package); needs `fix qeq/reaxff` for charge equilibration
- `airebo` — reactive hydrocarbon
- `comb`, `comb3` — charge-optimized many-body

### 4.7 Water Models
- **TIP3P**: `pair_style lj/cut/coul/long`, rigid bonds via `fix shake` or `fix rigid`
- **SPC/E**: same pair style, SPC/E parameters, `fix shake`
- **TIP4P**: `pair_style lj/cut/tip4p/long`, 4th virtual site
- **TIP5P**: requires appropriate pair_style and 2 virtual sites

### 4.8 Hybrid & Overlay
- `pair_style hybrid style1 args1 style2 args2 ...` — different styles for different type pairs
- `pair_style hybrid/overlay style1 args1 style2 args2 ...` — sum contributions from multiple styles
- `pair_coeff I J style1 args` — must specify which sub-style for each pair

---

## 5. Bonded Interactions

### Bond styles
`harmonic`, `fene`, `fene/expand`, `morse`, `nonlinear`, `class2`, `quartic`, `table`, `hybrid`

### Angle styles
`harmonic`, `charmm`, `cosine`, `cosine/squared`, `cosine/periodic`, `class2`, `table`, `hybrid`

### Dihedral styles
`harmonic`, `charmm`, `multi/harmonic`, `opls`, `helix`, `class2`, `table`, `hybrid`, `fourier`

### Improper styles
`harmonic`, `cvff`, `umbrella`, `class2`, `hybrid`

### Force Field Compatibility
- **CHARMM**: `atom_style full`, `pair_style lj/charmm/coul/long`, `special_bonds charmm`, `dihedral_style charmm`, `angle_style charmm`
- **AMBER**: `atom_style full`, `pair_style lj/cut/coul/long`, `special_bonds amber`, `bond_style harmonic`, `angle_style harmonic`
- **OPLS**: `atom_style full`, `pair_style lj/cut/coul/long`, `special_bonds lj/coul 0 0 0.5`, `dihedral_style opls`
- **COMPASS / Class II**: `pair_style lj/class2/coul/long`, `bond_style class2`, `angle_style class2`, `dihedral_style class2`, `improper_style class2`
- **DREIDING**: `pair_style lj/cut/coul/long` or `buck/coul/long`, specific parameters
- **UFF**: Universal Force Field via appropriate pair_coeffs

---

## 6. Key Fix Commands

Fixes perform operations during timestepping.

### Time integration
- `fix nve` — microcanonical (constant NVE)
- `fix nvt` — Nose-Hoover thermostat (canonical)
- `fix npt` — Nose-Hoover thermostat + barostat (isothermal-isobaric)
- `fix nph` — barostat only
- `fix langevin` — Langevin thermostat (use with `fix nve`)
- `fix rigid`, `fix rigid/nvt`, `fix rigid/npt` — rigid body integration
- `fix rigid/small` — efficient for many small rigid bodies

### Constraints
- `fix shake` — constrain bonds/angles (e.g., water OH bonds, all X-H bonds)
- `fix rattle` — velocity-level SHAKE
- `fix enforce2d` — zero out z-forces/velocities for 2D simulations
- `fix momentum` — zero linear/angular momentum periodically

### Thermostats & barostats (alternatives)
- `fix temp/berendsen` — Berendsen (weak coupling, does not generate canonical ensemble)
- `fix temp/csvr` — canonical velocity rescaling (correct ensemble)
- `fix temp/csld` — canonical stochastic dynamics
- `fix press/berendsen` — Berendsen pressure coupling

### Walls & confinement
- `fix wall/lj93`, `fix wall/lj126`, `fix wall/harmonic` — planar walls
- `fix wall/region` — wall defined by a geometric region
- `fix indent` — spherical/cylindrical/planar indenter

### Force application
- `fix addforce` — constant force on a group
- `fix efield` — electric field
- `fix gravity` — gravitational field
- `fix smd` — steered molecular dynamics (constant velocity or constant force)
- `fix spring` — tether atoms to a point or another group

### Advanced
- `fix qeq/reaxff`, `fix qeq/point`, `fix qeq/shielded`, `fix qeq/slater`, `fix qeq/dynamic` — charge equilibration
- `fix gcmc` — grand canonical Monte Carlo insertions/deletions
- `fix deposit` — deposit atoms during simulation
- `fix evaporate` — remove atoms
- `fix adapt` — dynamically change simulation parameters
- `fix python/invoke` — call Python functions during simulation

---

## 7. Compute Commands

Computes calculate per-atom or global quantities.

### Thermodynamic
- `compute temp` — temperature
- `compute pressure` — pressure tensor
- `compute pe`, `compute ke` — potential/kinetic energy
- `compute stress/atom` — per-atom stress tensor

### Structural
- `compute rdf` — radial distribution function
- `compute msd` — mean squared displacement (diffusion)
- `compute coord/atom` — coordination number
- `compute centro/atom` — centro-symmetry parameter (defect detection)
- `compute cna/atom` — common neighbor analysis
- `compute voronoi/atom` — Voronoi tessellation (requires VORONOI package)

### Properties
- `compute property/atom` — extract per-atom properties
- `compute pair/local` — pairwise energies, forces, distances
- `compute bond/local` — bond distances, energies
- `compute reduce` — reduce per-atom to scalar (sum, min, max, ave)
- `compute chunk/atom` — divide atoms into spatial/type chunks
- `compute chunk/spread/atom` — spread chunk values to atoms

---

## 8. Output: Dump & Thermo

### dump
- `dump ID group style N file args`
- Styles: `custom`, `atom`, `cfg`, `xyz`, `dcd`, `xtc`, `image`, `movie`, `netcdf`, `h5md`, `yaml`
- `dump_modify` — change element names, sort, format, append, etc.
- `dump_modify ... element Cu Al` — for visualization tools
- `undump ID` — remove a dump

### thermo
- `thermo N` — output every N steps
- `thermo_style custom step temp pe ke etotal press vol lx ly lz density`
- Custom keywords: `step`, `elapsed`, `elaplong`, `dt`, `time`, `cpu`, `cpuremain`, `temp`, `press`, `pe`, `ke`, `etotal`, `evdwl`, `ecoul`, `epair`, `ebond`, `eangle`, `edihed`, `eimp`, `elong`, `etail`, `vol`, `density`, `lx`, `ly`, `lz`, `xlo`, `xhi`, `ylo`, `yhi`, `zlo`, `zhi`, `pxx`, `pyy`, `pzz`, `pxy`, `pxz`, `pyz`, `fmax`, `fnorm`, `cella`, `cellb`, `cellc`, `cellalpha`, `cellbeta`, `cellgamma`
- Can reference computes, fixes, variables: `c_myCompute`, `f_myFix`, `v_myVar`

---

## 9. Data File Format

The `read_data` file has a header and body:

```
Comment line (ignored)

100    atoms
2      atom types
50     bonds          # if applicable
1      bond types
0.0 50.0  xlo xhi
0.0 50.0  ylo yhi
0.0 50.0  zlo zhi
0.0 0.0 0.0  xy xz yz   # for triclinic (optional)

Masses

1 12.011
2 1.008

Atoms  # full

1 1 1 -0.5 1.0 2.0 3.0    # ID mol-ID type charge x y z
...

Bonds

1 1 1 2    # ID type atom1 atom2
...

Angles

1 1 1 2 3  # ID type atom1 atom2 atom3
...
```

The `Atoms` section format depends on atom_style:
- **atomic**: `ID type x y z`
- **charge**: `ID type q x y z`
- **molecular**: `ID mol-ID type x y z`
- **full**: `ID mol-ID type q x y z`

Optional sections: `Velocities`, `Pair Coeffs`, `Bond Coeffs`, `Angle Coeffs`, `Dihedral Coeffs`, `Improper Coeffs`.

---

## 10. Common Simulation Recipes

### 10.1 Energy Minimization
```
min_style    cg           # or sd, hftn, fire, quickmin, spin/...
minimize     1.0e-6 1.0e-8 10000 100000
# args: etol ftol maxiter maxeval
```

### 10.2 Equilibration Protocol
```
# Step 1: Minimize
minimize 1.0e-4 1.0e-6 1000 10000
reset_timestep 0

# Step 2: NVT equilibration
fix eq all nvt temp 300 300 100
run 100000
unfix eq

# Step 3: NPT equilibration (if needed)
fix eq2 all npt temp 300 300 100 iso 1.0 1.0 1000
run 500000
unfix eq2

# Step 4: Production
fix prod all nvt temp 300 300 100
run 1000000
```

### 10.3 Thermal Conductivity (NEMD — Muller-Plathe)
```
fix thermalcond all thermal/conductivity N edim Nbin
# Use with fix ave/chunk for temperature profile
compute layers all chunk/atom bin/1d z lower 0.05 units reduced
fix profiles all ave/chunk 10 100 1000 layers temp file temp_profile.dat
```

### 10.4 Elastic Constants (static deformation)
```
# Apply strain, measure stress
variable strain equal 0.001
change_box all x delta 0 ${strain} remap units box
run 0
variable pxx equal pxx
# Elastic modulus C11 = -delta(pxx)/strain
```

### 10.5 Radial Distribution Function
```
compute rdf all rdf 100 1 1 1 2 2 2
fix rdf_out all ave/time 100 1 100 c_rdf[*] file rdf.dat mode vector
```

### 10.6 Mean Squared Displacement / Diffusion
```
compute msd all msd com yes
fix msd_out all ave/time 1 1 100 c_msd[4] file msd.dat
# D = MSD / (2 * d * t), d = dimension
```

### 10.7 Two-Dimensional Simulations
```
dimension 2
boundary p p p
# Must add:
fix enforce all enforce2d
# And z-boundaries must allow no motion
```

---

## 11. Accelerator Packages

For hardware-accelerated runs:

| Package | Suffix | Target | Notes |
|---------|--------|--------|-------|
| **GPU** | `/gpu` | NVIDIA/AMD/Intel GPU | OpenCL, CUDA, HIP, SYCL |
| **KOKKOS** | `/kk` | GPU or CPU threads | Portable, CUDA/HIP/SYCL/OpenMP |
| **INTEL** | `/intel` | Intel CPU/GPU | AVX-512 vectorization |
| **OPENMP** | `/omp` | Multi-core CPU | OpenMP threading |
| **OPT** | `/opt` | CPU | Optimized pair styles |

Usage pattern:
```bash
# GPU package
lmp -sf gpu -pk gpu 1 -in input.lammps

# KOKKOS on GPU
lmp -k on g 1 -sf kk -pk kokkos -in input.lammps

# OPENMP
lmp -sf omp -pk omp 4 -in input.lammps
```
Or in the script:
```
package gpu 1
suffix gpu
# or: package kokkos ...
```

---

## 12. Optional Packages (Selected)

LAMMPS has ~95+ optional packages. Key ones:

- **MOLECULE** — molecular topology, angles, dihedrals, impropers
- **KSPACE** — long-range Coulomb solvers (Ewald, PPPM, MSM)
- **MANYBODY** — many-body potentials (Tersoff, SW, AIREBO, etc.)
- **RIGID** — rigid body dynamics
- **REPLICA** — multi-replica methods (NEB, PRD, TAD, temper)
- **REAXFF** — ReaxFF reactive force field
- **ML-PACE** — ACE machine learning potentials
- **ML-SNAP** — SNAP machine learning potentials
- **ML-IAP** — unified ML interatomic potential interface
- **ML-POD** — Proper Orthogonal Descriptors
- **ML-QUIP** — GAP/SOAP (needs QUIP library)
- **ML-HDNNP** — Behler-Parrinello neural networks (needs n2p2)
- **CG-DNA** — coarse-grained oxDNA/oxRNA models
- **GRANULAR** — granular particle interactions
- **SPH** — smoothed particle hydrodynamics
- **RHEO** — rheology simulations
- **ELECTRODE** — constant-potential electrode simulations
- **EXTRA-MOLECULE** — additional bonded styles
- **EXTRA-PAIR** — additional pair styles
- **PYTHON** — Python-embedded functions
- **QEQ** — charge equilibration methods
- **VORONOI** — Voronoi analysis (needs Voro++ library)
- **PLUMED** — enhanced sampling via PLUMED library
- **COLVARS** — collective variables module
- **MDI** — MolSSI Driver Interface for code coupling

Enable in CMake:
```bash
cmake -D PKG_MOLECULE=on -D PKG_KSPACE=on -D PKG_MANYBODY=on ../cmake
```

---

## 13. Building LAMMPS (CMake)

```bash
git clone -b release https://github.com/lammps/lammps.git
cd lammps
mkdir build && cd build
cmake -D BUILD_MPI=on \
      -D PKG_MOLECULE=on \
      -D PKG_KSPACE=on \
      -D PKG_MANYBODY=on \
      -D PKG_RIGID=on \
      -D PKG_REAXFF=on \
      -D PKG_ML-PACE=on \
      ../cmake
make -j $(nproc)
```

Presets for common configurations:
```bash
cmake -C ../cmake/presets/basic.cmake ../cmake     # common packages
cmake -C ../cmake/presets/most.cmake ../cmake       # most packages
cmake -C ../cmake/presets/all_on.cmake ../cmake     # all packages
```

Python module:
```bash
cmake -D BUILD_SHARED_LIBS=on -D LAMMPS_EXCEPTIONS=on ../cmake
make install
pip install lammps   # or: from the build dir
```

---

## 14. Python Interface

LAMMPS can be driven from Python as a library:

```python
from lammps import lammps

lmp = lammps()  # or lammps(cmdargs=["-log", "none"])
lmp.file("input.lammps")  # run an entire input script
# or line by line:
lmp.command("units real")
lmp.command("atom_style full")
lmp.command("read_data system.data")
# ...
lmp.command("run 10000")

# Extract data
natoms = lmp.get_natoms()
pe = lmp.get_thermo("pe")
coords = lmp.gather_atoms("x", 1, 3)  # type=1(double), count=3

lmp.close()
```

Other library interfaces: C (`lammps_open`, `lammps_command`, etc.), C++, Fortran, SWIG-based wrappers.

---

## 15. Variables and Expressions

```
variable T equal 300.0
variable P equal 1.0
variable myTemp equal temp      # thermodynamic keyword
variable myPE equal pe
variable strain equal "(lx - v_lx0) / v_lx0"
variable pressure atom -1.0*(c_stress[1]+c_stress[2]+c_stress[3])/3.0
```

Equal-style variables can use: `+`, `-`, `*`, `/`, `^`, `%`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `&&`, `||`, `!`, math functions (`sqrt`, `exp`, `ln`, `log`, `abs`, `sin`, `cos`, `tan`, `asin`, `acos`, `atan`, `atan2`, `ceil`, `floor`, `round`, `ramp`, `stagger`, `logfreq`, `stride`, `vdisplace`, `swiggle`, `cwiggle`), group functions (`count`, `mass`, `charge`, `xcm`, `vcm`, `fcm`, `bound`, `gyration`, `ke`, `angmom`, `torque`, `inertia`, `omega`), region functions, special functions (`sum`, `min`, `max`, `ave`, `trap`, `slope`, `gmask`, `rmask`, `grmask`, `next`, `is_active`, `is_available`, `is_defined`, `label2type`).

---

## 16. Groups and Regions

```
# Groups
group water type 1 2
group solute molecule 1:100
group mobile subtract all fixed
group interface intersect water surface_region
group delete water    # remove group definition

# Regions
region box block 0 50 0 50 0 50
region sphere sphere 25 25 25 10 side in
region slab block INF INF INF INF 20 30
region combined union 2 sphere slab
region complement intersect 2 box sphere side out
```

---

## 17. Restart, Read/Write

- `write_restart filename` — binary restart file
- `read_restart filename` — resume from restart
- `write_data filename` — write current state as data file
- `read_data filename add merge` — add atoms from another data file
- `restart N file1 file2` — periodic restarts (alternating files)

---

## 18. Common Errors and Debugging

### Typical issues
1. **"Unknown command"** — check spelling; the command may require a package not installed.
2. **"Pair style does not have a single/respa setting"** — wrong pair style for chosen run_style.
3. **"Bond/angle/dihedral atom missing"** — atoms left the box or topology is inconsistent; check boundary conditions and `special_bonds`.
4. **"Neighbor list overflow"** — increase `neighbor` skin distance or use `neigh_modify one ...`.
5. **"Out of range atoms"** — atoms outside box; check initial configuration and timestep size.
6. **"Energy not conserved"** — timestep too large, cutoff too small, or missing long-range correction.
7. **"PPPM grid is too large"** — requested accuracy too high for system size; relax kspace accuracy or use `pppm/stagger`.
8. **Lost atoms** — simulation blew up. Reduce timestep, check for overlapping atoms, minimize first.

### Best practices
- Always minimize before MD.
- Start with a small timestep and ramp up.
- Monitor temperature, pressure, and total energy during equilibration.
- Use `thermo_modify lost warn` during debugging.
- For new force fields, compare single-point energies against reference implementations.

---

## 19. Multi-Replica Methods

Require the REPLICA package:

- **NEB** (Nudged Elastic Band): `fix neb`, `neb` command — find minimum energy paths
- **PRD** (Parallel Replica Dynamics): `prd` command — accelerate rare events
- **TAD** (Temperature Accelerated Dynamics): `tad` command
- **Parallel Tempering**: `temper` command — replica exchange (REMD)
- **Hyper-dynamics**: `hyper` command

Run with multiple partitions:
```bash
mpirun -np 16 lmp -partition 4x4 -in neb.lammps
```

---

## 20. OpenKIM Integration

LAMMPS interfaces with the OpenKIM repository of interatomic potentials:
```
pair_style kim ModelName
pair_coeff * * Si O
```
This automatically downloads and uses a KIM model. Requires the KIM-API library and the KIM package.

---

## 21. Tips for AI-Assisted Scripting

Section 4.28 of the LAMMPS manual (30Sep2026) addresses working with AI coding agents:

- AI tools can help write LAMMPS input scripts but should be verified against the manual.
- Always check that commands exist in your LAMMPS version and that required packages are installed.
- Test scripts on small systems first before scaling up.
- The manual is the authoritative reference: https://docs.lammps.org/

When helping users:
1. Ask about their system (material, phase, size, periodicity).
2. Ask about their goal (equilibrium properties, transport, mechanical, reaction, etc.).
3. Recommend appropriate units, atom_style, and force field.
4. Provide a complete, runnable input script with comments.
5. Flag any required packages.
6. Suggest appropriate analysis (computes, fixes for averaging).
7. Warn about common pitfalls (timestep, equilibration, finite-size effects).

---

## Quick Reference: Command Categories

| Category | Key Commands |
|----------|-------------|
| Setup | `units`, `dimension`, `boundary`, `atom_style`, `lattice`, `region`, `create_box`, `create_atoms` |
| Read/Write | `read_data`, `read_restart`, `write_data`, `write_restart`, `read_dump` |
| Force Field | `pair_style`, `pair_coeff`, `bond_style`, `bond_coeff`, `angle_style`, `angle_coeff`, `dihedral_style`, `dihedral_coeff`, `improper_style`, `improper_coeff`, `kspace_style`, `special_bonds` |
| Settings | `neighbor`, `neigh_modify`, `group`, `velocity`, `set`, `mass` |
| Fixes | `fix nve/nvt/npt/nph`, `fix langevin`, `fix shake`, `fix rigid`, `fix wall/*`, `fix qeq/*`, `fix spring`, `fix adapt` |
| Computes | `compute temp`, `compute pressure`, `compute rdf`, `compute msd`, `compute stress/atom` |
| Output | `thermo`, `thermo_style`, `dump`, `dump_modify`, `fix ave/time`, `fix ave/chunk`, `fix print` |
| Run | `run`, `minimize`, `timestep`, `reset_timestep`, `run_style verlet/respa` |
| Control | `variable`, `if`, `jump`, `label`, `next`, `loop`, `include`, `shell`, `python` |
| Replica | `neb`, `prd`, `tad`, `temper`, `hyper` |
