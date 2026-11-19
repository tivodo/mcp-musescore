# MuseScore 4.7 QML plugin API: what it can and cannot do

Research notes for the QML plugin `musescore-mcp-websocket.qml`. Every claim was checked against the MuseScore source at tag
[`v4.7.0`](https://github.com/musescore/MuseScore/tree/v4.7.0) (commit `9989a50ad8419258c58f50a42be22bcedbdd5267`).
The plugin API reference docs are generated from the `APIDOC` / `///` comments in `src/engraving/api/v1/`, so those
comments are quoted here as the "official docs". No forum posts or blogs were used.

Labels used below:

- **Verified**: read in the source at v4.7.0.
- **Inferred**: follows from the source, but depends on runtime behaviour (dialogs, layout, focus) that was not run. These are listed again under [Open questions](#open-questions--needs-a-live-test).

Links use `S/` as shorthand for `https://github.com/musescore/MuseScore/blob/v4.7.0/src/`.

### Still valid for v4.7.7 (latest release, 2026-11-12)

`git diff v4.7.0 v4.7.7` was checked for the paths these notes rely on. **The plugin API is unchanged**: there are no
commits under `src/engraving/api/`, and none to `dom/harmony.cpp`, `dom/marker.cpp`, `dom/jump.cpp`,
`dom/layoutbreak.cpp` or `style/styledef.cpp` (`styledef.h` only gains a helper, `DEFAULT_SMUFL_POINT_SIZE()`, and no
new keys). Everything below therefore also holds for 4.7.1–4.7.7. Changes between those releases that matter here:

- **MusicXML import of coda and segno on barlines** (`be34150742`, "import coda and sgeno on barlines", first in a 4.7.x
  patch release): a `<barline>` with a `coda` or `segno` attribute now becomes a real `Marker` (`MarkerType::CODA` or
  `SEGNO`). A coda written as `<words>` text still imports as StaffText.
- **Chord symbols on import** (`deef97be8f`, `af9df8b480`): chord symbols are attached to their fret diagrams. The
  import still doesn't set the chord symbol's text, so section 3's "no" for reading imported chord names stands.
- **Transposition by key** (`fbad26f97a`, `Transpose::keydiff2Interval` in `editing/transpose.cpp`): the interval
  computed from two keys is fixed, which affects the Transpose dialog's "to key" mode. `transpose-up`/`-down` are not
  affected.
- **SVG export** (`a3788d0c79`): items on invisible staves are no longer drawn.
- **Plugins** (`6885aa157e`): adding a hairpin from a plugin command no longer crashes.

## Summary

| # | Question | Possible? | Mechanism |
|---|----------|-----------|-----------|
| 1 | PNG/SVG image of the score | **Yes**, with caveats | `writeScore(curScore, path, "png"/"svg")`. One file per page: `path` as given for a 1-page score, `base-1.png`, `base-2.png`… for more pages. DPI comes from the user's preference (no plugin parameter). Existing files cause a modal dialog when there are 2 or more pages. Fallback with DPI control: the CLI `mscore -o out.png -r <dpi> [-T <margin>]`. |
| 2 | Markers / jumps (Segno, Coda, To Coda, Fine, D.S. al Coda…) | **Yes** | Create: `newElement(Element.MARKER)` + `markerType = MarkerType.CODA`. Jumps: `newElement(Element.JUMP)` + `text`, `jumpTo`, `playUntil`, `continueAt`. Attach: `cursor.add(e)` (the cursor sets the track) or `measure.add(e)`. Read: `measure.elements`, using `type`, `subtype`, `markerType`, `label`, `jumpTo`. Delete: `measure.remove(e)` or `removeElement(e)`. StaffText to marker: read `text`, create a marker, remove the StaffText. |
| 3 | Chord symbols (Harmony) | **Partly** | Create, modify, delete: yes (`newElement(Element.HARMONY)`, set `text`, `cursor.add`; set `text` again to change it; `removeElement`). Move to another tick: only by deleting and re-creating it. **Reading the displayed name of a MusicXML-imported chord: no**, because `text` is empty and no root, kind or harmonyName property exists. Fallback: parse `<Harmony>` in the .mscz, or the exported MusicXML. |
| 4 | Layout: system breaks, measures per system, stretch | **Yes** | Breaks: `newElement(Element.LAYOUT_BREAK)` + `layoutBreakType = LayoutBreak.LINE` + `measure.add(lb)`, or `cmd("system-break")` on a selection. Remove: find it in `measure.elements`, then `measure.remove(lb)`. Measures per system: `curScore.addRemoveSystemLocks(N, false)` on a range selection (since 4.6). `cmd("measures-per-system")` only opens a dialog. Stretch: `measure.userStretch`, or `cmd("stretch+")` / `cmd("stretch-")` / `cmd("reset-stretch")` on a range. |
| 5 | Transpose without the dialog | **Partly** | `cmd("transpose-up")` / `cmd("transpose-down")` move one semitone. They act on the selection, include chord symbols, and transpose key signatures only with a range selection. There is no plugin function for an arbitrary interval: repeat the command, or set `note.pitch`/`tpc` and chord-symbol `text` yourself. `cmd("transpose")` opens the dialog. Key change: `newElement(Element.KEYSIG)` + `concertKey`/`actualKey` + `cursor.add`. Fallback: CLI `--score-transpose`. |
| 6 | Style | **Yes** for single keys. **No** for loading or saving a style file | `curScore.style.value(key)` reads, `setValue(key, v)` writes (undoable), `resetValue(key)` resets (new in 4.7). An unknown key: `value()` returns `undefined` and `setValue` silently does nothing. Detect it with `value(key) === undefined`. Valid keys are the `Sid` names in `styledef.cpp`/`styledef.h`. `useGermanNoteNames` no longer exists: use `chordSymbolSpelling` (`NoteSpellingType`). `cmd("load-style")` / `cmd("save-style")` open a file dialog. Fallback: edit the `<Style>` in the .mscz, or CLI `-S file.mss`. |
| 7 | Selection and generic properties | **Yes** | `curScore.selection.elements`, `select(e, add)`, `selectRange`, `clear`. "Select all similar": `select(e)` + `cmd("select-similar")`, or loop over the score and call `select(e, true)` for each item. Generic properties: `placement`, `align`, `fontSize`, `fontFace`, `fontStyle`, `offset`, `color`, `visible`, `autoplace`, `minDistance`, and others (list below). Caveat: `align` reads back as `[h, v]` but must be written as an `Align` bitmask. |
| 8 | Open, save, close | **Partly** | `writeScore(…, "mscz"/"mscx")` **does not work** in 4.7: the writer only supports `writeList()`, which returns `NotSupported`, and the user gets a modal "error… retry?" dialog. `readScore(path)` works only when no score is open in the window (otherwise MuseScore opens a new window and returns the *old* score). `readScore(path, true)` always returns `null`. `closeScore()` closes the current score, and asks whether to save first if it has unsaved changes. To save .mscz: `cmd("file-save")` (saves to the score's own path) or edit the file. |

---

## 1. Rendering to PNG / SVG

**Possible: yes (verified).** The caveats are below.

### How `writeScore` works

- `PluginAPI::writeScore(Score*, name, ext)` only accepts the current score, then calls `helper()->writeScore(name, ext)`.
  [S/engraving/api/v1/qmlpluginapi.cpp#L272-L285](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.cpp#L272-L285)
- `EngravingPluginAPIHelper::writeScore` looks up a writer for `ext`, picks the unit type (`PER_PAGE` if the writer supports it, else `PER_PART`, else `MULTI_PART`), appends `.ext` if `name` does not end with it, and calls `ExportProjectScenario::exportScores({ currentNotation }, outPath, unitType, false)`.
  [S/project/internal/engravingpluginapihelper.cpp#L34-L50](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/engravingpluginapihelper.cpp#L34-L50), [#L69-L83](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/engravingpluginapihelper.cpp#L69-L83)
- `png`, `svg` and `pdf` writers are registered: [S/importexport/imagesexport/imagesexportmodule.cpp#L55-L57](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/imagesexportmodule.cpp#L55-L57).
  `PngWriter` and `SvgWriter` both support only `UnitType::PER_PAGE`:
  [pngwriter.cpp#L37-L39](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/pngwriter.cpp#L37-L39), [svgwriter.cpp#L47-L49](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/svgwriter.cpp#L47-L49).

### Per page, and how files are named (verified)

`exportScores` loops over every page in `PER_PAGE` mode
([exportprojectscenario.cpp#L183-L208](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L183-L208)):

- A **1-page** score ("only one file", see `guessIsCreatingOnlyOneFile`, [#L257-L291](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L257-L291)) is written to **exactly `path`**, with no suffix.
- A score with **more pages** is written to `<dir>/<basename>-<page+1>.<ext>`: `out-1.png`, `out-2.png`, …
  `completeExportPath` [#L326-L342](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L326-L342).
  When the current notation is the main score or the only score exported, no part-name suffix is added.
- So the caller cannot know the file names without knowing the page count. Read `curScore.npages` first
  ([S/engraving/api/v1/score.h#L142](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L142)).
- **Overwriting:** with a single file the conflict policy is `ReplaceAll` (silent overwrite). With 2 or more pages it is `Undefined`. In that case, if any target file already exists, `shouldReplaceFile` opens a **modal "File already exists" dialog** ([#L344-L382](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L344-L382)). Delete old files before exporting. Write errors open a modal "retry?" dialog through `askForRetry` ([#L384-L392](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L384-L392), [doExportLoop #L394-L440](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L394-L440)).
- Export switches the view to page mode and back ([#L141-L142](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L141-L142)), so the images are the page layout even if the user is in continuous view.
- The return value is `true` unless the user cancels. A write error that the user answers with "Abort" returns `false`.

### Resolution (verified: a plugin cannot set it)

- `PngWriter` uses `configuration()->exportPngDpiResolution()` ([pngwriter.cpp#L48-L54](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/pngwriter.cpp#L48-L54)). That comes from an override (set by the command line) or the user preference `export/png/resolution` ([imagesexportconfiguration.cpp#L35](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/imagesexportconfiguration.cpp#L35), [#L79-L86](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/imagesexportconfiguration.cpp#L79-L86)). Transparency and grayscale also come from preferences.
- Trimming (`trimMarginPixelSize`) is `-1` (off) unless set from the command line ([#L138-L146](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/imagesexportconfiguration.cpp#L138-L146)).
- The plugin API has no way to pass writer options. `writeScore` only takes `(score, name, ext)`.
- SVG is vector output, so resolution does not apply. Each SVG is one page ([svgwriter.cpp#L72-L85](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/svgwriter.cpp#L72-L85)).

### Alternatives

- **The command line** (outside the plugin, from the Python server): `-o <file>.png`, `-r/--image-resolution <DPI>`, `-T/--trim-image <margin>`
  ([S/app/internal/commandlineparser.cpp#L75](https://github.com/musescore/MuseScore/blob/v4.7.0/src/app/internal/commandlineparser.cpp#L75), [#L89-L90](https://github.com/musescore/MuseScore/blob/v4.7.0/src/app/internal/commandlineparser.cpp#L89-L90)). It needs a saved file, so the live, unsaved score is not covered.
- **Layout geometry without an image:** `writeScore(curScore, path, "mpos")` or `"spos"` writes measure or segment positions as XML (x, y, sx, sy, page) ([S/notation/notationmodule.cpp#L61-L62](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/notationmodule.cpp#L61-L62), [positionswriter.cpp#L51-L62](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/internal/positionswriter.cpp#L51-L62)). Each element wrapper also has `pagePos` and `bbox` ([elements.h#L245](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L245), [#L262](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L262)). Use these to map a PNG back to measures.

---

## 2. Markers and jumps

**Possible: yes (verified), with one inferred caveat about the track.**

### Types and enums exposed to QML (verified)

- `Element.MARKER`, `Element.JUMP` ([apitypes.h#L370-L371](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/apitypes.h#L370-L371)).
- `MarkerType.{SEGNO, VARSEGNO, CODA, VARCODA, CODETTA, FINE, TOCODA, TOCODASYM, DA_CODA, DA_DBLCODA, USER}` ([apitypes.h#L1405-L1418](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/apitypes.h#L1405-L1418), registered at [qmlpluginapi.h#L362](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.h#L362)).
- `JumpType.{DC, DC_AL_FINE, DC_AL_CODA, DS_AL_CODA, DS_AL_FINE, DS, DC_AL_DBLCODA, DS_AL_DBLCODA, DSS, DSS_AL_CODA, DSS_AL_DBLCODA, DSS_AL_FINE, USER}` ([apitypes.h#L1388-L1403](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/apitypes.h#L1388-L1403), [qmlpluginapi.h#L359](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.h#L359)). **This enum is read-only in practice: no property sets a jump type** (see below).

### Properties on the element wrapper (verified)

- `markerType` (int, `Pid::MARKER_TYPE`), `label`, `markerCenterOnSymbol` ([elements.h#L658-L667](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L658-L667)).
- `jumpTo`, `playUntil`, `continueAt` (strings), `playRepeats` ([elements.h#L652-L656](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L652-L656), [#L826](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L826)).
- `subtype` (read-only) returns `element()->subtype()` ([elements.h#L270](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L270), [#L1300](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1300)). That is `int(markerType)` for a Marker ([marker.h#L64](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.h#L64)) and `int(jumpType())` for a Jump ([jump.h#L57](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.h#L57)). `subtypeName()` returns the translated name ([elements.h#L1342](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1342)). Do **not** use `subType` (capital T): its doc says it "no longer returns the subtype" ([elements.h#L323-L328](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L323-L328)).

### Creating a marker (verified)

```js
var m = newElement(Element.MARKER);
m.markerType = MarkerType.CODA;   // Marker::setMarkerType fills in text + label + text style
cursor.rewind(Cursor.SCORE_START) /* …move to the measure… */;
cursor.track = 0;
cursor.add(m);                    // parented to the measure, track = cursor.track
```

- Setting `markerType` calls `Marker::setProperty(MARKER_TYPE)`, which calls `setMarkerType`. When the text is empty, that sets the text from `markerTypeTable` (for example `<sym>coda</sym>`, `"To Coda"`, `"Fine"`), sets `label` (`"codab"`, `"coda"`, `"segno"`, `"fine"`…) if it is still the default, and picks left or right repeat text style ([marker.cpp#L51-L61](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L51-L61), [#L86-L100](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L86-L100), [#L140-L156](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L140-L156)).
- An int is converted correctly: `PropertyValue::fromQVariant` turns it into `MarkerType(v.toInt())` ([propertyvalue.cpp#L309](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/types/propertyvalue.cpp#L309)).
- Before the element is added (PLUGIN ownership), the setter calls `setProperty` directly. After it is added (SCORE ownership), the setter calls `undoChangeProperty` ([scoreelement.cpp#L162-L227](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L162-L227)).
- `Cursor::add` handles `MARKER`/`JUMP` explicitly: it parents the element to the cursor's measure and calls `undoAddElement` ([cursor.cpp#L285-L291](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/cursor.cpp#L285-L291), [#L319-L331](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/cursor.cpp#L319-L331)). `Measure::add` then sets `repeatJump` for jumps and for "right" markers (To Coda, Fine…) ([measure.cpp#L890-L897](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/measure.cpp#L890-L897)).
- `measure.add(e)` (since 4.6) also works, but **does not set the track** ([elements.cpp#L444-L465](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.cpp#L444-L465)). A new item's track defaults to `nidx` ([engravingitem.h#L716](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/engravingitem.h#L716)), and the UI's drop code sets `track = 0` for markers and jumps ([measure.cpp#L1527-L1532](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/measure.cpp#L1527-L1532)). So with `measure.add`, set `e.track = 0` first. The source itself questions that this property converts correctly ([elements.h#L764](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L764): "does conversion work from P_TYPE::SIZE_T ?"). **Prefer `cursor.add` with `cursor.track = 0`** (inferred; needs a live test).

### Creating a jump (verified)

`Jump::setJumpType` is not reachable from QML. The `jumpType` is derived from the three strings by matching `jumpTypeTable` ([jump.cpp#L49-L63](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.cpp#L49-L63), [#L80-L92](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.cpp#L80-L92), [#L98-L106](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.cpp#L98-L106), [#L138-L153](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.cpp#L138-L153)). Set the strings yourself, using the table values:

| Jump | `text` | `jumpTo` | `playUntil` | `continueAt` |
|---|---|---|---|---|
| D.C. | `D.C.` | `start` | `end` | `` |
| D.C. al Fine | `D.C. al Fine` | `start` | `fine` | `` |
| D.C. al Coda | `D.C. al Coda` | `start` | `coda` | `codab` |
| D.S. al Coda | `D.S. al Coda` | `segno` | `coda` | `codab` |
| D.S. al Fine | `D.S. al Fine` | `segno` | `fine` | `` |
| D.S. | `D.S.` | `segno` | `end` | `` |

(The double-coda and double-segno rows are in the same table.) The labels must match the target markers' `label`s: Segno `segno`, Coda `codab`, To Coda `coda`, Fine `fine` ([marker.cpp#L51-L61](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L51-L61)).

**Caveat (inferred):** a Jump created with `newElement` does not go through `setJumpType`, so it keeps its constructor text style, `REPEAT_RIGHT` ([jump.cpp#L69-L74](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/jump.cpp#L69-L74)). That is the same style `setJumpType` would set, so it should look the same as a palette jump.

### Reading (verified)

Markers and jumps are **measure elements, not segment annotations**. Enumerate them with `measure.elements`, which the docs describe as the "List of measure-related elements: layout breaks, jump/repeat markings etc." ([elements.h#L1934-L1936](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1934-L1936), backed by `measureBase()->el()`, [#L1994](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1994)). Example:

```js
for (var m = curScore.firstMeasure; m; m = m.nextMeasure)
  for (var i = 0; i < m.elements.length; ++i) {
    var e = m.elements[i];
    if (e.type === Element.MARKER) { /* e.markerType, e.label, e.text */ }
    else if (e.type === Element.JUMP) { /* e.subtype (JumpType), e.jumpTo, e.playUntil, e.continueAt, e.text */ }
  }
```

`measure.repeatJump` tells you whether a measure has a jump or right-side marker ([elements.h#L1906](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1906)).

### Deleting (verified)

`measure.remove(e)` deletes the element through `score->deleteItem` (undoable), but only if `e` is a child of that measure ([elements.cpp#L467-L477](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.cpp#L467-L477)). Otherwise use `removeElement(e)` ([qmlpluginapi.cpp#L374-L378](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.cpp#L374-L378)).

### Converting an existing StaffText (verified building blocks; no single call)

There is no API to change an element's type. Do it by hand:

1. Find the StaffText in `segment.annotations` ([elements.h#L1832](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1832)) and read its `text` (an XML string that may contain `<sym>`, [textbase.cpp#L2632-L2633](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/textbase.cpp#L2632-L2633)).
2. Map the text to a marker or jump type yourself ("Fine", "To Coda", "D.S. al Coda"…).
3. Create the marker or jump as above, on the StaffText's measure.
4. Delete the StaffText with `removeElement(staffText)`.

Do all of this inside `startCmd`/`endCmd` so that one undo step reverts it.

---

## 3. Harmony (chord symbols)

**Possible: partly (verified).**

### What is exposed (verified)

The wrapper has no Harmony-specific class. Harmony items use the generic `EngravingItem` wrapper:

- `text` (`Pid::TEXT`), `harmonyType` (`HarmonyType.STANDARD/ROMAN/NASHVILLE`), `harmonyVoicing`, `harmonyDuration`, `harmonyVoiceLiteral`, `harmonyBassScale`, `harmonyDoNotStackModifiers`, `play`, plus the generic text, position and font properties ([elements.h#L548](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L548), [#L790-L804](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L790-L804), [#L1103](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1103)). See also `Harmony::getProperty` ([harmony.cpp#L1473-L1499](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L1473-L1499)).
- **Not exposed:** root or bass TPC, chord kind, `harmonyName()`, `displayText()`, `accessibleInfo()`. `htmlText` (`Pid::HTML_TEXT`) is declared ([elements.h#L550](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L550)) but no element handles that Pid ([property.cpp#L157](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/property.cpp#L157) is its only occurrence in `dom/`), so it returns nothing useful. `userName()` returns only the type name ([scoreelement.cpp#L86-L89](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L86-L89)).
- `curScore.hasHarmonies` / `harmonyCount` ([score.h#L210-L217](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L210-L217)).

### Why `text` is empty after a MusicXML import (verified)

- The MusicXML importer builds `HarmonyInfo` entries (root TPC, bass TPC, `textName` from the chord list) and calls `addChord(info)`. It **never sets the text** ([importmusicxmlpass2.cpp#L7800-L7990](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/musicxml/internal/import/importmusicxmlpass2.cpp#L7800-L7990)).
- The visible name is built on demand by `Harmony::harmonyName()` from those entries and the `chordSymbolSpelling` style ([harmony.cpp#L198-L240](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L198-L240)).
- Only `afterRead()` (loading a .mscz) and editing copy it into the text with `setPlainText(harmonyName())` ([harmony.cpp#L409-L447](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L409-L447), [#L686-L690](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L686-L690), [#L814](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L814)).
- `text` returns `xmlText()`, which stays empty ([textbase.cpp#L2367-L2375](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/textbase.cpp#L2367-L2375)).

**Fallbacks:**

- (a) Parse `<Harmony>` in the saved .mscz. `TWrite::write(Harmony)` writes `harmonyType`, `rootCase`, and per chord the root/bass TPCs and name through `writeHarmonyInfo` ([twrite.cpp#L1788-L1836](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/rw/write/twrite.cpp#L1788-L1836)).
- (b) Export MusicXML (`writeScore(…, "mxl")`, which works) and read `<harmony><root>/<kind>`.
- (c) **Inferred:** `cmd("file-save")`, then reopen. `afterRead` would then fill in `text`.

### Create, modify, move, delete (verified)

- **Create:** `var h = newElement(Element.HARMONY); h.text = "Bb7"; cursor.add(h);`. Setting `TEXT` on a Harmony calls `setHarmony(text)`, which parses the chord ([harmony.cpp#L1551-L1563](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L1551-L1563), [#L865-L886](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/harmony.cpp#L865-L886)). `Cursor::add` has no Harmony case, so it falls into `default`: the element gets the cursor's track and segment as parent, then `undoAddElement` ([cursor.cpp#L427-L430](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/cursor.cpp#L427-L430)). Put the cursor on a ChordRest segment at the target tick first.
- **Modify:** `h.text = "C#m7"` (undoable once the item is in the score). Placement, offset, font: the generic properties below.
- **Move to another tick:** there is no parent or tick setter. Delete the item and create a new one at the new tick. `offset` only moves it visually.
- **Delete:** `removeElement(h)`.

---

## 4. Layout: breaks, measures per system, stretch

**Possible: yes (verified).**

### System, page and section breaks (verified)

- **Element:** `newElement(Element.LAYOUT_BREAK)`, set `layoutBreakType = LayoutBreak.LINE` (or `PAGE`, `SECTION`, `NOBREAK`) ([elements.h#L511](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L511), [apitypes.h#L594-L600](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/apitypes.h#L594-L600), [qmlpluginapi.h#L193](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.h#L193)), then:
  - `measure.add(lb)` ([elements.cpp#L444-L465](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.cpp#L444-L465)), or
  - `cursor.add(lb)`, which handles `LAYOUT_BREAK` through `MeasureBase::addInternal` ([cursor.cpp#L314-L318](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/cursor.cpp#L314-L318)).

  `MeasureBase::add` sets the measure's `lineBreak`/`pageBreak` flags ([measurebase.cpp#L149-L180](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/measurebase.cpp#L149-L180)). Set the type *before* adding: the flags are taken from the type when the break is added.
- **Remove:** find `e.type === Element.LAYOUT_BREAK` in `measure.elements`, then `measure.remove(e)`.
- **Command alternative:** `cmd("system-break")`, `cmd("page-break")`, `cmd("section-break")` toggle a break on the selection (last measure of a range) ([notationactioncontroller.cpp#L265-L270](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L265-L270), [Score::cmdToggleLayoutBreak, cmd.cpp#L4883-L4922](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/cmd.cpp#L4883-L4922)). These commands only run when their check function (`toggleLayoutBreakAvailable`) allows it ([notationactioncontroller.cpp#L578-L603](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L578-L603), [actionsdispatcher.cpp#L106-L115](https://github.com/musescore/MuseScore/blob/v4.7.0/src/framework/actions/internal/actionsdispatcher.cpp#L106-L115)).
- The old `lineBreak` property is commented out in the wrapper ([elements.h#L2043](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L2043)).

### Measures per system (verified)

- `cmd("measures-per-system")` only **opens the dialog** (`openBreaksDialog` calls `interactive()->open("musescore://notation/breaks")`, [notationactioncontroller.cpp#L314](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L314), [#L1814-L1817](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L1814-L1817)).
- In 4.x the dialog creates **system locks**, not breaks ([breaksdialog.cpp#L67-L78](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/widgets/breaksdialog.cpp#L67-L78), [NotationInteraction::addRemoveSystemLocks, notationinteraction.cpp#L5989-L5997](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/internal/notationinteraction.cpp#L5989-L5997)).
- The plugin API exposes the same core function: **`curScore.addRemoveSystemLocks(interval, lock)`** (since 4.6) ([score.h#L421-L429](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L421-L429), [score.cpp#L629-L632](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.cpp#L629-L632)).
- Its behaviour ([editsystemlocks.cpp#L191-L244](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/editsystemlocks.cpp#L191-L244)) does **not** match its APIDOC:
  - It acts on `selection.startMeasureBase()` through `endMeasureBase()` (or to the end of the score). **Make a range selection first.**
  - `lock = true` locks the *existing* systems as they are and ignores `interval` (the dialog's "lock current layout").
  - `lock = false` removes the locks in the range. Then, if `interval > 0`, it adds a lock every `interval` measures. **So "N measures per system" is `addRemoveSystemLocks(N, false)`.** `addRemoveSystemLocks(0, false)` removes the locks.
- Related calls:
  - `curScore.makeIntoSystem(firstMeasure, lastMeasure)` ([score.h#L438](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L438)).
  - `system.isLocked` (read/write) ([elements.h#L2153](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L2153), [elements.cpp#L502-L513](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.cpp#L502-L513)).
  - `cmd("move-measure-to-prev-system")`, `cmd("move-measure-to-next-system")`, `cmd("apply-system-lock")`, `cmd("make-into-system")` ([notationuiactions.cpp#L646-L672](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationuiactions.cpp#L646-L672)), `cmd("toggle-system-lock")` ([#L1660](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationuiactions.cpp#L1660)).

### Layout stretch (verified)

- Property: `measure.userStretch` (`Pid::USER_STRETCH`) ([elements.h#L2067](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L2067)).
- Commands: `cmd("stretch+")` / `cmd("stretch-")` / `cmd("reset-stretch")` ([notationactioncontroller.cpp#L366-L369](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L366-L369)). `addStretch` does nothing unless the selection **is a range** ([#L1726-L1744](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L1726-L1744)).
- Global spacing comes from style keys such as `measureSpacing`, `minMeasureWidth`, `lastSystemFillLimit`, `minSystemDistance`, `maxSystemDistance` ([styledef.cpp#L58-L59](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L58-L59), [#L143](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L143), [#L230](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L230), [#L279](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L279)).

`cmd()` inside `startCmd`/`endCmd` is supported by design: `startCmd` locks the undo stack "so that all changes made by the plugin, including PluginAPI::cmd(), are committed as a single command" ([score.cpp#L591-L605](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.cpp#L591-L605)).

---

## 5. Transpose without the dialog

**Possible: partly (verified).**

- **No plugin-API transpose function.** The only transpose-related wrapper method is `staff.transpose(tick)`, which *reads* the instrument transposition ([elements.h#L2403](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L2403)).
- **`cmd("transpose-up")` / `cmd("transpose-down")`** call `Transpose::transposeSemitone(score, ±1)` ([notationactioncontroller.cpp#L473-L474](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L473-L474), [notationinteraction.cpp#L8291-L8296](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/internal/notationinteraction.cpp#L8291-L8296)). That calls `transpose(score, BY_INTERVAL, dir, Key::C, 1, trKeys=true, transposeChordNames=true, useDoubleSharpsFlats=false)` ([transpose.cpp#L371-L384](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/transpose.cpp#L371-L384)), where interval index 1 is an *augmented unison* ([interval.cpp#L127](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/interval.cpp#L127)).
- What gets transposed ([transpose.cpp#L41-L135](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/transpose.cpp#L41-L135), [#L150-L270](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/transpose.cpp#L150-L270)):
  - It acts on the **selection**. Make a range selection (`curScore.selection.selectRange(...)`) for the whole score.
  - **Chord symbols follow** (`transposeChordNames = true`): with a list selection when they are selected, and with a range selection when the chord-symbol selection filter is on.
  - **Key signatures** are transposed only with a **range** selection (`if (!rangeSelection) trKeys = false`).
  - Percussion staves are skipped.
- `cmd("pitch-up")`/`cmd("pitch-down")` (and the `-octave`/`-diatonic` variants) move selected notes only ([notationuiactions.cpp#L285-L303](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationuiactions.cpp#L285-L303)).
- `cmd("transpose")` opens the dialog ([notationactioncontroller.cpp#L315](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L315), [#L1819-L1822](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L1819-L1822)).
- **Any interval:** repeat `transpose-up` N times (spelling inferred: see open questions), or transpose in plugin code by setting `note.pitch`/`tpc1`/`tpc2` and setting chord symbols with `h.text = "<new name>"`. **Fallback outside the plugin:** the CLI option `--score-transpose <json>` ([commandlineparser.cpp#L108](https://github.com/musescore/MuseScore/blob/v4.7.0/src/app/internal/commandlineparser.cpp#L108)).
- **Key signature change (verified):**
  - `var ks = newElement(Element.KEYSIG); ks.concertKey = -2; ks.actualKey = -2; cursor.add(ks);` ([elements.h#L1068-L1071](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1068-L1071), [keysig.cpp#L245-L253](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/keysig.cpp#L245-L253)).
  - `Cursor::add` puts it into a KeySig segment at the cursor tick for the cursor's track ([cursor.cpp#L298-L303](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/cursor.cpp#L298-L303)).
  - **Inferred:** this is a plain `undoAddElement` on one staff, unlike the UI's `undoChangeKeySig`. Add one per staff, and expect no automatic handling of linked staves.
  - `curScore.keysig` and `staff.key(tick)` read the key ([score.h#L135](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L135), [elements.h#L2397](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L2397)).

---

## 6. Style

**Possible: yes for single values, no for style files (verified).**

- `curScore.style` returns an `MStyle` wrapper ([score.h#L127](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/score.h#L127)) with `value(key)`, `setValue(key, value)` and `resetValue(key)` (*since 4.7*) ([style.h](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/style.h), [style.cpp#L43-L104](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/style.cpp#L43-L104)).
- **Reading back works.** `value()` returns `PropertyValue::toQVariant()`. `setValue` on a score style is undoable (`undoChangeStyleVal`).
- **Unknown key:** `keyToSid` looks the string up in the Qt meta-enum of `Sid`. On failure it logs `Invalid style key: …` and returns `NOSTYLE` ([style.cpp#L43-L57](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/style.cpp#L43-L57)). Then `value()` returns an invalid QVariant (JS `undefined`), and `setValue`/`resetValue` **silently do nothing** (no exception, no return value). **Detection: `curScore.style.value(key) === undefined` means the key is invalid.** Do this check in the plugin before `setValue` and report an error. The current plugin calls `setValue` unchecked at `musescore-mcp-websocket.qml` line 1618.
- **List of valid keys:** `Sid` in [S/engraving/style/styledef.h](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.h) (registered with `Q_ENUM_NS(Sid)`, [#L2194](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.h#L2194)) and the defaults in [styledef.cpp](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp). The `styleDef(name, default)` macro uses the same identifier for the `Sid` and the .mss/.mscx XML tag ([styledef.cpp#L37](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L37)), so **plugin key = XML tag name**.
- **Value types:**
  - Spatium values are plain numbers.
  - Points are `QPointF`.
  - Colours are `QColor`.
  - Enums are ints.
  - **Align values are a 2-element list `[alignH, alignV]`**, using internal values (`AlignH`: LEFT 0, RIGHT 1, HCENTER 2, JUSTIFY 3; `AlignV`: TOP 0, VCENTER 1, BOTTOM 2, BASELINE 3). Write them back in the same form ([propertyvalue.cpp#L126-L129](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/types/propertyvalue.cpp#L126-L129), [#L244-L250](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/types/propertyvalue.cpp#L244-L250), [types.h#L267-L279](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/types/types.h#L267-L279)). A list of the wrong size hits an assert and the write fails.
- **Chord-symbol keys** ([styledef.cpp#L356-L408](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L356-L408), [#L493-L500](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L493-L500)):
  - Text style A: `chordSymbolAFontFace`, `chordSymbolAFontSize` (10.0), `chordSymbolAFontStyle`, `chordSymbolAAlign` (default HCENTER/BASELINE), `chordSymbolAPosAbove`/`PosBelow`, `chordSymbolAColor`, `chordSymbolAFrame*`. The same keys exist for `chordSymbolB*`.
  - `harmonyPlacement`, `minHarmonyDistance`, `harmonyHarmonyDistance`, `maxChordShiftAbove/Below`, `harmonyFretDist`.
  - **`chordSymbolSpelling`** (`NoteSpellingType`: `STANDARD` 0, `GERMAN` 1, `GERMAN_PURE` 2, `SOLFEGGIO` 3, `FRENCH` 4, [types.h#L1243-L1245](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/types/types.h#L1243-L1245); exposed as `NoteSpellingType` in QML, [qmlpluginapi.h#L413](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.h#L413)).
  - `automaticCapitalization`, `lowerCaseMinorChords`, `lowerCaseBassNotes`, `allCapsNoteNames`, `chordStyle`, `chordDescriptionFile`.
  - **`useGermanNoteNames`, `useFullGermanNoteNames`, `useSolfeggioNoteNames`, `useStandardNoteNames` are no longer `Sid`s.** They are only read from old files and converted to `chordSymbolSpelling` ("These settings were collapsed into one enum in 4.6", [style/style.cpp#L561-L573](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/style.cpp#L561-L573)). `setValue("useGermanNoteNames", …)` is silently ignored.
- **Repeat, marker and jump text keys:**
  - Left-side markers (Segno, Coda) use text style `repeatLeft*`, right-side ones (Fine, To Coda) and jumps use `repeatRight*`: `repeatLeftFontSize`/`repeatRightFontSize` (10.0), `…FontFace`, `…FontStyle`, `…Align`, `…Placement` ([styledef.cpp#L1400-L1427](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L1400-L1427); marker style choice at [marker.cpp#L98-L100](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L98-L100)).
  - Also `markerPosAbove`, `jumpPosAbove` ([#L804-L805](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L804-L805)) and `repeatMinDistance` ([#L1967](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/style/styledef.cpp#L1967)).
  - The size of the coda/segno *symbol* is the per-element `musicSymbolSize` property (default 18.0, [marker.cpp#L182-L183](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/marker.cpp#L182-L183); exposed on the wrapper, [elements.h#L664](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L664)).
- **Load/save a .mss:** no plugin API. `cmd("load-style")`/`cmd("save-style")` open a **file dialog** ([notationactioncontroller.cpp#L1841-L1878](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L1841-L1878)). Fallbacks:
  - Apply the keys one by one with `setValue`.
  - Edit `<Style>` in the .mscz XML.
  - Use the CLI `-S/--style <file.mss>` with `-o` ([commandlineparser.cpp#L115](https://github.com/musescore/MuseScore/blob/v4.7.0/src/app/internal/commandlineparser.cpp#L115)).

---

## 7. Selection and generic element properties

**Possible: yes (verified).**

### Selection API

`curScore.selection` ([selection.h#L42-L95](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/selection.h#L42-L95)):

- Properties: `elements`, `isRange`, `startSegment`, `endSegment`, `startStaff`, `endStaff`.
- Methods: `select(e, add=false)`, `selectRange(startTick, endTick, startStaff, endStaff)`, `deselect(e)`, `clear()`.

**Limit on `select`:** it refuses generated items, and accepts only notes, rests, text-based items, fret diagrams and sound flags (`UndoMacro::canRecordSelectedElement`, [selection.cpp#L73-L92](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/selection.cpp#L73-L92), [undo.cpp#L499-L508](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/editing/undo.cpp#L499-L508)). Markers, jumps, harmony, staff text and tempo text are text-based, so they can be selected. **Layout breaks, chords, spanners (voltas, hairpins), barlines and key signatures cannot** be selected this way.

### "Select all similar"

- `cmd("select-similar")` (also `select-similar-staff` and `select-similar-range`) needs **exactly one selected element** (`selection()->element()`). It selects every element matching the filter options for that element's type ([notationactioncontroller.cpp#L246-L248](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L246-L248), [#L1540-L1606](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationactioncontroller.cpp#L1540-L1606)).
- Pattern: `curScore.selection.select(oneMarker); cmd("select-similar"); var all = curScore.selection.elements;`.
- Plain-API alternative: walk the score (segments, `annotations`, `measure.elements`, `curScore.spanners`) and filter by `e.type`. This also reaches the types `select` refuses.

### Generic properties on the element wrapper

Every property is declared on the base `EngravingItem` and is backed by a `Pid`. Reading a `Pid` the element does not support returns `undefined` ([elements.h#L84-L131](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L84-L131), [scoreelement.cpp#L104-L129](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L104-L129)).

Setting a property:

- on an element already in the score: undoable, and marks the property `UNSTYLED`;
- on an element not yet added: no undo.

([scoreelement.cpp#L162-L227](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L162-L227)). Assigning `undefined`, or calling `reset`, restores the style default ([#L229-L240](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L229-L240)).

Properties most useful for this project (all in [elements.h](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h)):

| Property | Pid | Notes |
|---|---|---|
| `placement` | PLACEMENT | `Placement.ABOVE` / `BELOW` ([#L641](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L641)) |
| `hPlacement` | HPLACEMENT | [#L645](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L645) |
| `align` | ALIGN | **Write** an `Align` bitmask (`Align.HCENTER \| Align.BASELINE`, [apitypes.h#L489-L502](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/apitypes.h#L489-L502); converted in [scoreelement.cpp#L131-L157](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L131-L157), [#L207-L211](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/scoreelement.cpp#L207-L211)). **Read** returns `[h, v]` in internal values (`get` has no Align case, so it falls through to `toQVariant`). The read and write forms differ. ([#L902](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L902)) |
| `fontSize`, `fontFace`, `fontStyle` | FONT_SIZE/FACE/STYLE | [#L871-L875](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L871-L875) |
| `offset` | OFFSET | `QPointF` in **spatium units** (scaled by `spatium()` in get and set) ([#L476](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L476)) |
| `text` | TEXT | XML text with markup ([#L548](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L548)) |
| `color`, `visible`, `z`, `autoplace`, `minDistance` | | `color`/`visible`: [#L349-L355](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L349-L355) |
| `subStyle` | SUB_STYLE | text style id (`Tid` enum, [qmlpluginapi.h#L256](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.h#L256)) |
| `frameType`, `frameWidth`, `framePadding`, … | | text frame |
| `track`, `staff`, `parent`, `pagePos`, `bbox`, `type`, `name`, `subtype` | | read-only identity and geometry (`track` is writable) |

Full list of property names declared on the base wrapper (`EngravingItem`, [elements.h#L144-L1345](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L144-L1345)), extracted from source: parent, staff, staffIdx, effectiveStaffIdx, vStaffIdx, offsetX/offsetY, posX, posY, pos, pagePos, canvasPos, bbox, subtype, up, header, trailer, isMovable, enabled, addToSkyline, subType, selected, generated, color, visible, z, small, hideWhenEmpty, showCourtesy, keysig_mode, lineType, headType, headGroup, direction, stemDirection, slurDirection, barlineType, offset, play, boxHeight, boxWidth, topGap, bottomGap, margins, layoutBreakType, text, htmlText, tempo, tempoFollowText, accidentalType, placement, hPlacement, velocity, jumpTo, playUntil, continueAt, label, markerType, musicSymbolSize, markerCenterOnSymbol, lineStyle, lineColor, lineWidth, timeStretch, timesig, userLen, mag, track, harmonyVoiceLiteral, harmonyVoicing, harmonyDuration, harmonyBassScale, autoplace, playRepeats, subStyle, fontFace, fontSize, fontStyle, lineSpacing, frameType, frameWidth, framePadding, frameRound, frameFgColor, frameBgColor, align, beginText/continueText/endText (+ fonts, offsets), minDistance, harmonyType, concertKey, actualKey, positionLinkedToMaster, appearanceLinkedToMaster, textLinkedToMaster, excludeFromParts, excludeVerticalAlign, plus others for specific element types (bends, fretboards, ornaments, tremolos…).

Measure-specific properties: `repeatStart`, `repeatEnd`, `repeatJump`, `noOffset`, `irregular`, `breakMmr`, `userStretch`, `timesigActual` (and others), plus `elements`, `add`, `remove` ([elements.h#L1897-L2135](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L1897-L2135)).

---

## 8. Open / save / close

**Possible: partly (verified).**

### `writeScore` and .mscz / .mscx: does not work in 4.7 (verified)

- `mscz`/`mscx` writers *are* registered ([notationmodule.cpp#L63-L64](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/notationmodule.cpp#L63-L64)). However, `MscNotationWriter` supports only `UnitType::MULTI_PART` ([mscnotationwriter.cpp#L44-L47](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/internal/mscnotationwriter.cpp#L44-L47)).
- In `MULTI_PART` mode `exportScores` calls `writer->writeList(...)` ([exportprojectscenario.cpp#L228-L237](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L228-L237)), and `MscNotationWriter::writeList` is `NOT_SUPPORTED` and returns `Ret::Code::NotSupported` ([mscnotationwriter.cpp#L100-L104](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notation/internal/mscnotationwriter.cpp#L100-L104)).
- `doExportLoop` treats that as a write error. It shows a **modal "An error occurred while writing the file … Do you want to retry?"** dialog ([#L384-L392](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L384-L392), [#L418-L436](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/exportprojectscenario.cpp#L418-L436)). "Abort" returns `false`. **Inferred:** the output file was already opened for writing, so an empty or partial file may be left behind.
- This explains the repo note at `musescore-mcp-websocket.qml` lines 2565-2572, and the guard there is correct.

Ways to get a .mscz:

- `cmd("file-save")`, which saves the open score to its own path. Saving under a different path ("save as") would need a dialog.
- Export `.mxl` and convert it outside MuseScore.
- Edit the .mscz XML directly (the existing `src/mscz/` approach).

### `readScore(path, noninteractive=false)` (verified)

- `noninteractive = true` → **always returns `null`** ("Noninteractive flag is not yet implemented") ([qmlpluginapi.cpp#L297-L315](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.cpp#L297-L315)).
- The helper calls `projectFilesController()->openProject(file)` and then returns **`globalContext()->currentNotation()`'s score** ([engravingpluginapihelper.cpp#L52-L62](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/engravingpluginapihelper.cpp#L52-L62)).
- `ProjectActionsController::openProject` does one of the following ([projectactionscontroller.cpp#L221-L290](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/projectactionscontroller.cpp#L221-L290)):
  - if that file is already open in this window, it shows it;
  - if it is open in another window, it activates that window;
  - **if any project is open in the current window, it opens the file in a new window (new instance)**;
  - otherwise it opens the file in the current window.
- So with a score open, `readScore` returns the **old** score of the plugin's window, not the new one (inferred from the code path). **Call `closeScore()` first.**

### `closeScore()` / `closeScore(score)` (verified)

- Only the current score can be closed ([qmlpluginapi.cpp#L321-L339](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/qmlpluginapi.cpp#L321-L339)). It calls `closeOpenedProject()`, whose default `goToHome = true` opens the Home page ([engravingpluginapihelper.cpp#L64-L67](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/engravingpluginapihelper.cpp#L64-L67), [projectactionscontroller.h#L96](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/projectactionscontroller.h#L96)).
- If the project `needSave()`, a **modal "save changes?" dialog** appears ([projectactionscontroller.cpp#L692-L740](https://github.com/musescore/MuseScore/blob/v4.7.0/src/project/internal/projectactionscontroller.cpp#L692-L740)). To avoid it, save first with `cmd("file-save")`. **Inferred:** an import (for example MusicXML) counts as unsaved.

### Other write formats

`writeScore` also supports `mxl`, `musicxml`/`xml`, `mid`/`midi`, `pdf`, `png`, `svg`, `mei`, `mnx`, audio formats, `mpos`/`spos`, `lrc`, `mp4` ([registrations listed in §1](#how-writescore-works); [musicxmlmodule.cpp#L64-L65](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/musicxml/musicxmlmodule.cpp#L64-L65), [midimodule.cpp#L60](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/midi/midimodule.cpp#L60)).

---

## Open questions / needs a live test

Source reading cannot settle these. Each needs a short run against MuseScore 4.7.0 and the latest 4.7.x:

1. **PNG DPI:** what is the default value of the `export/png/resolution` preference on a fresh profile? The code default is `mu::engraving::DPI`, which is 1200 ([imagesexportconfiguration.cpp#L43](https://github.com/musescore/MuseScore/blob/v4.7.0/src/importexport/imagesexport/internal/imagesexportconfiguration.cpp#L43), [mscore.h#L70](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/dom/mscore.h#L70)). That suggests large default PNGs, but the Preferences UI may set its own value. Measure the output size.
2. **Does `writeScore(…, "mscz")` leave a 0-byte file behind** after the retry dialog is answered with Abort? Is the dialog shown when MuseScore runs headless or under xvfb in CI?
3. **Marker/jump track:** does `measure.add(marker)` without setting `track` work, or does a `nidx` track cause problems (parts, linked staves, saving)? Does `e.track = 0` work, given the source comment at [elements.h#L764](https://github.com/musescore/MuseScore/blob/v4.7.0/src/engraving/api/v1/elements.h#L764)? Recommended: `cursor.add` with `cursor.track = 0`.
4. **Plugin-created jumps:** do they play back correctly (playback reads `jumpTo`/`playUntil`/`continueAt` against marker `label`s)? Do they show up in parts? (Jumps and markers are system items, so inferred yes.)
5. **`cmd("system-break")`, `cmd("select-similar")`, `cmd("stretch+")`, `cmd("transpose-up")` from a plugin:** each has an enable check (`toggleLayoutBreakAvailable`, `hasSelection`, range selection…). Does a selection made with `curScore.selection.select/selectRange` satisfy them? Some actions are declared for the context `CTX_NOTATION_FOCUSED` ([notationuiactions.cpp#L634-L638](https://github.com/musescore/MuseScore/blob/v4.7.0/src/notationscene/internal/notationuiactions.cpp#L634-L638)). Does that matter when the plugin dispatches them? `ActionsDispatcher` checks only `canReceiveAction`, but confirm by running it.
6. **`transpose-up` spelling:** the interval is an augmented unison. Check the spellings across keys and for chord symbols, for example whether repeated steps give D# rather than Eb.
7. **Harmony text after a save and reload:** does `cmd("file-save")`, then `closeScore()`, then `readScore()` fill `h.text` with the displayed name (through `afterRead`)? Does the plugin keep running across the close and reopen?
8. **`readScore` with a score open:** confirm that it opens a new window and returns the old score.
9. **`addRemoveSystemLocks(N, false)`:** confirm it gives exactly N measures per system when the range is set with `selectRange`, and how it interacts with multimeasure rests (it walks `nextMM` when `createMultiMeasureRests` is on).
10. **Style `Align` round trip:** `style.value("chordSymbolAAlign")` should return `[2, 3]`. Confirm that writing `[0, 3]` (left, baseline) takes effect.

## Spike results on MuseScore 4.7.7 (Windows, GUI)

Run on 2026-11-14 with MuseScore Studio 4.7.7 on Windows 11, using the throwaway plugin `dev/spike/mcp-spike.qml`
(port 8766) and `dev/spike/spike_client.py`, on `dev/spike/spike-chords.musicxml` (12 bars, one chord symbol per bar,
`<words>To Coda</words>` in bar 8). These answer step 1 of [plan-notation-tools.md](plan-notation-tools.md).

| # | Question | Result |
|---|---|---|
| 1 | Does `cmd("select-similar")` see a selection the plugin made? | **Yes.** `selection.select(harmony, false)` (returns `true`, 1 element), then `cmd("select-similar")`: 12 elements selected, all `Harmony`, as many as the score has. |
| 2 | `writeScore` to PNG, and `"mpos"` | **Yes.** `writeScore(curScore, path, "png")` returned `true` and wrote one 9924×14028 px page (A4 at 1200 DPI, the default preference). `"svg"` and `"mpos"` work too. A one-page score is written to exactly `path` (no `-1` suffix). The mpos coordinates are 12 units per PNG pixel at 1200 DPI: the second system starts at x = 8503.93, which is 708.7 px, the 15 mm left margin. So a bar's box in pixels is its mpos box × dpi / 14400. |
| 3 | Marker with `cursor.add` | **Yes.** `newElement(Element.MARKER)`, `markerType = MarkerType.CODA`, `cursor.add` at the bar's first tick: the Coda sign is drawn in bar 10 and `measure.elements` returns it (`name` `Marker`, `type` 49, `markerType` 2, `label` `codab`, `text` `<sym>coda</sym>`, `track` 0). `MarkerType.CODA` = 2, `MarkerType.TOCODA` = 6. |
| 4 | Imported chord symbol names | **Empty after import, readable after save and reopen.** Straight after the MusicXML import, all 12 `Harmony` elements have `text` `""`. After Save As `.mscz`, Close and Open, `text` gives every name (`C`, `Am`, …, `Bb7`, `Gsus4`). |
| 5 | `addRemoveSystemLocks(4, false)` | **Yes.** With `selectRange` over the whole score, inside `startCmd`/`endCmd`: exactly 4 bars per system (3 systems of 4). |

Also seen:

- **The MusicXML import turns `<words>To Coda</words>` into a real marker** (`markerType` 6, TOCODA, label `coda`),
  attached to the bar before the words (bar 7). Text that only resembles it (the original session had "To Codacoda")
  stays a StaffText.
- **The images show what text data cannot:** in the first render the To Coda sign overlaps the next chord symbol.
- **The headless harness does not run on Windows.** MuseScore for Windows ships only the `windows` Qt platform plugin,
  so `QT_QPA_PLATFORM=offscreen` fails with "no Qt platform plugin could be initialized"; without it, the converter
  does not find the harness plugin in the throwaway profile (`ExtensionsProvider::action | ASSERT FAILED:
  m.actions.size() > 0`). On Windows, plugin changes are tested in the GUI; CI runs the headless tests on Linux.
