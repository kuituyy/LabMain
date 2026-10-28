# Chapter 2: Generating a GROMACS Topology with `pdb2gmx`

## 1. Introduction

A PDB file contains atomic coordinates, but coordinates alone are not sufficient to perform a molecular dynamics (MD) simulation.

GROMACS also needs information about molecular connectivity, atom types, atomic charges, bonded interactions, and nonbonded interactions.

This information is provided by a **molecular topology**.

In this chapter, we will generate a topology for a protein–protein complex using the GROMACS tool `pdb2gmx`.

## 2. Learning objectives

By the end of this chapter, you will be able to:

- Understand the purpose of a molecular mechanics force field.
- Generate a protein topology using `pdb2gmx`.
- Select a protein force field and a compatible water model.
- Understand the `.gro`, `.top`, and `.itp` files.
- Recognize common topology-generation errors.

## 3. What is a force field?

A molecular mechanics force field is a mathematical model used to describe interactions between atoms.

It includes parameters for:

- Bond stretching
- Bond-angle bending
- Dihedral rotations
- Electrostatic interactions
- van der Waals interactions

Different force fields use different parameter sets and assumptions.

For this tutorial, we use **AMBER99SB-ILDN**, a protein force field supported by GROMACS.

We pair it with the **TIP3P water model**.

Force-field selection should be based on the molecular system and the scientific questions being investigated. Parameters from different force fields should not be mixed without validation.

## 4. Prepare the working directory

Open Terminal and navigate to the directory containing your prepared PDB file.

```bash
mkdir -p gromacs-tutorial
cd gromacs-tutorial
```

Copy your validated protein–protein structure into this directory and name it:

```text
complex_prepared.pdb
```

Check that it exists:

```bash
ls -lh complex_prepared.pdb
```

## 5. Run `pdb2gmx`

Execute:

```bash
gmx pdb2gmx \
  -f complex_prepared.pdb \
  -o complex_processed.gro \
  -p complex.top \
  -ignh \
  -ff amber99sb-ildn \
  -water tip3p
```

### Explanation of the options

| Option | Meaning |
|---|---|
| `-f` | Input PDB structure |
| `-o` | Output coordinate file |
| `-p` | Output system topology |
| `-ignh` | Ignore input hydrogen atoms and reconstruct them |
| `-ff` | Select the force field |
| `-water` | Select the water model |

The command may ask you to select terminal-group configurations or other chemistry-related options.

Do not assume that the default protonation states and termini are correct for every protein.

## 6. Understanding the output files

After successful execution, the directory should contain several new files.

### `complex_processed.gro`

This file contains coordinates prepared for use with the selected force field.

Unlike PDB coordinates, which are normally expressed in ångströms, `.gro` coordinates are expressed in **nanometers**.

### `complex.top`

This is the main system topology.

It references force-field definitions, molecular topologies, water and ion parameters, and the composition of the simulated system.

Inspect the end of the file using:

```bash
tail -n 20 complex.top
```

The `[ molecules ]` section lists molecule types and counts.

### Protein `.itp` files

For a complex containing multiple protein chains, `pdb2gmx` may generate separate topology definitions for individual molecular species.

An `.itp` file may contain atom types, bonds, angles, dihedrals, and other molecular parameters.

### `posre.itp` files

Position-restraint files specify harmonic restraints that can be applied during equilibration.

These restraints are only active when the appropriate topology preprocessor definition is enabled, such as:

```ini
define = -DPOSRES
```

A typical equilibration protocol applies restraints to protein heavy atoms while water and ions adjust around the proteins.

## 7. Inspecting the generated topology

Check that the expected files were created:

```bash
ls -lh *.gro *.top *.itp
```

Inspect the beginning of the topology:

```bash
head -n 30 complex.top
```

Inspect the molecule list:

```bash
tail -n 20 complex.top
```

Pay attention to the **order of molecules**.

GROMACS atom ordering follows the processed topology and coordinate files. Do not assume that the original PDB chain identifiers or residue numbering will always be preserved in the same form.

## 8. Common errors

| Problem | Possible explanation |
|---|---|
| Missing atoms | The input structure may contain incomplete residues |
| Unknown residue | A nonstandard residue may require additional parameters |
| Incorrect chain separation | Chain identifiers or `TER` records may be missing |
| Unexpected termini | The input may have been divided into unintended molecules |
| Atom-count mismatch | The topology and coordinate files may not describe the same system |

Always investigate topology-generation errors rather than bypassing them.


## 9. Next chapter

**Chapter 3: Defining the Simulation Box, Adding Water, and Introducing Ions**

We will use `editconf`, `solvate`, `grompp`, and `genion` to create a solvated and electrically neutral simulation system.

## References

1. Lemkul, J. A. (2024). *Introductory Tutorials for Simulating Protein Dynamics with GROMACS.* Journal of Physical Chemistry B, 128, 9418–9435. https://doi.org/10.1021/acs.jpcb.4c04901
2. [GROMACS Documentation](https://manual.gromacs.org/current/)
3. [AMBER Force Fields](https://ambermd.org/)
