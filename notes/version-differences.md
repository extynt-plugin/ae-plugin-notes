# Version differences and API oddities

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro. **Not tested on any other AE version** (an AE 25 install was not available). Reports from other versions are welcome; see [CONTRIBUTING](../CONTRIBUTING.md).

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

## Other version-sensitive items

- `findMenuCommandId` label lookup is stale between menu openings; see
  [undo](undo.md).
- Dynamic-stream flag meanings and the effect parameter-0 layout were
  observed on 26.2.1 only; see [aegp-stream-gotchas](aegp-stream-gotchas.md).
- AEGP suite versions: request the newest available version of a suite and
  fall back; the expression text encoding changes between stream suite
  versions.
