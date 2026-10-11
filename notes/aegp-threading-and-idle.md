# AEGP threading and the idle hook

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro, plugin built universal (arm64 + x86_64).

Setting: a native AEGP plugin that serves requests from a local socket thread
(for example to an external process) and needs to read the project.

## Rule: AEGP calls belong on AE's main thread, inside an AE hook

The SDK documents `AEGP_CauseIdleRoutinesToBeCalled` as the one call safe from
another thread (acquire its suite pointer on the main thread first). Everything
else must run on the main thread. The practical dispatch pattern that works:

1. Socket/worker thread receives a request and puts it on a queue.
2. Worker thread calls `AEGP_CauseIdleRoutinesToBeCalled`.
3. The plugin's **idle hook** (registered with `AEGP_RegisterIdleHook`) drains
   the queue on the main thread, does the AEGP work, and hands results back.

```cpp
// Sketch only; suite acquisition and error handling omitted.
void worker_enqueue(Request r) {
  queue.push(std::move(r));
  g_causeIdle();            // function pointer saved on the main thread
}
A_Err IdleHook(AEGP_GlobalRefcon, AEGP_IdleRefcon, A_long *max_sleepPL) {
  while (auto r = queue.try_pop()) run_on_main_thread(*r);
  return A_Err_NONE;
}
```

## Idle-hook latency floor: about 32 to 33 ms

Method: an empty request that does nothing but go through the idle hook,
compared with an empty request answered directly on the socket thread
(transport floor, 0.28 ms). 30 runs each, p50 of client round trip.

- The empty request through the idle hook: **32.85 ms** p50. Subtracting the transport
  floor gives **about 32.5 ms** of idle-hook latency.
- It was flat across conditions: panel open with AE in the background, AE in
  front, and panel closed gave 32.5 to 32.9 ms.
- Requests sent right after a long one waited less (8 to 12 ms queued),
  which fits AE running idle routines on its own roughly **30 Hz** timer.
- `AEGP_CauseIdleRoutinesToBeCalled` does **not** wake AE early in this
  build.

Consequence: a request costs about 30 ms of queueing plus its own work, so
**batch related operations into one idle turn**. A write plus a read-back
verification as two requests cost two idle waits; one batch cost one (a batch
of apply + read + revision measured about 30 to 34 ms round trip with 0.55 ms
of main-thread time).

Measured main-thread read costs (20,485 properties, 15 comps, 126 layers):
see [aegp-stream-gotchas](aegp-stream-gotchas.md).

## What does not shorten it

### Lowering the sleep-tick value (`*max_sleepPL`)

Idea: lower the sleep-tick value the idle hook is given (to 0 or 1) while
requests are arriving. Result: AE ignores it. The empty request stayed at 32.8 to 32.9 ms
and queued p50 at about 30.5 ms in every configuration, with no CPU cost
either (30 to 31% of a core, AE's baseline). Capping `*max_sleepPL` at 30 as
a fallback is harmless but does not speed anything up.

### Dispatching from a CFRunLoop block

Idea: use `CFRunLoopPerformBlock` on the main run loop in the default mode to
run work without waiting for the idle tick. Result:

- The block does run quickly (0.2 to 0.4 ms from queue to run) and
  `AEGP_ExecuteScript` works inside it.
- **Every project read fails with a "no project" result**:
  `AEGP_GetNumProjects` reports 0 (or `AEGP_GetProjectByIndex` returns
  null). Reproducible 3 of 3 manual calls and in every benchmark run.
- A private-command variant (`AEGP_DoCommand` from a main-queue block) has the
  same problem: inside that command hook `AEGP_GetNumProjects` returned 0.

Calling suites outside an AE hook is unsupported, and here it fails silently
with wrong answers. The idle hook is the only place project reads were valid.

## Behaviour under UI load

Method: 300 requests during a supervised UI session.

- All 300 answered, no dialog, hang or crash.
- With a **menu open** (tracking), the idle hook kept running at about 30 ms.
- With a **modal dialog** open, AE ran the idle hook only about **once per
  second**; each request took about 900 ms until the dialog closed, then
  latency returned to about 30 ms.
- One observation with an Edit menu left open: queued time was 929 ms.
- Anything that starts a nested event loop on the main thread can do this;
  see [webview-panels-macos](webview-panels-macos.md).

Give requests a generous timeout (tens of seconds) and expect a stuck main
thread behind a modal dialog to be indistinguishable from a hang.
