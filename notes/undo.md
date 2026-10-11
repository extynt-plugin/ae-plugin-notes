# Undo from scripts

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro.

## Cost

Method: an undo group wrapped around the operations, then
`app.executeCommand(16)` (Edit > Undo) timed with `$.hiresTimer`. Ten
repetitions per case; state verified as restored after each.

| Case | begin group | operations | end group | undo |
|---|---|---|---|---|
| 1 operation (rename a layer) | about 10 us | about 38 us | about 6 us | **about 5.2 ms** (4.5 to 11 ms) |
| 50 operations (opacity `setValue` on 50 layers, 4 comps) | about 8 us | 1.4 ms (28 us each) | about 10 us | **about 4.8 ms** (4.3 to 5.7 ms) |

- Undo costs about 5 ms **regardless of group size** from 1 to 50 operations.
- The first two undos in a session were slower (about 10 ms), later ones
  settled near 4.5 ms.
- `beginUndoGroup` / `endUndoGroup` themselves are essentially free.
- Command id 16 is "Undo" for `executeCommand`. `findMenuCommandId("Undo")`
  returns a different id (2371 in this build, possibly because the live menu title is "Undo <action>"); do not use it to find the
  undo command.

```js
app.beginUndoGroup("My tool: rename layers");
try {
  // ... edits ...
} finally {
  app.endUndoGroup();
}
// later, to undo the whole group in one step:
app.executeCommand(16);
```

## `findMenuCommandId` undo labels are stale

Question tested: can a script prove that its own undo group is on top of the
undo stack by looking up the Edit menu label with `findMenuCommandId`?
**Not reliably.**

Method: begin a group titled with an em dash, rename a layer, end the group;
query `findMenuCommandId` with the expected "Undo ..." label at each stage,
while also reading the real Edit menu on screen.

- `findMenuCommandId` matches Edit menu titles **as AE last built them**, and
  AE rebuilds the titles only when the menu is opened.
- Right after your own undo group the lookup returns 0. After the Edit menu
  has been opened once it returns 16.
- After the user then makes a different edit, the lookup **still returned 16
  (stale)** until the menu was opened again; once opened, it returned a
  different, negative value and the label for the new edit matched instead.
- Matching is exact. A group named with an em dash only matches that exact
  character; en dash and hyphen variants never matched.

If you need safe "undo only my work" behaviour, do not depend on the menu
label. Check a state guard instead (for example the project revision and a
fingerprint of what you changed) before issuing the undo command.
