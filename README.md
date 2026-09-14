# ACBN0 in FHI-aims: brief user guide

## 1. Scope

This development version of FHI-aims evaluates the ACBN0 direct Coulomb
interaction, `U_bar`, and Hund exchange interaction, `J_bar`, during the
self-consistent-field (SCF) calculation. The effective correction passed to
the DFT+U Hamiltonian is

`U_eff = U_bar - J_bar`.

For an s shell, for which the exchange denominator vanishes, the code uses
`U_eff = U_bar`.

The implementation supports the FHI-aims fully localized limit (FLL),
around-mean-field (AMF), and Petukhov-interpolated double-counting treatments.
The tested and recommended projector for ACBN0 is the Lowdin-orthogonalized
projector. Hubbard parameters are evaluated separately for every corrected
atom and shell, including chemically inequivalent atoms of the same species.

## 2. Building the code

Build the development branch using the normal FHI-aims CMake procedure for
the target machine. No additional ACBN0-specific CMake option is required.

Record the branch name and exact commit hash together with calculated data.

## 3. Minimal input

Add the following global keywords to `control.in`:

```text
plus_u_acbn0
plus_u_use_lowdin
```

`plus_u_acbn0` also activates the RI-LVL infrastructure used to evaluate the
on-site electron-repulsion integrals; no separate RI-LVL input keyword is
required for this purpose.

For every chemical species and shell to which ACBN0 is applied, add a
`plus_u` line inside the corresponding species block:

```text
plus_u  3  d  0.0
```

The syntax is

```text
plus_u  n  l  U_initial_in_eV
```

The final value is calculated by ACBN0; `0.0` is the usual initial value and
does not prescribe the converged Hubbard parameter. More than one shell can
be selected for a species by adding more `plus_u` lines, for example:

```text
plus_u  3  d  0.0
plus_u  4  s  0.0
```

A compact spin-polarized example is:

```text
xc pbe
spin collinear

plus_u_acbn0
plus_u_use_lowdin
plus_u_petukhov_mixing 1.0
ACBN0_details

# In the relevant species block:
plus_u  3  d  0.0
acbn0_acc 1.0e-4
```

All ordinary FHI-aims settings, including the k-point grid, occupation
broadening, SCF thresholds, initial magnetic moments, basis functions, and
integration grids, must also be supplied as appropriate for the system.

## 4. Selecting the double-counting treatment

The global keyword `plus_u_petukhov_mixing` controls the interpolation between
AMF and FLL, accordingly to the released FHI-aims version:

```text
plus_u_petukhov_mixing 0.0   # AMF
plus_u_petukhov_mixing 1.0   # FLL
```

A fixed value between zero and one requests the corresponding fixed linear
interpolation. If the keyword is omitted, the Petukhov mixing factor is
recalculated from the occupation matrix for each correlated subspace.

The same choice is applied throughout one calculation. For energy
differences, use the same double-counting treatment for every structure and
molecular reference.

## 5. Optional convergence and diagnostic settings

### Convergence of the Hubbard correction

The species keyword

```text
acbn0_acc 1.0e-4
```

sets the requested convergence threshold for the change in `U_eff`, in eV.
It should be stated explicitly for every corrected species when this optional
criterion is used. After all corrected shells meet their thresholds, further
ACBN0 updates are stopped and the SCF cycle continues with the converged
Hubbard parameters.

If `acbn0_acc` is omitted, the interactions continue to be updated as part of
the ordinary SCF cycle. In either case, verify that both the electronic SCF
criteria and the Hubbard parameters are converged.

### Detailed output

The global keyword

```text
ACBN0_details
```

prints, for each atom and corrected shell, the numerators and denominators of
`U_bar` and `J_bar`, their values, `U_eff`, and the change in `U_eff` between
updates. This option is especially useful for identifying atom-resolved
parameters and diagnosing oscillations or anomalously small denominators.

