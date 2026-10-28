# Chapter 1: Preparing a Protein–Protein Complex for GROMACS

## 1. Introduction

Before starting a molecular dynamics (MD) simulation, the input protein structure must be carefully examined and prepared.

A successful GROMACS simulation depends not only on the simulation parameters but also on the quality of the initial structure.

In this chapter, we will learn how to inspect a protein–protein complex, identify structural problems, and prepare a PDB file for topology generation.

## 2. Learning objectives

By the end of this chapter, you will be able to:

- Understand the PDB file format.
- Identify protein chains and residue numbers.
- Inspect protein structures using UCSF ChimeraX.
- Detect missing atoms and steric clashes.
- Prepare a clean structure for GROMACS.

## 3. Required software

- [UCSF ChimeraX](https://www.rbvi.ucsf.edu/chimerax/)
- [GROMACS](https://www.gromacs.org/)
- A text editor
- Terminal (macOS or Linux)

We recommend using a publicly available protein complex from the [RCSB Protein Data Bank](https://www.rcsb.org/) for the tutorial exercises.

## 4. Understanding a PDB file

A PDB file contains information about the three-dimensional arrangement of atoms in a molecular structure.

Important records include:

| Record | Description |
|---|---|
| `ATOM` | Standard protein or nucleic acid atoms |
| `HETATM` | Nonstandard residues, ligands, ions, or water |
| `TER` | End of a polymer chain |
| `MODEL` | Beginning of a structural model |
| `ENDMDL` | End of a structural model |
| `END` | End of the coordinate file |

**Important:** A protein–protein complex normally contains two or more chains within the same structural model. Separate `MODEL` records do not automatically represent interacting protein chains.

## 5. Inspecting the structure with ChimeraX

Open UCSF ChimeraX and load your PDB file:

```text
open complex.pdb
```

Display the proteins as cartoons:

```text
cartoon
```

Color each chain separately:

```text
color bychain
```

Visually inspect the complex and identify the protein–protein interface.

Pay particular attention to:

- Unexpected gaps in the backbone.
- Missing side-chain atoms.
- Unusually close interatomic contacts.
- Incorrect chain assignments.
- Unresolved terminal regions.

## 6. Checking for steric clashes

A steric clash occurs when two atoms occupy an unrealistically close region of space.

Severe clashes can produce large forces during energy minimization and destabilize the beginning of an MD simulation.

ChimeraX provides a clash-detection command:

```text
clashes #1
```

Replace `#1` with the correct model identifier if needed.

Review the reported clashes rather than assuming that every close contact is physically impossible.

**Important:** Energy minimization can relieve local overlaps, but it cannot establish that a predicted protein–protein binding orientation is correct.

## 7. Preparing the structure for GROMACS

Before proceeding, verify that:

- Both protein chains are present in the intended complex.
- Chain boundaries are correctly defined.
- Missing heavy atoms have been investigated and repaired where necessary.
- Alternate conformations have been resolved consistently.
- Nonstandard residues and ligands have been evaluated.
- The intended protonation states and terminal groups are understood.
- Severe steric clashes have been addressed.

Save the prepared structure as:

```text
complex_prepared.pdb
```


## 8. Next chapter

**Chapter 2: Generating the Molecular Topology with `pdb2gmx`**

We will introduce molecular mechanics force fields, generate a GROMACS topology, add hydrogen atoms, and explain the files produced by `pdb2gmx`.

## References

1. Lemkul, J. A. (2024). *Introductory Tutorials for Simulating Protein Dynamics with GROMACS.* Journal of Physical Chemistry B, 128, 9418–9435. https://doi.org/10.1021/acs.jpcb.4c04901
2. [GROMACS Documentation](https://manual.gromacs.org/current/)
3. [UCSF ChimeraX](https://www.rbvi.ucsf.edu/chimerax/)
4. [RCSB Protein Data Bank](https://www.rcsb.org/)
