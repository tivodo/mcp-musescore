# Plan: notation tools for chord charts and lead sheets

Decisions from a design interview on 2026-11-14, based on a working session that transposed a chord-accompaniment
score (MusicXML imported into MuseScore 4) and on [musescore-4.7-plugin-api.md](musescore-4.7-plugin-api.md).

## The problem

In that session the MCP could read the notes well, but could not fix the presentation:

- Claude never sees the engraved score. `show_score` draws the notes itself from `getScore` data, not MuseScore's
  layout, so overlapping chord symbols, marker size and system breaks were invisible to it.
- Tools report success when nothing happened: `setStyle` accepts any key (`useGermanNoteNames` no longer exists, so
  it did nothing but returned success).
- `get_score` misses measure-attached elements (Coda, To Coda, jumps, breaks) and shows imported chord symbols with an
  empty name.
- There are no tools for markers and jumps, chord symbols, element properties (alignment, font size) or layout
  (breaks, measures per system), so all of these were fixed by hand.

## Direction

- **Destination:** first the own fork, used in practice; the parts that prove useful go upstream as PRs. Follow
  upstream's conventions from the start (tests against the mock plugin, docs in `docs/`, small PRs).
- **Driving use case:** chord charts and lead sheets. Prioritise by what that workflow needs.
- **Base branch:** `feature/notation-tools`, branched from upstream PR #23 (`feature/ruff_and_docstrings`), which
  reformats most files. Rebase if #23 changes before it merges.
- **One PR per step** below (steps 2–8). Each PR updates `docs/tools.md`.

## Steps

1. **Spike** on MuseScore 4.7.7, on a `prototype/` branch, thrown away afterwards. It answers:
   1. Does `cmd("select-similar")` see a selection the plugin made?
   2. Does `writeScore(..., "png")` work, at what resolution, and does `"mpos"` give usable measure positions?
   3. Does a Marker added with `cursor.add` work, and does it show up in `measure.elements`?
   4. Is an imported chord symbol's name readable after save and reopen?
   5. Does `curScore.addRemoveSystemLocks(4, false)` work on a range selection?

   Results go into [musescore-4.7-plugin-api.md](musescore-4.7-plugin-api.md).
2. **Connection:** a timeout on `recv`, a lock so concurrent tool calls cannot interleave on the one socket, and a
   retry only when sending failed (never after a sent command, which could apply it twice).
3. **Honest feedback:**
   - `setStyle` rejects an unknown key (`style.value(key) === undefined`) and returns the value read back.
   - New `get_style(keys)`.
   - `get_score` lists measure-attached elements: Marker, Jump, line and page breaks.
   - A chord symbol with no readable name shows as `Harmony (name not readable)`, not as an empty string.
   - A clear error when the plugin doesn't answer: start it from Plugins → MuseScore API Server.
   - The `get_score` header in English (it is in French now).
4. **`render`:** the score as MuseScore engraves it, returned as an image.
   - `page=N`: the whole page, to check layout and system breaks.
   - `measures=X-Y`: cropped from the same page image (using measure positions), to check details. A range across
     two systems gives both. Needs Pillow for the crop.
5. **`find_elements` + `set_property`:** find elements by type (and range), set generic properties on them:
   alignment, font size, placement, offset, and measure stretch (`userStretch`).
6. **Markers and jumps:** add and delete Segno, Coda, To Coda, Fine, D.S./D.C. variants. No conversion from text:
   a coda imported as text is an import setting, not the MCP's job.
7. **Chord symbols:** add, change, move and delete at a measure and beat. Moving deletes and re-creates with the same
   text; if the name is not readable, the move is refused and Claude asks for the name.
8. **Layout:** add and remove line and page breaks; set measures per system.

## Acceptance

- Tests use only made-up fixtures in the repository.
- At the end, the original session is re-run on the user's machine with the same requests, counting how many of the
  six manual fixes now succeed through the MCP. That score stays out of the repository (copyright).

## Out of scope

- Transposition (only one semitone at a time through the plugin; quick by hand).
- Opening and saving files (`.mscz` writing fails in 4.7; opening creates new windows).
- Loading and saving style files.
- Reading names of imported chord symbols (an import problem, not the MCP's).
- Converting text to markers (handled by the import settings).
- The user's own PDF → MusicXML converter (a separate project).

## After the spike

The spike answered all five questions (see
[Spike results](musescore-4.7-plugin-api.md#spike-results-on-musescore-477-windows-gui)):

- **`find_elements` can use MuseScore's own "select similar":** `cmd("select-similar")` sees a selection the plugin
  made.
- **`render`:** `writeScore` to PNG works; the page is 1200 DPI by default (from the user's preferences). The crop
  is computed from `"mpos"` (bar box in pixels = mpos box × dpi / 14400). Still to decide when building it: the
  output size handed to Claude, and where the temporary images go.
- **Markers are attached with `cursor.add`.**
- **Measures per system:** `addRemoveSystemLocks(N, false)` on a range selection.
- **Imported chord names become readable after save and reopen.** The step 3 message can say so:
  `Harmony (name not readable: save and reopen the score)`.
- **MuseScore 5.0 (main, not released) is not a reason to avoid `cmd()`.** 5.0 replaces action names with
  `command://...` ids and maps only 11 old names, so `select-similar` and most of the current plugin's `cmd()` calls
  will need new names there. That is one change across the whole plugin when 5.0 ships, so the tools use `cmd()`
  where it is the natural way, and build on 4.7.7.

## Progress (2026-11-19)

Steps 1–4 are merged upstream, each as its own PR:

- Step 2: [#25](https://github.com/ghchen99/mcp-musescore/pull/25).
- Step 3: [#26](https://github.com/ghchen99/mcp-musescore/pull/26), with `get_style` and `set_style`.
- Step 4: [#27](https://github.com/ghchen99/mcp-musescore/pull/27). `render` writes the score as PDF and crops it with
  pypdfium2 and Pillow, not PNG (1200 DPI is about 140 megapixels a page). It replaced `show_score` and the VexFlow
  viewer, as agreed with the maintainer.
- Outside the plan: [#28](https://github.com/ghchen99/mcp-musescore/pull/28). Every tool takes and gives bars as
  printed in the score, not by position.

Steps 5–8 and the acceptance run are still to do. The spike's code was thrown away; this file and the API research
are kept on the fork's `research/notation-tools` branch.
