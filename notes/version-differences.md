# Version differences and API oddities

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro, for the sections up to "Other version-sensitive items". Those sections were **not tested on any other AE version**. The four sections after them (new text justification, frame PNG timing, expression read-back, frame-grid times) are observations from scripted runs against After Effects 25.6.6 and 26.5.0 on Windows (October 2026), not timed micro-benchmarks. Reports from other versions are welcome; see [CONTRIBUTING](../CONTRIBUTING.md).

## There is no `isHidden` attribute on Property or PropertyGroup

It is not documented in the scripting reference on any version, and it is not a removed feature. On 26.2.1 (the only version checked), `typeof prop.isHidden` is `"undefined"`, `"isHidden" in prop` is
false, and it is absent from `prop.reflect.properties`, for both `Property`
and `PropertyGroup`. Reading it does not throw; it silently returns
`undefined`, so code that tests `if (prop.isHidden)` quietly treats everything
as visible.

Attributes that do exist include `elided`, `active`, `enabled`,
`isSeparationFollower` and `isSeparationLeader`.

Related: a camera layer's `Opacity` property exists and is not marked hidden
by any attribute, yet `setValue` throws "property or a parent property is
hidden". Exclude camera layers from opacity writes.

## `isModified`

Method: default solid (2D and 3D) and default text layers in a scratch comp;
read `isModified` on groups, then edit, restore and re-read.

- Works: false on default groups, true after an edit, back to false when the
  value is restored to its default (it compares against the default, not
  against history).
- A **keyframe or expression makes a property read as modified even if the
  value is unchanged.** A disabled expression still counts until cleared.
- **The Transform group always reports true** on default layers, because the
  hidden `ADBE Position_0` / `ADBE Position_1` separation followers report
  `isModified = true` while Position itself is false.
- **A text layer's Text group is always true**, and `Source Text.isModified`
  stays true even after the text is restored.

Use Transform's children rather than the Transform group, and skip
`isSeparationFollower` properties. `isModified` on Material Options, Geometry
Options, Audio, Masks, Effects, Layer Styles and text path/more options groups
is a reliable "has anything been touched" test.

## SVG import

On 26.2.1, `app.project.importFile()` of an `.svg` with an `ImportOptions`
type of `COMP_CROPPED_LAYERS`:

- `canImportAs(COMP_CROPPED_LAYERS)` returns true.
- **The call returns `null` but the composition was added.** If you treat
  `null` as failure and fall back to something else, you leave AE's imported
  composition behind in the project.
- `ImportAsType.COMP` (not cropped) is rejected: `canImportAs` false and the
  import throws. `FOOTAGE` also reports importable.
- Whether an `xmlns` attribute is present made no difference across the four
  variants tried.

Handling that worked: after `importFile`, look for the newly added item(s)
instead of trusting the return value, adopt the composition, and remove
anything else the import added. Real asset files (12 to 73 KB) imported in
9 to 153 ms. A 147 KB reshaped SVG was observed to fail with an unsupported
error plus an AE stall of 13+ seconds in an earlier build of the calling code,
before the null return was handled; treat very large SVGs with caution.

```js
var before = app.project.numItems;
var ret = app.project.importFile(opts);   // may be null even on success
var added = [];
for (var i = before + 1; i <= app.project.numItems; i++)
  added.push(app.project.item(i));        // items may also be re-sorted by name
```

Note that AE re-sorts project items by name after a composition is created,
so index-based "what was added" logic and any fingerprint that depends on item
order can break; compare by identity or sort first.

## New text takes the justification last used in the Paragraph panel

**Seen on:** After Effects 25.6.6, Windows. `layers.addText(...)` created a layer that was left-justified, and a centred title laid out from the composition centre ran off the frame. On After Effects 26.5.0, Windows, the same call gave a centred layer.

`addText` (and the box and vertical variants) does not apply a fixed default. It takes the justification last set in the user's Paragraph panel, so the result depends on that install's state, not only on the AE version. A script cannot assume centred.

Workaround: after creating the layer, set `justification` explicitly on the Source Text document and write it back.

```js
var layer = comp.layers.addText("Hello");
var prop = layer.property("ADBE Text Properties").property("ADBE Text Document");
var doc = prop.value;
doc.justification = ParagraphJustification.CENTER_JUSTIFY;
prop.setValue(doc);
```

## A rendered frame PNG can exist on disk before it is complete

**Seen on:** After Effects 25.6.6, Windows. A frame saved with `saveFrameToPng` showed up as a file as soon as After Effects started writing a heavy frame (glow, blur, a 3D camera), and was complete a second or more later. A read made as soon as the file existed saw a short file; treating that as a failure also left the file behind. After Effects 26 can also write a heavy frame a second or more after `saveFrameToPng` returns.

"The file exists" is not "the frame is done". Check for a complete PNG instead: the 8-byte PNG signature at the start and a complete `IEND` chunk at the end. If the check fails, poll again (we use every 150 ms, for up to 8 s) and only then report failure. Those 150 ms and 8 s are chosen values, not measured limits.

## Reading a property that has an expression returns the expression's output

**Seen on:** After Effects 25.6.6, Windows (a Position `wiggle(...)` on a null made a write-then-read check fail; not compared across versions). `Property.value` for a property with an enabled expression is the value after the expression, so setting a value and reading it back does not return what was set. A write-then-read check on, for example, Position with `wiggle(...)` always looks like a mismatch, however correct the write was.

One way to handle it: if the read-back differs from the written value and the property's `expressionEnabled` is true, do not call that a failed write. Record the write as applied but unverified (the static value was never seen), and verify the expression itself separately (expression enabled, source text equal to what was set). Without an enabled expression, a different value is still a mismatch.

## Times are snapped to the composition's frame grid

**Seen on:** After Effects 25.6.6, Windows. A layer out point set to 12 s in a 23.976 fps composition read back as 12.012012..., and an exact-equality check on the read-back failed a correct result.

Compare times with a tolerance of half a frame, `0.5 / frameRate`, not with `===`. (Fall back to 0.5 s if the frame rate is unknown.) The same half-frame rule works for layer in, out and start times and for the work area. A real difference of one tenth of a second at 25 fps is more than half a frame (0.02 s), so it is still caught.

```js
function sameFrameTime(observed, expected, frameRate) {
  var tol = frameRate > 0 ? 0.5 / frameRate : 0.5;
  return Math.abs(observed - expected) <= tol;
}
```

## Other version-sensitive items

- `findMenuCommandId` label lookup is stale between menu openings; see
  [undo](undo.md).
- Dynamic-stream flag meanings and the effect parameter-0 layout were
  observed on 26.2.1 only; see [aegp-stream-gotchas](aegp-stream-gotchas.md).
- AEGP suite versions: request the newest available version of a suite and
  fall back; the expression text encoding changes between stream suite
  versions.
