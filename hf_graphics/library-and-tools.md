# Can graphics/GUI modding become a library?

[Index](README.md) · [Implementation sources](sources.md)

**Yes.** Existing code is enough to build a small, native-looking Rust helper
library for Steam 1.163. It is not yet a general safe UI toolkit. This page is a
proposal, **not** implemented code or permission to refactor the mods.

## Recommended first scope

Provisional crate name: **`hf_graphics`**. Keep it a separately versioned crate,
not a new workspace spanning the independent repositories.

| Layer | Responsibility | Starting source |
|---|---|---|
| `native_1_163` | Reviewed address table, signatures, image/site verification, thin unsafe calls; layouts from pinned highfleet-rs | `native_layout.rs`, `precombat.rs` native types/constants |
| `text` / `sprite` | Bounded strings, explicit style and alignment, complete shared-state restoration | Pre-combat labels, overlay `LineBuf`, native font colour setter |
| `projection` | Distinct world/GUI/frame coordinate types, battle-context projection, finite checks | Reticles/overlay |
| `gui` | Complete registry snapshots, namespaced handles, native construction/update/load/hide, one-sprite repaint | Pre-combat core plus Shipworks template recipe |
| `input` | Route owned native click events; per-control debounce | Existing single click dispatcher |
| Host adapter | Own native input/render/click hooks, provide callback contexts and scene gates | Existing ally installers |

Do **not** put fleet roster, AI, ammo funding, ballistics, save serialization or
campaign rules into the graphics library. Buttons report actions; the mod
decides what those actions mean.

### API direction, not code that exists

```text
inside a verified game-thread GUI callback:
    snapshot = ui.scan_registry()
    if snapshot.incomplete: skip updates
    button = ui.ensure_button(snapshot, "MYMOD_ADD", style_from_native_test)
    button.set_text("ADD")
    button.place(GuiPoint(x, y), GuiSize(width, height))

inside an owned drawing callback:
    frame.with_font("flash_large_shadow", style, draw_label)
    screen_point = frame.project_battle(WorldPoint(x, y))
    frame.circle(screen_point, pixel_radius, colour)

inside the shared native click adapter:
    dispatch_owned_identifier(event) -> handled_or_pass_through
```

Raw constructors and dereferences stay behind an explicitly unsafe integration
boundary. Callback-local contexts can prevent accidental cross-thread or
cross-scene use by ordinary callers; a runtime native pointer cannot be made safe
merely by wrapping it in a Rust type.

Persistent handles should hold a namespaced identifier and cached hint/epoch,
not expose a permanently live `&mut Gui`. A complete fresh snapshot must
revalidate identity, state, registration and root ownership before returning a
callback-local operation. Keep contexts/handles non-Send/non-Sync unless a
specific thread-safe contract has actually been established.

Use scoped state restoration, but do not promise recovery from access violations,
abort, device loss or arbitrary native re-entrancy. Preserve the validated one-
sprite lifecycle, rather than adding repair heuristics to hide invalid state.

## What a Rust crate alone cannot solve

Every DLL statically linking this crate still has its **own globals**. A local
“hook registry” in the crate is not shared between DLLs.

Concrete conflicts already exist:

- ally and tacnet guest both patch `0x14029a8c1`;
- ally and probe can both patch ClickFunc `0x14012f120`;
- QoL elevation, draw-test and probe can use editor seam `0x140303fb4`;
- ally and voice can want input entry `0x14018dbf0`.

**Stage 1:** leave hook ownership with the host mod. The library accepts
callbacks from that host; byte-check and refuse conflicting installations.
This provides reuse within a DLL without pretending cross-mod composition works.

**Stage 2, if multiple independent GUI mods require it:** a single graphics
runtime DLL or Modloader-provided service owns those hooks and dispatches
registered callbacks. That is a separate, larger project.

A runtime boundary needs a versioned C ABI, deterministic callback/layer order,
explicit callback registration/unregistration, host/version discovery, load-order
handling and unload policy. Pass plain data/borrowed callback-local views with
specified lifetimes; do not pass Rust trait objects, `String`, `Vec`, unwinding
or ownership across independently built DLLs. This would be a good candidate
for coordination with logdot's Modloader rather than another competing patcher.

## Audit blockers before extraction

1. **Font ABI:** reconcile the extra argument/return discrepancies; keep initial
   API limited to drawing with zero extra and no return-value dependency.
2. **State restoration:** exact zero-colour restore, glyph colours, individual
   sprite corners/blend/scale/Z, and early-return/lock-failure paths.
3. **Native GUI lifecycle:** review constructor/registration and native cleanup
   contracts; no Rust-owned copies of game strings, vectors or object allocations.
4. **Coordinates/input:** prove supported resolution/viewport mappings and native
   virtual-cursor behaviour before exposing generalized pointer input.
5. **Resources:** missing assets must fail quietly and diagnostically; document
   resource-manager lifetime and what is valid during screen rebuilds.
6. **Panels:** matching target/sprite release, partial-failure cleanup, device
   reset, temporary target-link/marker restoration and unload are unresolved.
   Keep previews out of the initial safe/high-level API.
7. **Upstream layouts:** review current highfleet-rs and pin a full reviewed SHA;
   do not automatically update the existing consumers.

No hot reload, generic skin/theme engine, every-screen builder, full D3D11
wrapper or ImGui integration is needed for the first version. Those would
multiply compatibility and input problems before the basic contracts are settled.

## Tools worth building

Inspect/reuse existing `hf_probe`, `hf_draw_test` and `hf_dev_tools` first.
Proposed standalone repetitive tools belong in `hf_dev_tools` with README entries.

| Priority | Proposed tool | Benefit / bounded scope |
|---|---|---|
| 1 | Offline `.res` / `.fnt` resource catalog | Resolve includes, names, atlas rects, hotspots, font metrics and missing references. Emit a searchable catalog/contact sheet from a supplied game directory; no injection or game writes |
| 2 | GUI dump normalizer/diff | Parse existing logs into identifier/parent/state/rect/resource records where available; compare before/after rebuilds. Report incomplete captures instead of inferring deletion |
| 3 | Dedicated graphics sandbox | Reuse the existing probe font sampler/seam test; show selected native fonts/sprites and a few controls. Explicitly select screens/hooks; never enable competing production hooks silently |
| 4 | Read-only native inspector | Extend existing GUI diagnostics to include parent, state, layer, sprite count/load state and completeness; start with logs before adding an interactive overlay |
| Later | Native-style layout preview | Fast anchors/spacing iteration from known assets and a virtual frame. Offline output is a design preview, not proof of native focus, input, animation or layering |

The best immediate combination is **a small library plus resource catalog**,
not a drag-and-drop editor first. The library reduces unsafe boilerplate;
the catalog answers “which native asset/font/style should I use?”.

## Incremental implementation plan

1. Agree initial consumers (new mod alone, or ally plus new mod) and supported
   game versions/screens. Review ownership/ABI gaps above.
2. Extract text/sprite/projection wrappers with scoped restoration and a host-owned
   hook adapter; keep existing native guards and make no gameplay changes.
3. Extract the GUI lifecycle using Shipworks and pre-combat as two **distinct**
   recipes sharing one core. Compare in-game behaviour before/after.
4. Build the offline catalog and deliberately small sandbox as separate tools.
5. Add panels or a cross-DLL runtime only when a concrete consumer requires them.

Cross-build the chosen consumers with their existing targets/features, run
existing relevant checks, and do the [manual validation pass](validation.md).
Ask for explicit permission before writing new regression tests. No dependency
updates, code extraction, new tooling, builds or deployments were done for this
knowledge-base task.
