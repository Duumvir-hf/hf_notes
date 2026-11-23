# Calling the game's graphics correctly

[Index](README.md) · [Source catalog](sources.md) · [Native GUI](native-gui.md)

Scope: Steam 1.163, preferred base `0x140000000`. These are source-checked
recipes and inherited native evidence, not a new ABI audit.

## Choose the smallest appropriate route

| Need | Route | Starting example |
|---|---|---|
| Native-looking clickable button / text that participates in GUI layering | Native `Gui` object | `ally/src/shipworks.rs::create_control` |
| Fixed label over an existing screen | Text inside an established render pass | `precombat.rs::draw_marker_text`, QoL `elevation.rs::draw_hint` |
| Markers following battle objects | Battle HUD post-pass and native projection | `ally_reticles.rs`, `debug_overlay.rs` |
| Ship schematic/health preview | Private render targets and native panel renderer | `ally_panels.rs` — advanced, not a first widget |
| Diagnostic overlay without a native draw hook | Existing Win32 overlay-window implementation | QoL `elevation.rs::overlay_thread` — separate OS window, not native GUI |

Do not start with a D3D11/Present hook or a replacement GUI framework just to
draw a button. Existing game primitives already give the correct fonts, assets
and input behaviour. A Win32 overlay does not automatically work in every
fullscreen/Wine/capture setup or inherit the native GUI's stacking and hit tests.

## Hook at the correct point

| Purpose | Site → original target | Current caveat |
|---|---|---|
| Editor text | `0x140303fb4` → label pass `0x1400d4380` | In-game text/sprite evidence; below later palette buttons |
| Pre-combat PLAYER/loadout text | B23 `0x140309408` → flush `0x140083b50` | Current ally choice; gate by live screen, roller covers and Esc |
| Battle reticles/debug shapes | `0x14029a8c1` → native box pass `0x14029c5c0` | Shared within ally; battle HUD/camera context |
| Strategic map overlay | M11 `0x14030dae4` → flush `0x140083b50` | Probe-confirmed clean, including dialogue frames |
| Ship preview generation | `0x140309ab4` → panel `0x1402fde80` | Ally extra panels run before the original panel call |
| Ship preview composite | B21 `0x14030915f` → flush `0x140083b50` | Ally composites before original flush; not a universal overlay seam |
| GUI mutation/input service | `0x14018dbf0` input-task entry | Game-thread callback; conflict with other input hooks |
| Native clicks | `0x14012f120` ClickFunc entry | Single hook owner; consume only owned names |

**Order is part of the contract.** The ordinary text post-flush recipe is
`original call → mod draw → continuation`. The ship-panel composite is
`mod draw → original flush → continuation`. Do not substitute one for the other.
Drawing after the entire editor renderer returns ran but produced no visible
output in the original experiment.

Do not choose B24 (`0x140309897`) as an always-running seam: it is skipped with
GLITCH OFF. B22 can put pre-combat text under sliding doors. A renderer branch
is not a screen identifier: the battle branch also handles non-combat screens.
Menu seam candidates remain weaker evidence; see [overview](frame-overview.md).

### Hook safety

- Check supported game version, mapped image/base and exact original bytes.
  A mismatch can mean another mod owns the site: refuse, do not overwrite it.
- For an `E8 rel32`, call its resolved original target explicitly. Do not blindly
  replay the displaced relative instruction from trampoline memory.
- Preserve live GPR/XMM state, shadow space, stack alignment and required return
  registers. Current entry hooks pad RSP by eight before `preserved_call`;
  mid-function call sites have their own alignment. Do not generalize one layout.
- Run drawing synchronously inside the active native render context. Do GUI
  creation, sprite-list mutation, resource loading and screen repair in an
  appropriate game-thread builder/input callback, not in arbitrary worker threads.
- Never unwind into native code. `catch_unwind` is not protection against an
  access violation, does not undo native writes, and cannot catch aborting panics.
- `plausible(pointer)` and readable pages do not prove object identity, lifetime,
  exclusivity or the right thread. Preserve current membership and traversal guards.

## Coordinate spaces

| Space | Convention |
|---|---|
| Native top-level GUI | Centre-origin virtual frame; +x right, +y down |
| Ordinary screen text / projected HUD | Top-left-origin virtual frame |
| World | Project through native camera in the relevant render pass |
| OS cursor/client pixels | Not automatically the game's virtual cursor |

