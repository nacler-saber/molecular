---
name: pymol-figure
description: Create publication-ready PyMOL protein–ligand figures. Use whenever the user asks for PyMOL overall or local structure views, transparent protein surfaces, pocket sticks, ligand coloring, hydrogen-bond or contact annotations, residue labels, label placement, or ray-traced image export. Covers cases where PyMOL fails to identify a hydrogen bond automatically and the interaction must be drawn from explicitly named atoms.
---

# PyMOL protein–ligand figure workflow

Use this skill to produce a complete PyMOL command block for protein–ligand figures. Separate the overall figure from the local interaction figure so that local display changes are deliberate and reproducible.

## Defaults and required checks

Use these defaults unless the user specifies otherwise:

- protein object: `protein`
- ligand object: `ligand`
- protein chain: `A`
- background: white
- protein color: `gray70`
- protein surface: white, transparency `0.5`
- ligand: sticks, element-based coloring
- pocket radius: `4.0` Å
- pocket representation: sticks, transparency `0.5`
- hydrogen-bond dashes: red, width `2.0`, dash length `0.35`, gap `0.30`
- distance labels: red, two decimal places, size `16`

Before writing final selections, verify the object and atom names. PyMOL commands such as `get_names objects` are not valid at the command prompt; use either the object panel or:

```pymol
python
from pymol import cmd
print(cmd.get_names("objects"))
python end
```

Inspect atoms with:

```pymol
iterate ligand, print(model, chain, resi, resn, name, elem)
iterate protein and chain A and resi 275, print(model, chain, resi, resn, name, elem)
```

Use `count_atoms` after every critical selection. A hydrogen-bond endpoint selection should normally contain exactly one atom:

```pymol
count_atoms prot_hbond
count_atoms lig_hbond
```

Do not assume a LigPlot label such as `O1` is the PyMOL atom name. LigPlot may number equivalent atoms for display while the loaded structure names all oxygen atoms `O`. In that case select ligand oxygen atoms with `elem O` and resolve the correct one by distance and inspection.

## Overall figure

The standard overall view is:

```pymol
bg_color white

show cartoon, protein
color gray70, protein
set cartoon_transparency, 0, protein

show surface, protein
set surface_color, white, protein
set transparency, 0.5, protein

show sticks, ligand
util.cba ligand
set stick_radius, 0.20, ligand
```

`util.cba ligand` preserves recognizable element colors. If a unified ligand accent color is requested, define and apply one explicitly, for example:

```pymol
set_color ligand_accent, [0.12, 0.48, 0.62]
color ligand_accent, ligand
```

## Local pocket figure

To keep the rest of the protein in its overall representation while changing only the nearby residues:

```pymol
select pocket, byres (protein within 4.0 of ligand)
hide cartoon, pocket
hide surface, pocket
show sticks, pocket
color gray70, pocket
set stick_radius, 0.16, pocket
set stick_transparency, 0.5, pocket
```

The `byres` selector is important: it displays complete residues rather than isolated atoms. Keep the ligand visible as sticks. If the user means that the entire protein chain, rather than only the pocket sticks, should be transparent, apply the appropriate setting to that representation explicitly and explain the effect; do not silently alter the cartoon and surface together.

## Automatic hydrogen-bond attempt

PyMOL may identify donor/acceptor contacts when chemistry and hydrogens are available, but the result is not guaranteed to match LigPlot. A basic attempt is:

```pymol
h_add protein
h_add ligand
hide sticks, (protein or ligand) and elem H

distance hb_auto_prot, (pocket and donor), (ligand and acceptor), 3.6, 2
distance hb_auto_lig, (ligand and donor), (pocket and acceptor), 3.6, 2
```

Use this only when the user wants automatic detection. Do not present it as proof that all hydrogen bonds were found. If PyMOL shows no line while an external diagram shows one, use explicit atom selections.

## Manual hydrogen bonds

When the interacting residues and atoms are known, create one distance object per interaction. This avoids the common problem where a broad selection causes only one contact to be displayed or connects the wrong ligand oxygen.

Example for one interaction, protein `A/His275/NE2` and a ligand oxygen:

```pymol
select prot_his275, protein and chain A and resi 275 and name NE2
select lig_his275, (ligand and elem O) within 3.1 of prot_his275
count_atoms prot_his275
count_atoms lig_his275

delete hb_his275
distance hb_his275, prot_his275, lig_his275, 3.6, 0
```

For three interactions, use separate endpoint selections and separate distance objects. For example, for ASP302 `O`, GLN306 `N`, and GLN306 `O`:

