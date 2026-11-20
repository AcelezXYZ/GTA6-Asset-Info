# GUI

*Part of the [SixNG tool guide](README.md). Numbers and timings as of 2026-11-20.*

```powershell
sixng-gui
```

The **Game** and **Work** fields take the same defaults as the CLI; use
**Browse…** to choose another install.

## Browser tab

1. The left panel lists the archives. Click **Index** on any that are not
   indexed yet.
2. The middle panel lists the selected archive's entries. Filter with a
   regex and the kind and type drop-downs. Tick rows, then **Extract
   selected** or **Extract all N filtered** into **Output**; the **PNG**
   and **DDS** boxes pick texture formats.
3. The right panel previews the entry: decoded textures for dictionaries,
   the JSON description for models, a text or hex view otherwise. **Open
   in model viewer** opens models.
4. The **World data** row runs the world steps (Placements, Batches,
   Collision, Road navmesh, Lane graph, Lights, Interiors, Surfaces) into
   **Output**. Run them left to right.

## Model viewer tab

Open a model by entry id, by name regex (`^phoenix_hi$`; prefix another
archive as `data:^w_pi`), from an exported `.glb`, or from the Browser.
Entries are exported on the fly into `<work>/gui_cache`.

- Shading: base colour, normal map (DirectX green), packed
  roughness/metal/AO with a channel-order switch, alpha / cutout / decal /
  glass classes, emissive.
- Car paint: primary, secondary, pearl colour and amount, finish (metallic,
  matte, chrome, gloss).
- **LOD** buttons; debug views (albedo, normals, UV checker, single texture
  slot, **BCPack gaps** green/amber/red, vertex colour, lighting only);
  wireframe and normal lines.
- Skeleton with joint axes and labels; **Rig** sliders spin the wheels and
  open the doors. Cars get a default shared rim and tyre at their wheel
  mounts (the real size is not decoded).
- Orbit or fly camera; time of day, exposure, ambient, emissive and a wet
  look.
- Per-material list with class, alpha mode, gap status and every texture
  slot; show, hide and solo.
- **Screenshot** saves the 3D view and the window under the work directory.

## Map 2D tab

First bake the map from the world exports:

```powershell
sixng-gui --bake-map            # full bake, ~5 min, ~1 GB in <work>/map
sixng-gui --bake-map raster     # rasters only, ~45 s, reuses the collision cache
```

The bake writes a raster tile pyramid (`tiles/<layer>/<z>/<x>_<y>.png`, 256 px,
2 m per pixel at the top level; layers: hill-shaded terrain, surface
categories, building footprints, navmesh, vegetation/prop density,
texture-gap heatmap, LOD regions, water / no collision) plus vector data
(`roads.json`, `junctions.json`, `lights.bin`, `entities.bin`,
`strings.json`, `interiors.json`, `heights_8m.bin`).

In the tab:

- pan and zoom; a base layer (terrain, surfaces or water) with overlays at
  their own opacity;
- the lane graph coloured by speed limit, lane count or road class, with
  one-way arrows;
- junctions cycling through their signal phases; stop and yield entrances;
- lights with exterior/interior toggles and an hour slider (each light's
  hour mask);
- interior outlines;
- **Inspect** (click a placement: model, archetype, LOD level, position in
  game and glTF axes, map file, lights) and **Open in model viewer**;
- **Search** by name, model id or archetype hash;
- **Measure**; **Box select** shows the `--bbox` for region exports;
- a readout of game x/y/z and the glTF/BeamNG position (x, z, −y).

## More coming

Planned: a 3D map view, per-target previews (BeamNG, Assetto Corsa),
day/night, a change report, and an audio browser.

## Command-line screenshots

For checks and docs:

```powershell
sixng-gui --model levels.rpf@10f266d010 --mode 5 --yaw 2.4 --pitch 0.25 --screenshot car.png --exit
sixng-gui --tab map2d --center -670,60 --mpp 2 --base 0 --overlays 2,3 --lights --hour 22 --screenshot map.png --exit
```

Model options: `--lod N`, `--mode 0-7` (shaded, albedo, normals, UV, slot,
gaps, vertex colour, lighting), `--slot 0-3`, `--rma 0-3`, `--wire`,
`--skeleton`, `--normals`, `--tod H`, `--paint r,g,b`, `--finish 0-3`,
`--pearl A`, `--wet A`, `--exposure E`, `--spin R`, `--door R`, `--yaw R`,
`--pitch R`, `--zoom F`, `--size WxH`. Map options: `--center X,Y`,
`--mpp M`, `--base N`, `--overlays a,b`, `--lights`, `--hour H`,
`--search name`.

---

← [CLI](02-cli.md) · [Index](README.md) · [Output layout and coordinates](04-output.md) →
