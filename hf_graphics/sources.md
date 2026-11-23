# Workspace graphics reference catalog

[Index](README.md) · [Calling guide](calling-guide.md)

Survey date: **2026-11-13**. Workspace root:
`/home/maksim/Development_Highfleet`. Searched source/documentation names and
graphics API terms across the workspace, excluding Git internals, build targets
and dependency caches from the broad scan. Read the principal implementations
below; this is a curated source inventory, **not** an exhaustive audit of every
binary, screenshot, shader or archived version.

## Primary implementation: highfleet-allies

Local source revision `d44dd49b1a237da8d29878ddf317fea08eace98f`, ally 1.2.4.
Portable repository: <https://github.com/mxtrnhf/highfleet-allies>.
Paths in this table are relative to that repository.

| Source | Useful entry points / lesson |
|---|---|
| [ally/src/shipworks.rs](../../highfleet-allies/ally/src/shipworks.rs) | `tick`, `create_control`, `show_control`, `native_string`, `hide_member`: actual native TEST styling, native text controls, read-only labels, dynamic text assignment |
| [ally/src/precombat.rs](../../highfleet-allies/ally/src/precombat.rs) | `install`, `scan_gui_list`, `get_or_make_with`, `allocate_gui`, `attach_gui`, `move_gui_rect`, `load_gui`, `paint_button`: reusable lifecycle core; most helpers are crate-private, not a public library |
| same file, screen logic | `follow_toggles`, `card_ships_from`, `gui_health_tick`, `hide`, `on_click_pre`: moving controls, slot-versus-ship identity, per-button debounce, complete/incomplete scans |
| same file, drawing | `draw_marker_text`, `draw_plane_labels`, `doors_closed`: font styling, GUI-to-frame conversion, B23, roller-cover/Esc gating |
| [ally/src/ally_reticles.rs](../../highfleet-allies/ally/src/ally_reticles.rs) | `install`, `draw_battle_overlay`, `draw_box`: one shared HUD post-pass, projection and shared frame sprite |
| [ally/src/debug_overlay.rs](../../highfleet-allies/ally/src/debug_overlay.rs) | `project`, `draw_ring`, `draw_rings`, `draw_waypoint`, `LineBuf`: world-size shapes, culling, bounded labels; do not extract ballistics/AI decisions into graphics |
| [ally/src/ally_panels.rs](../../highfleet-allies/ally/src/ally_panels.rs) | `ensure_panels`, `collect_panel_parts`, `insert_layer_markers`, `render_allies`, `composite_allies`, `invalidate`: advanced targets and ship previews |
| [ally/src/ally_hud.rs](../../highfleet-allies/ally/src/ally_hud.rs) | Colour-only native argument hooks, alpha preservation and shared sprite tinting |
| [ally/src/native_layout.rs](../../highfleet-allies/ally/src/native_layout.rs) | Pinned highfleet-rs type reuse, `offset_of!` / `size_of!` assertions, borrowed native ownership rules |
| [ally/src/native.rs](../../highfleet-allies/ally/src/native.rs), [diag.rs](../../highfleet-allies/ally/src/diag.rs), [bootstrap.rs](../../highfleet-allies/ally/src/bootstrap.rs) | Version/base/fingerprint setup, native pointer and readable-span guards; guards alone are not lifetime proof |
| [ally/src/sound.rs](../../highfleet-allies/ally/src/sound.rs) | `button`, `play_sfx`: native UI click and retained status-pointer lifetime |
| [qol/src/elevation.rs](../../highfleet-allies/qol/src/elevation.rs) | `prepare`, `native_tick`, `draw_hint`, `overlay_thread`: editor label seam plus an alternate click-through Win32 window |
| [perf/src/font_color.rs](../../highfleet-allies/perf/src/font_color.rs) | Font-level colour plus all 256 glyph pointers/four corners; why restoring just the font field is insufficient |
| [perf/src/hud_hooks.rs](../../highfleet-allies/perf/src/hud_hooks.rs), [hud_query.rs](../../highfleet-allies/perf/src/hud_query.rs) | Performance-oriented HUD spatial query adapters, whole-function fingerprints and callback-local views; not a widget framework |
| [highfleet-benchmark/docs/vanilla-performance.md](../../highfleet-benchmark/docs/vanilla-performance.md), [hud-spatial-queries.md](../../highfleet-benchmark/docs/hud-spatial-queries.md) | Companion performance evidence is in the benchmark checkout, not the allies docs directory; verify branch/experiment validation before adopting optimizations |
| [docs/ally-gui-v0.14.5-tester.md](../../highfleet-allies/docs/ally-gui-v0.14.5-tester.md) | Native lifecycle contracts and stale-pointer incident; dated tester status, not a current release/install instruction |
| [docs/debug-overlay.md](../../highfleet-allies/docs/debug-overlay.md), [shipworks-test-allies.md](../../highfleet-allies/docs/shipworks-test-allies.md) | Feature behaviour, address tables and validation limits |

## Other implementations and tools

