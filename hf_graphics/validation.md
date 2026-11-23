# Graphics development validation checklist

[Index](README.md) · [Calling rules](calling-guide.md)

Documentation/source review is not visual validation. Use this checklist when
implementing a new graphics mod or extracting the proposed library.

## Record the baseline

- Exact game build (Steam 1.163 here), DLL versions/source commits and configs.
- Modloader version, Wine/native Windows, display mode, resolution/aspect/DPI.
- Installed shader/texture/font modifications versus known stock assets.
- Owned hook sites and any conflicts/refused optional features in the log.
- Before/after screenshots and reproducible steps; do not infer a shader cause
  from “a button is invisible” without separating icon, text and input behaviour.

## Text, images and shapes

- Correct font, colour, alpha, alignment and anchor; ordinary game text unchanged
  after the mod draws. Test multiple mod labels sharing the same font/resource.
- No `@`/NUL/newline surprises; bounded long input; explicit handling of unsupported
  characters. Native localized controls must preserve the game's encoding.
- Correct hotspot and sprite size versus hit-box size; verify hover/pressed states.
- GLITCH ON/OFF, Esc, cover/door animation, modal/dialogue overlays, hidden HUD.
- Camera pan/zoom: world markers follow the intended point, screen text stays put,
  world-size radii scale once, line widths remain sensible.
- Missing resource and malformed/incomplete snapshot paths skip safely and log
  useful bounded diagnostics rather than creating uncontrolled duplicates.
- Validate each supported resolution/aspect explicitly; old 16:9 observations do
  not establish a universal viewport conversion.

## Native controls and lifecycle

- One physical click gives one action; distinct adjacent buttons debounce
  independently. Unowned native clicks retain normal behaviour.
- Enter/leave/re-enter, RETRY, ship/card swaps, load/reset and several consecutive
  battles in the **same process**. Hidden objects must not stay clickable.
- Verify current registry membership before cached-pointer reads, including
  hidden controls and root/state checks. An incomplete scan never proves absence.
- Repeated repainting does not grow the sprite vector. New/re-shown controls load
  assets outside rendering; no native teardown from a drawing callback.
- Do not trigger destructive diagnostic actions as part of an automated GUI pass.
  Use inert/harmless buttons for blind-click checks.

## Advanced targets and coexistence

- Preview text and hull both render, with correct damage/layer ordering.
- No stale texture/ship association after handoff, death, retreat or teardown;
  hidden panels are not keyboard-selectable.
- Target links and temporary marker counts are restored after every preview.
- Repeated allocation/re-entry, resize/device-reset and partial-failure behaviour
  need explicit evidence before a resource wrapper promises cleanup/recovery.
- Check hook combinations (ally, QoL, probe, tacnet/voice as applicable).
  A conflict must disable/report the affected integration, not overwrite a detour.
- Compare frame time/work counts at representative load before “optimizing”;
  do not remove native guards or increase scan caps merely to conceal a failure.

## Current evidence limits

Existing notes contain real in-game seam/font/sprite observations, pre-combat
history and user-confirmed Shipworks ADD/FIGHT. Some later paths remain build-
only or historically reported; CLEAR was still pending verification in v1.2.4.
The 2026-11-13 knowledge-base consolidation adds no new runtime validation.
No regression tests were written, no game configs/DLLs were changed, and no
diagnostic termination action was invoked.
