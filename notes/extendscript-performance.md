# ExtendScript performance

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), ExtendScript 4.5.6, macOS 15.7, Apple M4 Pro, CEP 12.

## Summary

| Operation | Cost |
|---|---|
| Typical DOM getter (mixed set) | about 1.5 to 1.9 microseconds |
| `layer.property("matchName")` lookup | about 5.4 us |
| `prop.value` (Position, 3D array) | about 3.1 us |
| `matchName`, `numKeys`, `numProperties` | under 0.5 us |
| Wrapping a getter in a `safe(function(){...})` closure | adds 0.7 to 0.9 us per call |
| A single inline `try { } catch` around getters | about 0 |
| A getter that makes AE throw, caught by `try/catch` | **2.6 to 3.9 ms** (about 1,000x a normal getter) |
| A plain JS `TypeError` (e.g. `.value` on `null`) | about 14 us |
| `Property.setValue` | about 30 us |
| CEP `evalScript` round trip, trivial script | about 45 ms |

## Getter cost

Method: a micro-benchmark script run with File > Scripts > Run Script File.
Each variant's loop body is compiled once with `new Function`, run as 100 blocks
of 100 iterations, and timed with `$.hiresTimer`; the median per call over
blocks is reported. The whole set ran three times in different shuffled orders
(30,000 calls per variant). Test layer: a footage layer with an unkeyed
Position. The panel was closed during the run.

Ten mixed getters (`layer.name`, `layer.property(...)`, `matchName`, `value`,
`numKeys`, `numProperties`, `propertyValueType`, `index`, ...): 14.8 us for
all ten, **1.48 us each**. One inline `try` around all ten: 14.9 us. Ten
`safe()` closures: 22.7 us, **2.27 us each**, roughly 50% slower.

A whole-project walk using one `try` per property and no closures read 20,468
properties (98,180 getters) in **185 ms**, i.e. 1.47 ms per layer and 1.89 us
per getter. A single 25-layer composition with 4,119 properties took 37 ms.
About 80% of the roughly 162 properties on an ordinary footage layer are
hidden or default groups, so pruning default groups saves real time.

Takeaway: the walk is getter-bound. Do not assume getters cost tens of
microseconds; they do not. Do not wrap each getter in a closure.

## The cost of throwing

When AE itself throws (for example `prop.keyValue(1)` on a property with no
keyframes), the exception costs **2.6 to 3.9 ms**, measured over 3,000 throws
per variant. The `try/catch` is not the expensive part; AE raising the error
is. Catching it does not make it cheap.

So the pattern that hurts is "call, let it throw, catch": one such failure
costs as much as roughly 1,000 normal getters. Guard first instead:

```js
// Guard with cheap metadata getters rather than relying on try/catch.
function firstKeyValue(prop) {
  if (prop.numKeys < 1) return null;        // 0.3 us guard
  return prop.keyValue(1);                  // never throws here
}

function setExpr(prop, text) {
  if (!prop.canSetExpression) return false; // guard, not catch
  prop.expression = text;
  return true;
}
```

Not every miss throws: `PropertyGroup.value` returns `undefined` in about
5 us instead of throwing.

## setValue

Method: 100 `position.setValue([x, y])` calls on one layer, timer read after
every call, each round in its own undo group and undone afterwards.
Result: **about 30 us per call** (100 calls take 2.9 to 3.6 ms), the same
whether a large composition or an empty one was active. Fifty opacity
`setValue` calls across four compositions took 1.4 ms in total.

## CEP evalScript round trip

Method: timed inside the panel page via the Chrome DevTools Protocol, calling
`evalScript` sequentially, 5 warm-up calls, then 50 repetitions per case.
The host stamped times with millisecond resolution to split the legs.

| Payload | p50 (AE frontmost) | p95 |
|---|---|---|
| trivial (`"x"`) | 43 ms | 55 ms |
| 1 KB | 44 ms | 52 ms |
| 100 KB | 41 ms | 52 ms |
| 1 MB | 77 ms | 88 ms |
| 3 MB | 161 ms | 172 ms |

- Floor of about **45 ms**: roughly 26 ms from panel to the start of the host
  script, **0 ms** of host execution, roughly 25 ms from host back to the
  callback. It looks like a polling tick on each side, not work.
- Payload cost is about 0.04 ms per KB above about 100 KB.
- **Concurrent calls batch into one tick:** 20 calls fired at once all
  completed within 58 ms in total. Fire independent calls together, or
  merge them into one script, rather than chaining them.
- Bringing AE to the front changed the floor little (43 ms vs 49 ms in
  background).

Consequence: a read-verify loop through CEP cannot go below about 45 ms per
round trip, however fast the ExtendScript is. For lower latency you need a
native path (see [aegp-threading-and-idle](aegp-threading-and-idle.md)).
