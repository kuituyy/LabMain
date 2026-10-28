# gromacs-md

## Requirements

- GROMACS (tested with 2026.4 Homebrew)
- macOS Terminal / zsh (commands are also broadly compatible with Linux shells)
- UCSF ChimeraX or a similar molecular viewer
- A **publicly shareable**, structurally validated two-chain PDB file
- A text editor

```bash
gmx --version
```

## GROMACS file formats

| Extension | Purpose |
|---|---|
| `.pdb` | Structure coordinates, usually in ångströms (Å) |
| `.gro` | Coordinates and simulation box, in nanometers (nm) |
| `.top` | Topology of the complete system |
| `.itp` | Included molecular topology or restraint definitions |
| `.mdp` | Simulation settings |
| `.tpr` | Binary run input assembled by `grompp` |
| `.cpt` | Checkpoint containing the state needed to continue a run |
| `.edr` | Energies and thermodynamic properties |
| `.xtc` | Compressed trajectory |
| `.ndx` | Custom atom index groups |
| `.xvg` | Plot-ready numerical data |

## 1. Prepare the complex and generate its topology

Make sure both proteins are in the **same coordinate model**, with appropriate chain identifiers or `TER` records. Distinct PDB `MODEL` blocks are not interchangeable with two chains in a single complex. Check for missing atoms, alternate conformations, unsupported residues, and initial atomic overlaps.

```bash
gmx pdb2gmx -f complex_input.pdb -o complex_processed.gro -p complex.top -ignh -ff amber99sb-ildn -water tip3p
```

- `-f`: input structure.
- `-o`: output structure with force-field-compatible atoms.
- `-p`: output topology.
- `-ignh`: reconstruct hydrogen atoms according to the chosen force field.
- `-ff`: choose the force field.
- `-water`: select the water model.

**Verify:** Both proteins have been read; residue counts, termini, protonation states, molecule types, and topology includes are correct. A generated topology does not establish that the starting complex is physically plausible.

## 2. Define the simulation box

```bash
gmx editconf -f complex_processed.gro -o complex_box.gro -c -d 1.2 -bt dodecahedron
```

- `-c`: center the complex.
- `-d 1.2`: provide approximately 1.2 nm of padding to the periodic boundary.
- `-bt dodecahedron`: use a rhombic dodecahedral box, often more volume-efficient than a cube for globular solutes.

Inspect its box vectors:

```bash
tail -n 1 complex_box.gro
```

## 3. Add explicit water

```bash
cp complex.top complex_before_solvation.top
gmx solvate -cp complex_box.gro -cs spc216.gro -o complex_solv.gro -p complex.top
```

`spc216.gro` supplies an initial arrangement of three-site water molecules. The water interaction parameters come from the **TIP3P topology** chosen above, not from the name of the coordinate template.

Check the output:

```bash
head -n 2 complex_solv.gro
tail -n 15 complex.top
```

**Verify:** The increase in atom count matches the water added. Account for any crystallographic waters already present.

## 4. Neutralize the system and add salt

Prepare an `ions.mdp` file suitable for constructing the input topology. This is a preparation step, not a production simulation. Use a consistent electrostatics setup such as PME.

```bash
gmx grompp -f ions.mdp -c complex_solv.gro -p complex.top -o ions.tpr
cp complex.top complex_before_ions.top
gmx genion -s ions.tpr -o complex_ions.gro -p complex.top -pname NA -nname CL -neutral -conc 0.15
```

When prompted, select the **SOL** group, not `System` or `Protein`.

**Verify:** Ion counts neutralize the net charge; the number of replaced water molecules is correct; `.gro` and `.top` remain consistent.

## 5. Run energy minimization

Investigate severe steric clashes before running MD. A minimization may reduce clashes, but it may also get trapped in an incorrect pose. Position restraints should reference atom indices **within each molecule type** and must be activated explicitly in the topology.

Example `em.mdp`:

```ini
integrator       = steep
emtol            = 1000.0
emstep           = 0.001
nsteps           = 50000
cutoff-scheme    = Verlet
coulombtype      = PME
rcoulomb         = 1.2
vdwtype          = Cut-off
rvdw             = 1.2
pbc             = xyz
constraints     = none
comm-mode        = None
```

Without restraints:

```bash
gmx grompp -f em.mdp -c complex_ions.gro -p complex.top -o em.tpr
gmx mdrun -deffnm em
```

When using restraints, enable the appropriate `define` macro in the `.mdp` file and pass reference coordinates with `-r`. For example, if your topology includes restraints under `#ifdef POSRES_LOCAL`, use `define = -DPOSRES_LOCAL` and:

