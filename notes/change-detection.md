# Change detection with `app.project.revision`

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro. `app.project.revision` and `saveFrameToPng` need After Effects 17 (2020) or newer; older installs lack them.

`app.project.revision` is an integer that is cheap to read (about 66 to 141 us p50 through
`AEGP_ExecuteScript`, see [aegp-executescript](aegp-executescript.md)) and moves **only on undoable edits**. That makes it a
good "did the user change something" guard: compare it to the last value you
saw and skip re-reading the project if it is equal.

## What moves it

Method: each action was performed in the AE UI, with the revision read by a
script before and after. Values from one session:

| Action | Revision change |
|---|---|
| Edit a property value | +1 |
| Add a keyframe (stopwatch) | +3 |
| Add an effect | +2 |
| Toggle layer visibility (eye) | +1 each toggle |
| Toggle layer lock | +1 each toggle |
| Edit > Undo | +1 |
| Edit > Redo | +1 |

## What does not move it

Select or deselect layers; switch the active composition; move the time
cursor; twirl a layer open; `saveFrameToPng`; open the Render Queue; File >
Save (the `dirty` flag clears but the revision stays); clicks inside a panel;
a panel idling and reading.

`saveFrameToPng` returned in 0.44 ms; the PNG existed about 30 ms later, and
the revision was unchanged 14 seconds later too, so thumbnails are safe to
take without invalidating a cache.

## Caveats

- **Undo and Redo increase the revision**; they do not restore an earlier
  value. You cannot detect "back to a previous state" from the number alone.
- The step per user-visible edit is not 1 (keyframe +3, effect +2). Compare
  for equality, do not count edits.
- Scripts move it a lot: 20 undo-group cycles in one benchmark moved it by
  about 1,000. Record the revision after your own writes if you want to tell
  your edits from the user's.
- It tells you that something undoable changed, not what. Selection, view
  and playhead state are invisible to it, so track those separately if you
  need them.

```js
// Cheap guard: only re-read when an undoable edit happened.
var lastRevision = -1;
function projectChanged() {
  var r = app.project.revision;
  if (r === lastRevision) return false;
  lastRevision = r;
  return true;
}
```

## Property.isModified as a pruning hint

`isModified` ("changed from default") can skip default subtrees during a walk,
but it has traps; see [version-differences](version-differences.md).
