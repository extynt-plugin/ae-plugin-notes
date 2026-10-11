# AEGP stream and property-walk gotchas

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro. Test project: 135 items, 15 comps, 126 layers, 20,485 properties.

Findings from comparing a native AEGP property walk (DynamicStream/Stream
suites) against an ExtendScript walk of the same project, property by property.

## Native read speed

Method: a native "read whole project" request, timed on the main thread inside
the idle hook and as client round trip (median of 30 runs, after one cold run).

| Read | Main thread | Round trip |
|---|---|---|
| Whole project, structure only | about 2.5 ms | about 32 ms |
| Whole project, every property (20,485) | **about 81 ms** | about 118 ms |
| One 25-layer comp, every property (4,120) | about 16 ms | about 29 ms |
| One property by path | about 0.3 ms | about 33 ms |

The round trip is dominated by the idle-hook wait, not the read. For
comparison, a careful ExtendScript walk of the same project took about 185 ms
(see [extendscript-performance](extendscript-performance.md)), and an earlier,
less careful ExtendScript read of the same project took 2.6 to 4.3 seconds.
The two walks did not read identical field sets, so treat the ratio as
indicative: native was faster than a tuned ExtendScript walk and far faster
than a naive one.

First (cold) whole-project read was slower: about 103 ms main thread.

## Expression text encoding

`AEGP_GetExpression` on **`AEGP_StreamSuite4`** returns 8-bit text. If you
decode it as UTF-16, ASCII becomes CJK: `//` (0x2F 0x2F) turned into the single
code unit U+2F2F, so every comment came out as garbled CJK.

Fix: acquire the **newer stream suite** (the versions with a Unicode
`GetExpression`) newest-first, and fall back to the old one only as a last
resort, decoding its bytes as UTF-8 with a Latin-1 fallback. After the fix,
all expressions in the test project matched the ExtendScript values
byte for byte (the project had no non-ASCII expression, so that part was not
exercised). Name-returning calls (layer, item, project path, stream name) use
UTF-16 handles; `GetMatchName` is a 40-char UTF-8 buffer; the
`AEGP_ExecuteScript` result and error texts are 8-bit.

## Effect parameter 0

In native walks, an effect's stream group includes the **input-layer
parameter at index 0**, which ExtendScript's `PropertyGroup` does not show.
Check whether your effect indexing starts at 0 or 1 and compare against the
ExtendScript order. In the test project every effect's parameters mapped
cleanly after accounting for it (no missing paths, and 0 of 101 layers
differed in order).

## The `enabled` rule

Comparing `enabled` between a native walk and ExtendScript exposed 100
mismatches, all on `ADBE Layer Styles / ADBE Blend Options Group`, where the
native flags were `3` (ACTIVE_EYEBALL | HIDDEN) while ExtendScript reported
`enabled = false`.

Dynamic-stream flag bits seen: ACTIVE_EYEBALL = 1, HIDDEN = 2, an
undocumented bit 8, DISABLED = 16, ELIDED = 32, SHOWN_WHEN_EMPTY = 1024.
`DISABLED` never occurred in the project, so `EYEBALL && !DISABLED` gave 100
differences. The rule that fit all 1,116 compared entries was:

```
enabled = (flags & ACTIVE_EYEBALL) && !(flags & HIDDEN)
```

(The undocumented bit appeared on 9 and 10 flag values and did not change
the outcome.) Treat this as observed behaviour on one version, not a spec.

## Quiet errors during walks

A generic walk calls suites on streams where the call is invalid: for
example `AEGP_GetExpressionState` on a stream that cannot vary over time, or a
sampling-quality call on a camera or light layer. AE reports each failed call
to the user as an error dialog even if your code ignores the return code.
In one run these dialogs stacked up and blocked AE's main thread until the run
timed out and AE had to be force-quit.

Fixes that worked, with no change to the data returned:

- Wrap request execution in `AEGP_StartQuietErrors` / `AEGP_EndQuietErrors` (with `report_quieted_errors` false)
  (Utility suite) inside the idle hook.
- Skip calls that cannot apply: only ask for expression state when the
  stream `CanVaryOverTime`; skip layer-quality calls for camera and light
  layers.

After these changes one failure remained (`AEGP_GetTrackMatteLayer` on a
camera layer) and it was reported as a warning without any dialog.
