# .MD2/.MDX Model Tools for Blender

A Blender addon for importing and exporting **Quake II (.md2)** and **Kingpin (.mdx)** models, with extra tooling for working with vertex-animated meshes.

Tested on Blender 5.1. Should work on 4.4+ where the slotted-actions API is available; older versions (2.80–4.3) take a legacy code path that is still supported.

---

## Installing

1. Download or clone this repository.
2. Place the `kingpin/` folder inside your Blender addons directory:
   - **Linux:** `~/.config/blender/<version>/scripts/addons/kingpin/`
   - **Windows:** `%APPDATA%\Blender Foundation\Blender\<version>\scripts\addons\kingpin\`
   - **macOS:** `~/Library/Application Support/Blender/<version>/scripts/addons/`
3. In Blender, open `Edit > Preferences > Add-ons`, search for **MD2**, and tick the checkbox to enable.

If you used Steam to install Blender, the addon directory may live inside Steam's Proton prefix — check `bpy.utils.user_resource('SCRIPTS', path="addons")` in Blender's Python console to find the right path.

---

## Where things live

- **File > Import > Kingpin Models (md2, mdx)** — import a model.
- **File > Export > Kingpin Models (md2, mdx)** — export the selected mesh(es).
- **Properties editor → Scene tab → .MD2/.MDX Model Tools** — the modeling, animation, and conversion helpers (see below).

---

## Importing a model

`File > Import > Kingpin Models (md2, mdx)`.

Useful options in the import dialog:

- **Use Existing Material** — if a material with the same name as the model's skin already exists in the scene, reuse it instead of making a duplicate.
- **Store .pcx Internally** — Blender doesn't natively load `.pcx` textures, so the addon decodes them and packs them into the blend file. Disable if you'd rather skip pcx skins.
- **Skip Mesh Cleanup** — by default Blender validates imported mesh data and may strip faces it considers invalid. Tick this to keep the raw geometry.
- **Type** — how to bring in animation:
  - *Vertex Keys* — per-vertex F-curves. Most flexible, slowest to import.
  - *Shape Keys (absolute)* — one shape key per frame, driven by `eval_time`. Good middle ground.
  - *Shape Keys (relative)* — one shape key per frame with value keyframes. Old behaviour.
- **Import Frame Names** — turns frame name prefixes (e.g. `stand1`, `stand2`, `run1`…) into timeline markers.

Multi-select is supported: select multiple `.md2`/`.mdx` files in the dialog and they'll all import.

---

## Exporting a model

`File > Export > Kingpin Models (md2, mdx)`. Select the meshes you want to export first, then choose `.md2` or `.mdx` as the file extension.

Notes worth knowing:

- **Skin paths** are truncated to 63 characters (engine limit). If your texture is set up as a material with an image node, the addon will use the image path; otherwise the material name is used as the skin name.
- **Frame names** come from timeline markers. The frame range exported is the scene's `frame_start` to `frame_end`.
- **Hitbox** is generated automatically as a bounding box covering all selected meshes across all frames.
- **Custom vertex normals** can be exported as well — useful for player models with seams that need consistent normals across parts (see "Custom vertex normals" below).
- **HD precision (`.mdx5`)** uses 2 bytes per vertex coordinate instead of 1, eliminating compression wobble at the cost of file size.

---

## The Scene panel

Under `Properties > Scene > .MD2/.MDX Model Tools` you'll find five collapsible sub-panels:

### MD2 vertex grid

MD2/MDX compresses vertex positions into bytes, so vertices snap to a coarse grid and animate with visible jitter ("wobble"). This tool builds a reference grid in the scene at the exact resolution the exporter will quantize to. Snap your vertices to grid points and the wobble disappears.

Options: solid/wireframe display, floor-only mode, subdivision count.

### Smooth animation wobble

An alternative approach to the same problem: imports an existing vertex animation, runs a Gaussian smoothing pass over each vertex's position curve, writes the smoothed values back.

Set a start/end frame range and tick **Loop** if the animation is cyclic (idle, run). Then press **Smooth**. Only meaningful on HD (`.mdx5`) models — standard MD2 precision will be re-quantized on export anyway.

### Retarget animation

Drive a static mesh from an existing animated source. Useful for:

- Splitting an animated model into parts (head/body/legs) while keeping the original animation.
- Re-applying animation to a re-topologized or cleaned-up mesh.

Workflow:

1. Set the scene to a bind frame where the source and target meshes are aligned (usually frame 0, T-pose).
2. Duplicate the animated source, then on the duplicate: clear all animation, delete the geometry you don't want.
3. In the panel, set **Source** to the original animated mesh.
4. Set the start/end frame range.
5. Press **Animate Mesh**. For each vertex on the target, the addon finds the nearest source vertex and copies its animation.
6. Scrub the timeline to confirm. Remove any modifiers that might interfere.

You can also point Source at a collection if the animation spans multiple objects.

### Vertex keyframes

Manual editing tools for vertex-animated meshes — based on the AnimAll addon's approach.

Insert, delete, and clear keyframes for selected vertices (or all vertices). Switch interpolation type (linear, bezier, constant). Step through the timeline frame by frame. Works for both vertex-key animation and shape-key animation depending on what the active mesh uses.

### Quake 3 model converter

Convert a Quake 3 character (lower/upper/head + weapon) into Kingpin-compatible parts.

1. Import the Q3 `.md3` files using a `.md3` importer (e.g. the "Quake 3 Model" addon).
2. In this panel, set leg idle behaviour (static or animated), running rotation angles, crouch-death fallback, and a scale factor matching Kingpin player proportions.
3. Press **Animation.cfg** and select the matching `.cfg` from the Q3 model.
4. Hide any unrelated scene objects.
5. Press **Convert to Kingpin**.

Now you can export head/body/legs separately as `.md2` or `.mdx`. The `animation.cfg` often needs hand-tweaking to land the timings.

---

## Custom vertex normals (for seamed player models)

When a model is split into parts (head/body/legs), the seam between parts has duplicate vertices with different normals, which looks like a visible crease in-game. The fix:

1. Duplicate the model. Clear its animation. Merge all parts and weld the seam vertices.
2. Retarget the animation onto this merged mesh (use the Retarget panel in collection mode).
3. On the mesh you actually want to export, add a **Data Transfer** modifier.
4. Source = the merged/welded mesh. Use "Face Corner Data → Custom Normals". Make sure the source object has Auto Smooth enabled.
5. When exporting, tick **Custom vertex normals**.

The seam parts will now share identical normals along the seam and shade as one smooth mesh.

---

## File format limits

| Constant | MD2 | MDX | MDX5 (HD) |
|---|---|---|---|
| Max vertices | 2048 | 4096 | 4096 |
| Max triangles | 4096 | 8192 | 8192 |
| Max frames | 1024 | 1024 | 1024 |
| Max skins | 32 | 32 | 32 |
| Skin name length | 63 | 63 | 63 |
| Vertex precision | 1 byte/axis | 1 byte/axis | 2 bytes/axis |

---

## Troubleshooting

**Import fails with `struct.error: unpack requires a buffer of N bytes`** — the file is truncated, corrupt, or wasn't produced by a strictly-conformant exporter. Try opening it in another tool (MeshLab, the Autodesk FBX Converter) to confirm it's valid. If you can, re-export from there.

**Textures show as solid colour** — the skin path inside the file points somewhere the addon can't find. The addon searches the model's folder, then walks up to `models/`, `players/`, or `textures/` parent directories. If your texture is elsewhere, copy it next to the model and re-import, or set the material's image manually after import.

**`.pcx` skins don't appear** — make sure "Store .pcx Internally" was enabled on import. Otherwise Blender won't render the pcx and you'll need to convert it to PNG/TGA externally.

**The Scene panel is missing** — make sure the addon is actually enabled (`Preferences > Add-ons`, search "MD2", tick the box). The panel lives in the **Properties editor → Scene tab**, not the 3D viewport sidebar.

**Animation imports but plays at wrong speed** — Kingpin/Q2 models don't store a frame rate; the addon imports each frame as one scene frame. Adjust scene FPS to taste, or use timeline markers (which the addon imports from frame name prefixes) to find the loop points.

---

## Credits

This addon is descended from the original Quake 2 MD2 import/export work, extended for Kingpin's MDX format by HypoV8. Contributors over the years:

DarkRain · Bob Holcomb (MD2 normals, original exporter) · David Henry (MD2 format documentation) · Sebastian Lieberknecht · Dao Nguyen · Bernd Meyer · Damien Thebault · Erwan Mathieu · Takehiko Nawata · Daniel Salazar (AnimAll) · Patrick W. Crawford (2.7/2.8 support)

Tracker: <https://github.com/hypov8/blender_kingpin_models>