```pymol
select prot_hb1, protein and chain A and resi 302 and name O
select prot_hb2, protein and chain A and resi 306 and name N
select prot_hb3, protein and chain A and resi 306 and name O

select lig_hb1, (ligand and elem O) within 3.2 of prot_hb1
select lig_hb2, (ligand and elem O) within 3.2 of prot_hb2
select lig_hb3, (ligand and elem O) within 3.2 of prot_hb3

count_atoms prot_hb1
count_atoms lig_hb1
count_atoms prot_hb2
count_atoms lig_hb2
count_atoms prot_hb3
count_atoms lig_hb3

delete hb_asp302
delete hb_gln306_N
delete hb_gln306_O

distance hb_asp302, prot_hb1, lig_hb1, 3.6, 0
distance hb_gln306_N, prot_hb2, lig_hb2, 3.6, 0
distance hb_gln306_O, prot_hb3, lig_hb3, 3.6, 0
```

The `within` cutoff is only a way to identify the matching ligand atom. It is not the displayed hydrogen-bond distance. The final `distance` command calculates and labels the actual coordinate distance.

If a ligand endpoint selection contains more than one atom, lower the cutoff or inspect candidate distances:

```pymol
select lig_oxygens, ligand and elem O
count_atoms lig_oxygens
distance candidate_contacts, prot_hb1, lig_oxygens, 4.0, 0
```

Keep the oxygen giving the expected contact distance, then make a narrower selection. If atom names are duplicated, identify the atom using `iterate` output and, when necessary, use its object, residue, name, and index selection.

If the protein atom is a side-chain oxygen or nitrogen, use the actual PDB atom name (`OD1`, `OD2`, `OE1`, `OE2`, `ND1`, `ND2`, `NE2`, etc.) rather than the generic residue atom shown by a diagram.

## Red dash style and distance labels

Apply the same style to every distance object:

```pymol
set dash_color, red, hb_asp302
set dash_color, red, hb_gln306_N
set dash_color, red, hb_gln306_O

set dash_width, 2.0
set dash_length, 0.35
set dash_gap, 0.30
set dash_round_ends, on

set label_color, red
set label_size, 16
set label_distance_digits, 2
set label_bg_color, white
set label_bg_transparency, 0.25

show labels, hb_asp302
show labels, hb_gln306_N
show labels, hb_gln306_O
```

For one interaction, replace the object name with `hb_his275`. Larger `dash_gap` makes the points more separated; larger `dash_length` makes each red segment longer. Keep these settings together so all hydrogen bonds remain visually consistent.

To move all distance labels together, use the same object-specific `label_position` offset on each distance object:

```pymol
set label_position, (0.0, 0.0, 1.2), hb_asp302
set label_position, (0.0, 0.0, 1.2), hb_gln306_N
set label_position, (0.0, 0.0, 1.2), hb_gln306_O
```

The tuple is an offset in model coordinates, not screen pixels. If labels still overlap the dashes, increase the third component or use a shared offset such as `(0.8, 0.5, 1.0)`. If labels need independent separation, set a different tuple for each distance object.

## Residue labels

Label only the residues involved in the displayed interactions. For a single His275 contact:

```pymol
select his275_label, protein and chain A and resi 275 and name NE2
count_atoms his275_label
label his275_label, "His275"
set label_color, black, his275_label
set label_size, 16, his275_label
set label_position, (0.0, 0.0, 1.2), his275_label
set label_bg_color, white, his275_label
set label_bg_transparency, 0.15, his275_label
show labels, his275_label
```

For a selected residue atom, generate a residue label from the structure:

```pymol
label prot_hb1, "%s%s" % (resn, resi)
set label_color, black, prot_hb1
set label_size, 16, prot_hb1
show labels, prot_hb1
```

If two interactions use the same residue, avoid duplicate residue labels by labeling a single representative atom or hiding one duplicate label. Do not mix labels from separate figures: keep yesterday's `his275_label` object separate from today's `prot_hb1`, `prot_hb2`, and `prot_hb3` selections.

## Ray-traced rendering

Interactive labels can disappear from a ray-traced image unless label tracing is enabled. Before rendering:

```pymol
set ray_trace_labels, on
show labels, all
set ray_opaque_background, on
```

Render and export at the intended resolution:

```pymol
ray 1800, 1400
png local_hbond.png, dpi=300
```

Or render directly during export:

```pymol
png local_hbond.png, 1800, 1400, dpi=300, ray=1
```

If a label appears interactively but not in the image, check `ray_trace_labels`, confirm the label selection still exists with `count_atoms`, and run `show labels, <selection>` before rendering. Keep the final camera orientation fixed before ray tracing.

## Response pattern

When the user supplies exact residue and atom names, return one complete executable command block. Preserve the user's object names and chain. Include `count_atoms` checks before drawing manual contacts. If an atom mapping is uncertain, state exactly which selection is unresolved and provide the `iterate` or candidate-distance command needed to resolve it; do not guess which duplicated ligand atom corresponds to a diagram label.
