# vek VM Reference

This document is the complete technical reference for the Reference vek Virtual Machine (VM) implementation.

## Table of Contents

1. [Overview](#overview)
2. [Constants](#constants)
3. [External Functions](#external-functions)

## Overview

The reference vek VM provides a standard execution environment fully compliant with the vek ISA specification.
This reference implementation includes built-in support for specialized instructions like `ECALL`, alongside a pre-defined set of native functions and mathematical constants.
It serves as an ideal runtime solution for language designers and developers who require an ISA-compliant execution engine without the overhead of building an ISA-compliant custom virtual machine from scratch.

## Constants

Constant values are defined within the reference vek VM.
They are treated as literal static values during the compilation or evaluation process.

| Names | Type |
|---|---|
| `NaN` | Float |
| `Inf` | Float |
| `E` | Float |
| `Pi`, `π` | Float |
| `Pi2` | Float |
| `Pi3` | Float |
| `Pi4` | Float |
| `Pi6` | Float |
| `Phi`, `Φ`, `φ`, `ϕ` | Float |
| `Sqrt2` | Float |
| `SqrtE` | Float |
| `SqrtPi` | Float |
| `SqrtPhi` | Float |
| `Ln2` | Float |
| `Log2E` | Float |
| `Ln10` | Float |
| `Log10E` | Float |
| `Euler`, `γ` | Float |
| `Gauss` | Float |
| `Gelfond` | Float |
| `Apery` | Float |
| `Khinchin` | Float |
| `Catalan` | Float |
| `Faraday` | Float |
| `Avogadro` | Float |
| `Chaitin` | Float |
| `Planck` | Float |
| `ReducedPlanck` | Float |
| `Rydberg` | Float |
| `Boltzmann` | Float |
| `StefanBoltzmann` | Float |
| `Conway` | Float |
| `Lehmer` | Float |
| `Ramanujan` | Float |
| `VonKlitzing` | Float |
| `ConventionalVonKlitzing` | Float |
| `Josephson` | Float |
| `GelfondSchneider` | Float |
| `SqrtGelfondSchneider` | Float |
| `FirstFeigenbaum` | Float |
| `SecondFeigenbaum` | Float |
| `InvSqrtPi2` | Float |
| `Erf1` | Float |
| `GaussianGravity` | Float |
| `NewtonianGravity` | Float |
| `AtomicMass` | Float |
| `ElectronMass` | Float |
| `ProtonMass` | Float |
| `ProtonElectronMassRatio` | Float |
| `BohrRadius` | Float |
| `PlanckLength` | Float |
| `PlanckTemperature` | Float |
| `PlanckTime` | Float |
| `PlanckMass` | Float |
| `BaselProblem` | Float |
| `FineStructure` | Float |
| `InvFineStructure` | Float |
| `Gas` | Float |
| `Magnetic` | Float |
| `Electric` | Float |
| `SpeedOfLight` | Float |
| `ElementaryCharge` | Float |
| `ElectronVolt` | Float |
| `MolarMass` | Float |
| `ConductanceQuantum` | Float |
| `ThomsonCrossSection` | Float |
| `HartreeEnergy` | Float |
| `BohrMagneton` | Float |
| `NuclearMagneton` | Float |
| `MagneticFluxQuantum` | Float |

## External Functions

External functions are defined within the reference vek VM and are accessible via the `ECALL` instruction.

| Name | Index | Parameters | Description |
|---|---|---|---|
| `_EXTERN_PRINT` | 0 | 1 | Prints paramer `1` to stdout with new-line |