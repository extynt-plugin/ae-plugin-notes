# `AEGP_ExecuteScript`

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro.

`AEGP_ExecuteScript` (Utility suite) runs ExtendScript from native code. It is
a convenient way to reuse ExtendScript logic for writes while doing reads
natively.

## Cost

Method: inside the plugin on the main thread, time a call that returns
`app.project.revision` and a trivial script `1` (the floor), many repetitions,
p50.

| Dispatch | revision script p50 | floor (`1`) p50 |
|---|---|---|
| From the idle hook | about 66 to 141 us | about 137 to 206 us |
| From a run-loop block | about 10 us | about 10 us |

- Numbers vary run to run; first calls are not noticeably dearer (cold call
  128 us vs 66 to 141 us warm). A few outliers reached 2.4 to 5.4 ms.
- Results come back as UTF-8 text when you ask for non-platform encoding; the
  revision is a plain integer string.
- No dialog and no UI change from calling it.
- A batch of apply (an undo-grouped `setValue`) + native read-back +
  revision took about 0.55 to 0.6 ms of main-thread time, inside one idle turn
  (see [aegp-threading-and-idle](aegp-threading-and-idle.md)).

In short, the per-call overhead is a fraction of a millisecond; the real cost
is the idle-hook wait, not the script engine.

## Same engine as CEP

`AEGP_ExecuteScript` runs in AE's shared `main` ExtendScript engine, the same
one CEP panels use. Globals defined by one are visible to the other:

- A library loaded by a CEP panel (or by Run Script File) is already present
  when native code runs a script, and stays defined after the panel closes.
- **Last loader wins.** If two components each load their own copy of a
  library with the same global names, whichever loaded last replaces the
  other's functions. Put a version global next to the library and check it
  before use:

```js
// Library file (loaded at top level):
var MYLIB_VERSION = "1.4.0";
function myLibApply(plan) { /* ... */ }
```

```js
// Caller: reload only when absent or different.
// $.evalFile must be at the TOP LEVEL of the script string (see below).
if (typeof MYLIB_VERSION === "undefined" || MYLIB_VERSION !== "1.4.0") {
  $.evalFile(File("/path/to/mylib.jsx"));
}
```

A version guard test in practice: an old function replaced on first call; a
differing version global caused a reload; the next call found it current.

## `$.evalFile` scope inside an IIFE

`$.evalFile` evaluates in the **caller's scope**. If the script string you
pass to `AEGP_ExecuteScript` is wrapped as an immediately-invoked function,
the library's top-level `function` and `var` declarations become locals of that
function and vanish when it returns.

Test (one function and one variable in the library, same engine):

| Where `$.evalFile` is called | `typeof fn` | `typeof var` |
|---|---|---|
| inside `(function(){ ... })()` | undefined | undefined |
| at top level of the script | function | number |

Symptom: the library appears to load ("loaded" returned) but a follow-up check
reports it is not resident. It can be masked if something else (a CEP panel)
already loaded it into the shared engine. Fix: put `$.evalFile` at the top
level of the script string, and do the rest of your logic in the IIFE after it.

## First-load cost

The first `$.evalFile` of a large multi-file library took **22 to 40 ms**; one cold run showed about 77 ms. A residency
check afterwards (`typeof fn`) costs about 16 to 28 us. Load once, then check
the version global per call.