For the existing 1920×1080 recipe, top-level GUI → text is
`[x + 960, y + 540]`. Bitmap labels use rounded coordinates to avoid blur.
Do not apply this conversion to already projected points or parent-relative
child positions. Hit-box dimensions are not sprite dimensions.

The 1920×1080 mapping was originally checked at 2560×1440, 16:9; it is not proof
of ultrawide, 4:3, DPI or letterbox behaviour. `hf_tacnet::cursor_world` contains
an experimental client-to-world inverse, but uses the foreground window and
simple per-axis scaling. Do not ship it as a general cursor helper without
verifying game-window identity, viewport, focus, clipping and battle relative aim.

## Text: lookup, style, draw, restore

Native addresses:

| Operation | Address |
|---|---|
| Media resource-manager pointer | global `0x143a119e8` |
| `GetFont(manager, name)` | `0x1401703e0` |
| Style by name `(name, f32 scale, i32 blend, u32 ARGB)` | `0x1401a8710` |
| Modified font render | `0x14016eb40` |
| Native font colour / blend setters | `0x14016f120` / `0x14016f0c0` |

Recipe from `precombat.rs::draw_marker_text`:

1. Check screen gates, manager and returned font before use.
2. Prepare a bounded, NUL-terminated string. Plain sampler fonts have ASCII
   glyphs; `@` and newline are line breaks. Sanitize user text according to whether
   line breaks are intended. Native control/localized text is a different path:
   preserve its game encoding rather than blindly UTF-8/ASCII-converting it.
3. Save font scale `+0x100c`, colour `+0x1020`, blend `+0x1028`.
4. Apply the desired style (blend 2 in the ordinary label recipes).
5. Render `(font, x, y, align, text, out_width, out_height, extra, angle)`;
   existing labels pass null outputs, zero extra and `0.0` angle.
   Align 0/1/2 is left/right/centre in this path; y is line top.
6. Restore the shared font on **every** exit after styling.

**ABI gap:** current ally declares the extra parameter as `i32` and return as
`f32`; draw-test, elevation and tacnet declare `u64`/`i32`. They pass zero and
ignore the result. That common usage is evidence for drawing, not for a stable
measurement/rotation API. Verify native callers before standardizing the
disputed types, using nonzero extras, or consuming the return value.

**Restoration is more than saving three numbers.** The style helper skips colour
assignment when colour is zero; it is not an exact zero-colour restore. Font
colour also exists on glyph quads. Raw-writing only `font+0x1020` leaves glyphs
unchanged. See `perf/src/font_color.rs` and the native colour setter. Restore
through the verified setters, or explicitly restore all affected state.

Current implementations are examples, not audited reusable guards:
`debug_overlay::draw` styles the font before acquiring `CIRCLE_RADII.lock()` and
returns without restoration if that lock is poisoned. The tacnet status helper
raw-restores font fields without glyph colours. A library should close these
gaps before extraction; this task does not change those implementations.

## Sprites: borrow resources, preserve state

| Operation | Address |
|---|---|
| `GetAnimation(manager, name)` | `0x140170230` |
| `Render(sprite, x, y)` | `0x1401725c0` |
| `RenderEx(sprite, x, y, rotation, scale_x, scale_y)` | `0x140172aa0` |
| `SetColor(sprite, ARGB, corner)`; `corner = -1` means all | `0x140172ae0` |
| `SetZ(sprite, z, corner)` | `0x140172ee0` |

Getters return shared resources, not owned sprites. Animation `.res` definitions
give atlas, rect, hotspot, frames and resource group. Use the declared hotspot;
not every resource is centre-anchored. For a white frame, tinting works well;
for coloured art, tint is modulation, not a replacement texture.

Known modified sprite state: corner colours `+0x18/+0x70/+0xc8/+0x120`,
blend `+0x168`, scale `+0x16c`, dimensions `+0x188/+0x18c`,
hotspot `+0x198/+0x19c`. Restore every field/corner you change, plus Z if changed.
The current ally reticles deliberately leave style set because the native pass
restyles per draw; that local assumption is not a general library guarantee.
`RenderEx(..., scale_x, 0.0)` uses matching Y scale in existing native-box code.

Assets: [image reference](images-and-sprites.md), [font inventory](fonts.md),
[HGE formats and slots](../docs/engine-hge.md). Stock `hge_src` explains algorithms,
but HighFleet's expanded x64/D3D11 layouts and vtable slots are not stock ABI.