## 6. What to check in the output

A correctly initialized calculation should echo messages equivalent to:

```text
Using ACBN0 method for Hubbard U calculations
DFT+U occupation matrix is now calculated using the Lowdin orthogonalization
```

During the SCF cycle, check for:

```text
+U was obtained with ACBN0 successfully
correlated subspace ... | U = ... eV
petukhov mixing factor  : ...
```

With `ACBN0_details`, also inspect the atom index, shell, `U_bar`, `J_bar`,
`U_eff`, and `Change in U_eff`. At the end, confirm:

```text
Self-consistency cycle converged.
```

The number of correlated subspaces should agree with the number of corrected
atom-shell combinations. 

Warnings about a zero or very small denominator require inspection. A zero
exchange denominator is expected for an s shell, where the implementation
uses `U_eff = U_bar`; analogous warnings for other shells may indicate an
empty, full, one-electron, or numerically unstable corrected subspace.

## 7. Practical recommendations

1. Start from a well-converged geometry and physically reasonable magnetic
   initialization. Competing spin states can lead to different self-consistent
   solutions.
2. Use `plus_u_use_lowdin` for the calculations described with this
   implementation. The default on-site and Mulliken projectors can produce
   numerical instability.
3. Set the projector, parent exchange-correlation functional,
   double-counting treatment, basis settings, integration grids, initial spin
   state, and SCF thresholds. The self-consistent Hubbard parameters depend on
   these choices.
4. A smaller basis can assign more electronic weight to the selected Hubbard 
   subspace and thereby produce a stronger correction; this does not imply
   better numerical convergence.
5. For adsorption or reaction energies, apply a consistent ACBN0 setup to the
   clean surface, every adsorbate-covered structure, and all molecular
   references. Select the physically relevant corrected shells in each
   component of the energy cycle.

## 8. Present limitations

The current implementation is intended primarily for self-consistent
electronic-structure and total-energy calculations. Analytical derivatives of
the complete ACBN0 energy with respect to atomic positions and lattice vectors
are not yet implemented. Geometry optimization with continuously updated
ACBN0 interactions is therefore not fully self-consistent. Use geometries
relaxed with another chosen electronic-structure method and perform ACBN0
single-point calculations unless the limitations of the force path are
explicitly acceptable for the intended test.

The implementation reuses the existing FHI-aims parallelization, but does not
add a separate distribution of all ACBN0-specific work over corrected shells
and Kohn-Sham states. Large unit cells may therefore show less favorable
scaling than the per-iteration timings obtained for small bulk systems.

## 9. Reproducibility checklist

Archive the following with each reported calculation:

- FHI-aims development branch and commit hash;
- complete `control.in` and `geometry.in`;
- species defaults, including every `plus_u` and `acbn0_acc` line;
- projector and double-counting choice;
- parent functional and complete numerical settings;
- initial magnetic moments and final magnetic state;
- final output containing the converged total energy and atom-resolved
  Hubbard parameters.

## 10. Main source files

For code review and further development, the principal implementation files
are:

- `src/acbn0_ERI_calc.f90`: construction and storage of the on-site
  electron-repulsion integrals;
- `src/acbn0_renorm_density_matrix.f90`: projected occupations and
  renormalized density matrices;
- `src/plus_u.f90`: evaluation of `U_bar`, `J_bar`, and `U_eff`, plus the
  next-iteration update and convergence test;
- `src/read_control.f90`, `src/read_species_data.f90`, and
  `src/dimensions.f90`: input parsing and allocation flags;
- `src/update_density_densmat.f90`, `src/density_matrix_evaluation.f90`, and
  `src/scf_solver.f90`: integration into the SCF cycle;
- `src/reinitialize_scf.f90`: reconstruction of ACBN0 data when the SCF
  calculation is reinitialized;
- `src/CMakeLists.txt`: inclusion of the ACBN0 modules in the build.