```bash
gmx grompp -f em.mdp -c complex_ions.gro -r complex_ions.gro -p complex.top -o em.tpr
```

**Verify:** Maximum force (`Fmax`), convergence status, remaining clashes, residue geometry, and structural displacement. Potential energy alone is not evidence that docking was successful.

## 6. Equilibrate temperature (NVT)

In `nvt.mdp`, configure a time step of 0.002 ps and 50,000 steps for a 100 ps tutorial run. Set a suitable thermostat (for example, `V-rescale`), `ref_t = 300`, `pcoupl = no`, and `constraints = h-bonds`. For the **first** MD run, generate velocities (`gen_vel = yes`) and apply position restraints explicitly if needed.

```bash
gmx grompp -f nvt.mdp -c em.gro -r em.gro -p complex.top -o nvt.tpr
gmx mdrun -deffnm nvt
gmx energy -f nvt.edr -o temperature_nvt.xvg
```

Select `Temperature` in the energy menu.

**Verify:** Temperature fluctuates around its setpoint without a sustained drift. Remember that an `.xvg` file is data, not a shell command; use `head`, `tail`, Python, or a plotting program to inspect it.

## 7. Equilibrate pressure and density (NPT)

Continue from the NVT checkpoint without regenerating velocities. Select an appropriate pressure-coupling method, such as `C-rescale`; set a target pressure (for example, 1 bar). If position restraints are active, consider `refcoord-scaling = all`.

```bash
gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -r nvt.gro -p complex.top -o npt.tpr
gmx mdrun -deffnm npt
gmx energy -f npt.edr -o density_npt.xvg
```

Check **Density**, **Volume**, **Temperature**, and **Pressure**. Instantaneous pressure can fluctuate dramatically in small systems; trends and adequate sampling matter more than one short-run average.

For an additional 400 ps, *only after the initial NPT has completed*:

```bash
gmx convert-tpr -s npt.tpr -extend 400 -o npt_extended.tpr
gmx mdrun -s npt_extended.tpr -cpi npt.cpt -deffnm npt -append
```

Never run two processes that write to the same output file names simultaneously.

## 8. Production MD and analysis (planned)

After verifying equilibration, start a **separate unrestrained** simulation. Do not mistake restrained NPT for an unbiased production trajectory.

For protein–protein complexes, plan to analyze:

1. Backbone RMSD and per-residue RMSF of each protein.
2. Substrate RMSD **after fitting to the enzyme**, to distinguish relative motion from internal folding changes.
3. Interprotein contacts, hydrogen bonds, and active-site distances over time.
4. Periodic-boundary artifacts, using appropriate trajectory centering and reimaging.
5. Independent replicate runs and convergence. No observed transition in one short trajectory does **not** rule out a proposed mechanism.

## Common problems and solutions

| Problem | Explanation / fix |
|---|---|
| `File 'complex.top' does not exist` | Check `pwd`, `ls`, and relative paths (`../` means the parent directory). |
| A complex was exported as separate `MODEL` blocks | Re-export the intended assembly in one coordinate model; do not merely concatenate incompatible coordinates. |
| ChimeraX cannot find an atom after `.gro` → `.pdb` | Check actual residue numbering and chain IDs. PDB atom serial numbers are **not** the same as ChimeraX atom-name selectors. |
| `grompp` notes center-of-mass motion with restraints | Understand the effect before deciding whether to change `comm-mode`; a `NOTE` is not automatically fatal. |
| `.xvg: command not found` | View or plot the data file; do not execute it in the shell. |

## References

- Lemkul, J. A. (2024). *Introductory Tutorials for Simulating Protein Dynamics with GROMACS.* **Journal of Physical Chemistry B**, 128, 9418–9435. https://doi.org/10.1021/acs.jpcb.4c04901
- Yao, Q. et al. (2012). *Structural mechanism of ubiquitin and NEDD8 deamidation catalyzed by bacterial effectors that induce macrophage-specific apoptosis.* https://doi.org/10.1073/pnas.1210831109
- [Official GROMACS documentation](https://manual.gromacs.org/current/)

## Roadmap

- [ ] Add reviewed, reusable `.mdp` example files.
- [ ] Add diagrams and screenshots using **public** protein structures.
- [ ] Provide validated trajectory-analysis examples and plots from a **public** dataset.
- [ ] Test reproducibility on a second computer.

## Data privacy and scientific reproducibility

The repository intentionally excludes unpublished structures, experimental measurements, coordinates, trajectories, and derived model files. Use a separate **private** research directory for confidential data. Review every file before committing, even when `.gitignore` is configured.