| Workspace path | What to use it for | Caveat |
|---|---|---|
| [hf_probe/src/lib.rs](../../hf_probe/src/lib.rs) | Seam arrays/installers, per-seam hit counters, editor HUD, font sampler, sprite tests, click/text/sound/load logging | Diagnostic hooks default off in its v3 source because they conflict with production mods; do not enable all hooks at once |
| [hf_draw_test/src/lib.rs](../../hf_draw_test/src/lib.rs) | Small editor render-call trampoline and font gallery (`SAMPLER_FONTS`) | Throwaway experiment; its sampler is also integrated into hf_probe. Old ABI declarations are not a canonical binding |
| [hf_tacnet/src/guest.rs](../../hf_tacnet/src/guest.rs) | Draw streamed world positions without native ship objects; status text and cursor-to-world experiment | Same battle hook as ally; sprite style is left changed despite broader comments claiming restoration; font raw-field restore misses glyph colours |
| [hf_intro/README.md](../../hf_intro/README.md) | Existing native logo GUI/resource sequence and a minimal state-gate patch | Not a general graphics API; recorded logo playback was not yet game-validated |
| [hge_src/src/helpers/](../../hge_src/src/helpers/) | `hgefont.cpp`, `hgesprite.cpp`, `hgeanim.cpp`, `hgeresource.cpp`, parser/resources: algorithm and file-format reference | Stock source is not HighFleet's binary layout or x64 ABI |
| [hge_src/include/](../../hge_src/include/), [hge_src/src/core/graphics.cpp](../../hge_src/src/core/graphics.cpp) | HGE interfaces/GUI helpers and stock graphics pipeline; fonted/texasm/particle tools under `hge_src/src` | HighFleet's renderer is modified; do not transplant vtable indices or stock GUI widgets |
| [hf_dev_tools/README.md](../../hf_dev_tools/README.md) | Existing `DumpFunction.java`, `AnalyzeHookLoops.java`; data parsers/decoder if needed for assets/localization | No reusable graphics catalog/GUI-authoring tool was found in this tool index |
| [hf_notes/docs/engine-hge.md](../docs/engine-hge.md) | Version-specific HGE slots and native helper addresses | Some sections are historical guesses; preserve their evidence tags |
| [hf_notes/docs/structures.md](../docs/structures.md) | Existing D3D11 shader class/vtable findings and GUI/native layouts | Structure/name evidence does not establish a safe setter API |
| [hf_ml/out/ds3-subview/](../../hf_ml/out/ds3-subview/), [ds3-target/](../../hf_ml/out/ds3-target/), [ds-hge/](../../hf_ml/out/ds-hge/), [ds-viz/](../../hf_ml/out/ds-viz/) | Raw historical RE reports and captures behind panel, target and renderer notes | Archive discovery only; not reverified in this survey. Prefer curated notes over raw agent conclusions |
| [clickfunc_video/](../../clickfunc_video/), [hf_ml/out/ds3-target/gui_logs/](../../hf_ml/out/ds3-target/gui_logs/) | Existing local click/screen evidence | Not current GUI implementation or a directory to rename |
| [mail/gui_troubles/](../../mail/gui_troubles/) | Reporter logs and screenshots behind stale GUI-pointer diagnosis | Raw evidence may contain user data; do not package indiscriminately |
| [mail/improving_test_combat/ref_1.png](../../mail/improving_test_combat/ref_1.png) | User's Shipworks roster visual mock/reference | Design intent, not implementation/native proof |
| [highfleet_qol_fun/docs/src/highfleet_qol/shootable_gui/](../../highfleet_qol_fun/docs/src/highfleet_qol/shootable_gui/) | Existing generated/source-browser GUI material | This workspace copy is mostly packaged DLL/docs, not the primary editable implementation |

The Steam test installation's `Media/media.res`, `Media/Tex/*.res`,
`Media/Fnt/fonts.res`, `Media/Fnt/*.fnt`, `Media/Dst` and shader assets are the
authoritative **installed asset** inputs. It is at
`/home/maksim/.local/share/Steam/steamapps/common/HighFleet/`.
The installation may contain modded assets; compare with a known stock copy
before labelling an asset vanilla. No game assets were copied into these notes.

## Layout library and duplicate work

Ally's manifest pins `logdot/highfleet-rs` revision
`05c62078c6f896c141713a68b3991663e121b7bf`.
The source actually consumed is in the Cargo Git checkout, including
`src/v1_163/gui.rs`. The separate local [highfleet-rs/](../../highfleet-rs/)
checkout is older and did not contain `Gui` in this survey. Check the pinned
source and current upstream before adding a local duplicate layout; no upstream
HEAD audit or dependency update was performed here.

`hf_ally/`, `hf_ally_release/`, `hf_qol_fork/`, `old_standalone/`,
`hf-allies-rewrite/`, `highfleet-benchmark/`, `hf_notes_bench/notes_copy/`,
`release_packs/` and `*_build/` contain predecessors, alternate branches,
snapshots, experiments or packaged copies. They are useful for provenance and
before/after comparisons, not interchangeable sources of current behaviour.
Do not merge them into a second current graphics implementation by copying
everything. `hf_hee` is a campaign-pool editor, not the GUI toolkit implied by
the word “editor”; no graphics helper implementation was found in its source scan.

## Historical notes corrected for new development

| Older claim / assumption | Current guidance |
|---|---|
| Custom GUI objects never get freed | Scene teardown/rebuild can invalidate them; establish current registry membership before any cached-pointer read |
| Check a cached identifier to prove it is still live | That identifier read can itself be the use-after-free |
| Re-add detached GUI nodes | Native pending-deletion queues are separate; never resurrect them |
| SetBackground replaces current art | It appends; retarget one native Sprite and reload |
| “Refresh” just asks for redraw | GUI virtual `+0x88` loads native resources recursively |
| B24 is always a clean overlay seam | Later evidence: skipped with GLITCH OFF; current pre-combat text uses B23 |
| Every image is centre-anchored | Hotspot is defined per resource; use it |
| Restoring font colour field restores text | Glyph quads also carry colour; use a verified setter/complete restore |
| Build success proves graphics work | Validate layering, clicks, rebuilds, long sessions and supported resolutions in game |
