# Hosting a web view in an AEGP panel (macOS)

**Measured on:** After Effects 26.2.1 (build 26.2.1.2), macOS 15.7, Apple M4 Pro. Windows is untested. Web view: `WKWebView` inside a panel registered by an AEGP plugin, docked and floating.

Method: type a fixed key script into a textarea, a single-line input and a
contenteditable div inside the page; log every key the page and an AppKit
event monitor see; watch AE's project revision and active tab for leaks.

## AE's menu steals keys

AE's main menu key equivalents are processed before the web view sees the
event. Without help, **Cmd+A** selects all layers in AE instead of the text
field (project revision moved when it leaked), the arrow keys move AE's
selection and steal keyboard focus, and Cmd+C/V/X/Z do not reach the page.

## Fix: an NSEvent local monitor

Install `+[NSEvent addLocalMonitorForEventsMatchingMask:handler:]` for key
events. When the web view (or one of its descendants) is first responder,
route the event to the web view yourself and return `nil` to consume it.

```objc
// Sketch: route keys to the web view when it owns focus.
g_monitor = [NSEvent addLocalMonitorForEventsMatchingMask:NSEventMaskKeyDown
  handler:^NSEvent *(NSEvent *e) {
    NSResponder *r = e.window.firstResponder;
    if (![r isKindOfClass:[NSView class]] ||
        ![(NSView *)r isDescendantOf:g_webView])
      return e;                       // not ours: let AE handle it
    [g_webView keyDown:e];            // deliver to WebKit
    return nil;                       // consume so AE's menu never sees it
  }];
```

With routing on, these landed in the page with no leak into AE: Cmd+A, Cmd+C,
Cmd+V, Cmd+X, Cmd+Z, Shift+Cmd+Z, arrows, text and editing keys, and
right-click Cut/Copy/Paste. Input method composition (Japanese, committed
`日本語`) also worked through the routing.

## Home, End and Escape

- **Home and End:** the monitor sees them and routes them, but WebKit does not
  handle them in a text field by default (on macOS they scroll the document).
  The unhandled event travels back up the responder chain to AE, which
  brought its Composition tab to the front and hid the panel; later keys then
  went to AE. Fix: when the web view does not handle the editing selectors,
  perform them yourself (`moveToBeginningOfLine:`, `moveToEndOfLine:`, with
  Shift variants, and `scrollPage*:` for Page Up/Down), or fall back to
  page-level JS. After that change, Home/End/Page keys passed in docked and
  floating panels over repeated reopen cycles.
- **Escape never reaches the monitor.** No key log line was produced with
  routing on or off, and focus stayed in the field, so something in AE takes
  Escape before a local monitor runs. Consequences: Escape cannot cancel an
  IME composition (Backspace works), and Escape does not close the web view's
  right-click menu.

## Autocorrect and nested modal loops stall the idle hook

A native plugin that serves requests through the idle hook (see
[aegp-threading-and-idle](aegp-threading-and-idle.md)) stops being served
when AE is inside a nested modal event loop. WebKit's **autocorrect
suggestion bubble** did this: while the bubble showed (for example after
typing `de` offering "De"), AE sat in a nested event loop (a spell-checker
correction indicator running `_nextEventMatchingMask:untilDate:inMode:`) and
no request was answered; an empty request timed out at 60 seconds. Pressing Escape,
which dismissed the bubble, restored latency immediately (4 ms queued on the
next empty request).

Generalize this: **anything that runs a modal or nested event loop on the main
thread freezes your idle-hook work**, including alerts, pickers and context
menus that keep a loop open. A native context menu on the web view also held
the hook while open. Measured with a menu open (tracking) in AE's own menu bar,
by contrast, the hook kept running at about 30 ms; a modal dialog slowed it to
about once per second.

Mitigations:

- Disable autocorrect, spellcheck and text prediction on the page's fields
  and on the web view. With that in place no suggestion bubble appeared and no
  new `Web*` defaults were written.
- Replace JavaScript `alert`/`confirm`/`prompt` and file inputs with
  non-modal in-page UI or stubs.
- Trim the context menu (drop Spelling). AppKit still appended "AutoFill" and
  "Services" after the filter in one build, and an app-level default could
  not be set early enough inside AE to remove AutoFill.

## Emoji picker

Ctrl+Cmd+Space reaches the page (`keydown` with Cmd+Ctrl) and opens the
system picker. Choosing an emoji could not be tested.
