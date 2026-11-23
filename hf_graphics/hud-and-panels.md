# World overlays and ship render targets

[Index](README.md) · [Calling guide](calling-guide.md)

Source-checked against current ally 1.2.4. This is not a new in-game or Ghidra pass.

## Battle overlays: the straightforward reusable part

`ally_reticles.rs` owns the call at `0x14029a8c1`. It calls native
`0x14029c5c0` first, then dispatches ally boxes and `debug_overlay::draw`.
Both features share the same hook even when only the debug overlay is enabled.
Check HUD byte `0x143a11961` before drawing.

Projection:

```text
point = [world_x, world_y]                 # mod-owned two-float scratch
0x1401adfc0(&point, absolute=0, zoom=1.0)   # in place, battle camera
0x1402e51c0(&point)                        # native pixel/grid floor
```

Use hull centre fields `Creature::bounds_center_x/y` when that is the intended
anchor. Native target boxes additionally use offsets `+0x240/+0x244`; copying
that anchor for every overlay is not necessarily appropriate.
Reject non-finite world/projected values and cull off-screen work.

- **Reticles:** shared `sign_aim` pointer at `0x14800eab0`, white 62×62 frame,
  hotspot 31×31. Project the target and use `RenderEx` with an ARGB tint.
  Current ally excludes the player-controlled ship and marks only hostile targets.
- **Circles:** `Gfx_DrawCircleClipped` at `0x140244b00`. Existing binding:
  `(centre*, radius, ARGB, u8 zoom_scale, u8 dashed, u32 style, i32 width,
  clip_min*, clip_max*)`. Current solid screen-space ring passes
  `0, 0, 3, 1` after colour. Use this tested recipe; those observed values are
  not a fully established enum set.
- **World-size radii:** project the centre and a point 100 world units away,
  then `pixels_per_world = projected_distance / 100`.
  Multiply the desired world radius by that scale and pass `zoom_scale=0`
  because radius is already screen units. Do not project/zoom twice.
- **Lines:** the earlier native reference identifies `0x140248ba0`
  (`Gfx_DrawWorldLine`), including an already-screen path.
  The current overlay uses repeated tiny circles for dotted waypoints;
  a full generalized line binding still needs caller/ABI review.
- **Labels:** use the shared-font recipe, bounded NUL buffers and explicit
  style restoration. Current overlay `LineBuf` avoids per-label allocations.

Current ring work is capped at eight configured radii and 256 ships, with
screen-radius limits 3..4000; these are mod policy, not engine capacities.
Keep traversal/capacity limits when adapting the implementation.
Ballistic lead solving belongs to the mod's logic, not a general graphics crate:
graphics should take a world point/label, not know weapon trajectory rules.

## Ship panels: possible, but keep experimental/advanced

`ally_panels.rs` reuses `DrawBattleShipPanel(surface_a, surface_b, mode)`
at `0x1402fde80`, mode 3 (native target view).
Current flow:

1. At the own-panel call `0x140309ab4`, render allies into private 350×300
   colour/depth targets, then run the original own-panel call.
2. At `0x14030915f`, composite their sprites at half width/height into the
   already prepared native panel layer, then run the original flush.
3. The composite uses the previously prepared frame, like the stock panels.
   Associate each texture with the ship actually rendered, not the next roster.
   Invalidate content/displayed identities on battle teardown or handoff.

HGE object global `0x143a119e0`: current code uses vtable byte slots `+0x300`
for colour target creation and `+0x308` for depth/stencil creation, 350×300,
colour format `0x1c`, target flag 1. It allocates a `0x1a8` low-level sprite
and calls constructor `0x140171d50`, then composites with blend `0x10`.
These version-specific slots are not interchangeable with stock HGE headers.

### Why a ship may show text but no hull

The native panel reads the per-frame HUD marker table, not just a Creature
pointer. Ships deployed outside stock spawn may lack native part registration.
Ally temporarily inserts type-2 child markers using the native layer/scale rules:

| Table | Address / shape |
|---|---|
| HUD markers | `0x143faa680`; 70 layers, 40000 entries/layer, stride `0x18` |
| Per-layer counts | `0x143a51160` |
| Marker payload | type at 0, Body pointer at 8, sprite index at `0x10`, scale at `0x14` |
| Panel-consumed layers | 1..39 |
| Render-body grid | `0x146449688`; 60 layers, 40000 entries/layer, stride `0x18` |
| Render-body counts | `0x143a51200` |

The mounted-part vector alone can omit hull bodies; ally consults the render
grid too, deduplicates parts and existing markers, and checks layer capacity
before appending. Stacking everything in one low layer produced dark/misordered
hulls. Preserve the native inherited-stage, floor and sprite-stage calculation,
not a guessed constant layer.

Rendering temporarily redirects the player's target **through native SetLink**,
not a raw `+0xb00` write; registration and saved IDs must remain consistent.
Restore the saved target and every touched layer count immediately after each
native panel call. This is mutation of game state inside a tightly bounded
render callback, not a general permission to modify objects while drawing.

### Not ready for a blanket safe library wrapper

- Current target/sprite allocations persist in process state. Allocation failure
  cleanup, destruction, device reset/recreation and DLL unload are not established
  by this code. Do not advertise a leak-free or hot-unloadable resource owner.
- Partial allocation failure must not silently leak resources in a future wrapper;
  review matching native release/destructor contracts first.
- Temporary native state needs restoration on all exits; Rust RAII cannot recover
  from process termination, access violations or an aborting panic.
- Preserve HUD/message/end-game suppression gates. Do not keep stale previews
  selectable when the native HUD hides them.
- Keep the advanced panel adapter separate from ordinary text/button APIs.

Full prior findings and screenshots/history:
[ship-panel reference](battle-ship-panels.md).

## Existing native recolour hooks

`ally_hud.rs` is a different approach: change the colour arguments of native
damage popups, their leader sprite and friendly stars, retaining native
placement/animation. It preserves module-line alpha while replacing RGB.
Use when only recolouring is required; do not rebuild the HUD unnecessarily.
Its register-specific patch sites are feature adapters, not generic draw calls.
Reference: [HUD callouts and palette](battle-hud-callouts.md).
