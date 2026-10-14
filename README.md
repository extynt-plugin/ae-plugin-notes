# ae-plugin-notes

For ExtendScript / CEP authors and AEGP plugin developers. Last measured: October 2026.

Measured facts about After Effects scripting (ExtendScript / CEP) and native
AEGP plugin development. Every number here comes from a timed run on a real
After Effects install, with the method written down so you can repeat it.
A few notes (marked "Seen on") record behaviours observed on After Effects 25.6.6 and
26.5.0 on Windows instead of timings.

No Adobe SDK files, third-party plugin data or product internals are included.
Where a claim could not be backed by a measurement, it is left out or marked.

## Trademarks and third-party software

**extynt is an independent project. It is not affiliated with, endorsed by, sponsored by or approved by Adobe Inc. or any other company named in this section.**

- Adobe, After Effects and ExtendScript are either registered trademarks or trademarks of Adobe Inc. in the United States and/or other countries. CEP, AEGP and the After Effects SDK are Adobe's extension and plug-in interfaces; they are named only to say what the notes are about.
- Apple, macOS and WebKit are trademarks of Apple Inc., registered in the U.S. and other countries. Apple Silicon and WKWebView are Apple product and API names.
- Microsoft and Windows are trademarks of the Microsoft group of companies.
- Google and Chrome are trademarks of Google LLC.
- Intel is a trademark of Intel Corporation. GitHub is a trademark of GitHub, Inc.

Nothing here is Adobe code or documentation. Adobe effect and property match names (e.g. `ADBE ...`) are referenced only to interoperate with After Effects scripting. These notes are the author's own observations and measurements, not Adobe documentation, and may be wrong for your version.

All other trademarks are the property of their respective owners. References are for identification and interoperability only.

Provided as is, without warranty. Use with Adobe software is subject to Adobe's own license terms.

The same text is in [NOTICE](NOTICE).

## Index

| Note | What it covers |
|---|---|
| [extendscript-performance](notes/extendscript-performance.md) | DOM getter cost, the price of `try/catch` and `safe()` closures, CEP `evalScript` round trip, `setValue` cost |
| [change-detection](notes/change-detection.md) | `app.project.revision` as a cheap change guard, and what does and does not move it |
| [undo](notes/undo.md) | Cost of `executeCommand` undo, undo groups, why `findMenuCommandId` labels are stale |
| [aegp-threading-and-idle](notes/aegp-threading-and-idle.md) | Which thread may call AEGP, the idle hook latency floor, what does not work (run-loop blocks, lowering the sleep-tick value) |
| [aegp-executescript](notes/aegp-executescript.md) | `AEGP_ExecuteScript` cost, the shared `main` engine, `$.evalFile` scoping, library versioning |
| [aegp-stream-gotchas](notes/aegp-stream-gotchas.md) | Expression text encoding, effect parameter 0, the `enabled` flag rule, quiet errors, native read speed |
| [webview-panels-macos](notes/webview-panels-macos.md) | Hosting a WKWebView in an AEGP panel: keyboard routing, autocorrect stalls, modal loops |
| [version-differences](notes/version-differences.md) | Behaviour that differs by AE version, and API oddities (no `isHidden` attribute, SVG import, `isModified`, new text justification, half-written frame PNGs, expression read-back, frame-grid times) |

## How the numbers were measured

- **Environment (unless a note says otherwise):** After Effects 26.2.1 (build
  26.2.1.2, ExtendScript 4.5.6), macOS 15.7, MacBook Pro with Apple M4 Pro,
  48 GB RAM, CEP 12. Test project: 135 items, 15 compositions, 126 layers,
  about 20,500 properties.
- **ExtendScript timings** use `$.hiresTimer` (microseconds). Micro-benchmarks
  compile the loop body once, run blocks of repeated calls, report the median
  per call, and repeat in shuffled order to avoid ordering bias.
- **CEP round trips** are timed in the panel page with `performance.now()`
  around `evalScript`, with warm-up calls and 50 repetitions per case.
- **Native timings** are taken inside the plugin with a monotonic clock, and
  on the client side as round-trip time. Medians (p50) are reported; p95 is
  given where it matters.
- Each note has a **Measured on** line. A number measured on one machine is a
  data point, not a law. Absolute values will shift with hardware; ratios and
  behaviours are the more portable part.

## Contributing

Measurements on other AE versions, other operating systems (Windows
especially) and other hardware are very welcome. See
[CONTRIBUTING.md](CONTRIBUTING.md).

## License

Copyright 2026 extynt.

- Text (this README, CONTRIBUTING.md and everything under `notes/`): [CC BY 4.0](LICENSE).
- Code snippets inside the text: [MIT](LICENSE-MIT).

SPDX: `CC-BY-4.0 AND MIT`.

### How to cite

> "ae-plugin-notes" by extynt, CC BY 4.0, https://github.com/extynt-plugin/ae-plugin-notes

---

Maintained by the team behind [extynt](https://extynt.com).
