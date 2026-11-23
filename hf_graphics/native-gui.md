# Native GUI cookbook and lifetime rules

[Index](README.md) · [Calling guide](calling-guide.md)

Primary source: `highfleet-allies/ally/src/precombat.rs` and `shipworks.rs`
at `d44dd49`. Native verification record:
[GUI tester contracts](../../highfleet-allies/docs/ally-gui-v0.14.5-tester.md#native-contracts-checked-for-this-build).
Do not confuse native GUI `Gui`/Body Sprite objects with stock HGE `hgeGUI`
controls or the low-level `hgeSprite` returned by `GetAnimation`.

## A Shipworks-style button or label

`shipworks::tick`, `show_control`, and `create_control` are the most recent
compact example; they share the pre-combat lifecycle rather than installing a
second click/input detour.

1. From a game-thread input callback, obtain a **complete** GUI-list snapshot.
2. Locate the current active `EDC_TEST` control as the template. Read its actual
   width/height, position, stage/layer, normal/focus/pressed resource names and
   font. Reject non-finite/invalid geometry and malformed strings.
3. Use `get_or_make_with` to reuse a current valid owned control or construct a
   fresh one. A persistent identifier is a key, **not** a native pointer lifetime.
4. Allocate with the game's allocator `0x140334094` and run its constructor
   `0x14012a910(mem, 0)`. Current `Gui` allocation is `0xe70`; derive with
   `size_of::<highfleet::v1_163::Gui>()` plus a compile-time assertion.
5. Invoke virtual slot `+0x2c0`, the **12-argument initializer**:

   ```text
   init(gui, identifier, x, y, width, height,
        text, background, focus_background, pressed_background, font, font_scale)
   ```

   This copies borrowed input strings into native storage. It is not merely
   a name setter; the floating arguments and Win64 stack layout matter.
6. Apply the template's stage/layer and intended flags. A button is focusable
   and clickable; a label is neither, with empty backgrounds. Shipworks roster
   labels use `flash_large_shadow`, scale 1, left alignment.
7. Attach **only the fresh object** to the global root `*(0x147fc6f88)` through
   root virtual `+0x50(root, gui, 0, 0)`.
8. Call GUI virtual `+0x88` (`GUI::Load`, `0x140163110`) to load media.
   The initializer path registers the GUI synchronously; do not invent an
   extra registry insertion.
9. On reuse, update text with native string assignment `0x1400056e0`, move
   through `0x1400520b0`, update hit-box floats `+0x218/+0x21c`, and reload
   resources when required after a rebuild/show transition.

Native strings use SSO: the current Shipworks reader bounds length to 256,
checks length/capacity and NUL, then copies bytes into a Rust `CString`.
It never copies/drops a native `EscadraString`. Never memcpy an entire GUI
template: its strings, sprites, links and registration are owning native state.

The Shipworks ADD/FIGHT feature was user-confirmed in game at v1.2.1.
CLEAR remains pending independent in-game verification in the v1.2.4 record.
This is not evidence that every control recipe, resolution and lifecycle is tested.

## Registry membership before dereference

Registry head/count: `0x143a13ac8` / `0x143a13ad0`; sentinel list node `+0`
is next, `+0x10` is the GUI pointer. Current `scan_gui_list`:

- walks to the sentinel with `clamp(count + 16, 256, 4096)` iterations;
- records whether it actually reached the sentinel;
- includes inactive objects, unlike the native active-only find helper;
- returns incomplete on an invalid traversal or bound exhaustion.

For a cached address, **first** prove membership in this callback's snapshot.
Only then inspect:

| Field | Required interpretation |
|---|---|
| `Gui.identifier` (`+0xa40`) | Expected namespaced identifier |
| `Node.state` (`+0x48`) | 0 hidden/unused or 2 shown; not 1 removed/dying |
| Registry cell (`+0xba8`) | Nonzero registered membership |
| `Node.master.target` (`+0x88`) | Still owned by the current GUI root |

An incomplete walk is not proof that an object/screen is gone. Do not create
duplicates, revive cached objects, or run an unbounded native fallback search.
Explicit exit handling may hide positively observed members and retry later.

Detached/foreign-parented/dying elements are abandoned and replaced only after
the appropriate complete scan. **Never re-add them to the root:** `Node::add`
does not remove their pending native deletion entries. Reading a cached name
first already risks a use-after-free; readable-page checks do not solve that.

The motivating capture in `mail/gui_troubles/` read
`old ALLY_TOGGLE_2 pointer + 0xa40` after that pointer disappeared from the next
registry. See the [report analysis](../../highfleet-allies/docs/ally-gui-v0.14.5-tester.md#evidence-from-the-reporters-v0144-logs).

## Own exactly one background sprite where that is the recipe

`SetBackground_` (`0x14001be50`) **allocates/appends**, including when the final
argument is `-1`; that argument is an insertion position, not a replacement slot.
Calling it on every repaint accumulates sprites.

Current pre-combat `paint_button`:

1. Validate the `Body.sprites` vector header (`+0x8c0`), ordered begin/end/capacity,
   pointer-size alignment and a bounded capacity (4096 maximum in current code).
2. With one sprite, use native name setter `0x1402e2030` on that existing Sprite.
   The setter skips an unchanged resource name.
3. With none, call native `SetBackground_` once.
4. With multiple accumulated sprites, validate their pointers, clear through
   `ObjectArray_DeleteAll` (`0x14004bb10`), then create one. This uses native
   deleting destructors, not Rust `drop` or a guessed `free`.
5. Call `GUI::Load` on the validated control outside rendering.

The Body Sprite's loaded animation handle at `+0x78` distinguishes missing
resources from a missing sprite entry. That wrapper is **not** the same object
as the low-level HGE animation and must not use its offsets.

Hit-box size does not resize the sprite: `precombat_capital` on a 64×64 hit box
once covered the screen with a large paper texture. Start with known small
`order_*` resources or the actual native button backgrounds.

## Clicks, movement and hiding

- `ClickFunc` (`0x14012f120`) receives `(context, gui, button, id)` in
  RCX/RDX/R8/R9. Ally's entry hook can consume its names and skip the original;
  unowned events continue unchanged. The native caller ignores its return.
- Check the screen/visibility gate and left button (`1`) before acting.
  Debounce **per control**; current ADD/CLEAR and pre-combat code use 120 ms.
  One physical click was observed arriving twice. Debounce is not a general
  replacement for proper press/release/text-input semantics.
- Use unique prefixes; route handlers through one dispatcher instead of
  each feature patching ClickFunc independently.
- For moving pre-combat cards, follow actual card geometry each input tick.
  Slot position and ship identity differ when cards swap: plane columns follow
  their ship. Do not use one-time build positions or stale slot-to-ship mapping.
- Set state to 0 only for positively identified owned live controls on exit;
  do not free arbitrary GUI nodes. Revalidate on re-entry and after RETRY/rebuild.
- Seam-drawn labels bypass native GUI occlusion. Existing pre-combat code gates
  them for `ROLL_DESK` and `QUIT_BLACK`. Prefer native text controls when
  participation in native layering is desired.
- Stock click sound uses `sound.rs::button`, `button_01`, base volume 30.
  Its native status/volume pointer is retained; stack-local storage is unsafe.
  Keep sound playback optional and preserve the documented long-lived storage.

## Layout dependency, not an allocation API

Ally pins logdot's `highfleet-rs` to full revision
`05c62078c6f896c141713a68b3991663e121b7bf` in Cargo.toml/lock.
`native_layout.rs` derives offsets/sizes and asserts the contract.
The workspace's separate `highfleet-rs/` checkout is older and has no `Gui`
definition; the pinned Cargo checkout does. Do not conclude a layout is missing
from upstream from that older working directory.

Use `highfleet::v1_163::{Gui, Body}` and `highfleet::general::{Node, Link}` where
covered. This audit did not check current upstream HEAD or change dependencies.
Before actual library implementation, review upstream and the chosen pinned
revision. Raw scalar/pointer reads do not transfer native ownership; never
read an owning GUI/string/vector/Link into a Rust value or use Rust drop on it.
