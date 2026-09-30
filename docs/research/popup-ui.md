# Pop-up UI: native panel, immediate-mode drawing, native text services

Research for [ticket #6](https://github.com/saiashirwad/jot/issues/6), 2026-09-30. Scope: the bottom-centre pop-up containing the editable **Note**, not building jot. Terminology follows `CONTEXT.md`; focus/activation and permissions remain shared with [ticket #5](https://github.com/saiashirwad/jot/issues/5).

## Decision

**Use an app-owned nonactivating `NSPanel`, one custom `NSView`, `vendor:microui` for the small amount of UI layout/control state, and a small AppKit/Quartz command renderer. Use AppKit's text layout services (TextKit: `NSTextStorage` + `NSLayoutManager` + `NSTextContainer`) inside a custom immediate-mode Note editor. Implement `NSTextInputClient` on the view. Do not start with raylib, SDL, NanoVG or a custom Metal renderer.**

This is a recommendation under the map's **immediate-mode UI** constraint, not a claim that a complete editor already exists in `vendor:`. It minimizes competing window/event infrastructure and avoids implementing font shaping and line layout. Immediate-mode controls and redraw-on-demand are compatible: keep editor state, regenerate drawing commands on updates, and do not create a retained native control for each widget. TextKit's retained storage/layout cache is not a retained UI widget; Apple explicitly supports using it without `NSTextView`. [1][2]

**The important catch:** Odin's microui is better than the original C microui in text selection and clipboard handling, but its textbox is still single-line. No renderer turns it into the requested multiline, wrapping, IME-capable Note editor. We must write that editor integration regardless of which drawing backend we choose. [1]

If “immediate-mode” can later permit **one native editor exception**, `NSTextView` inside this same panel is substantially simpler and the best text/IME implementation. That would be a scope change, **not** the recommendation silently substituted here. Apple describes it as the normal/easiest interface to the text system, with editing, selection, wrapping and copy/paste already implemented. [2][3]

## What the candidates actually provide

| Setup | Window and run loop | Note editing / fonts | Assessment |
| --- | --- | --- | --- |
| **microui + our AppKit/Quartz renderer** | We own `NSPanel` and `NSApplication`; no extra event pump. | Custom Note editor uses native TextKit layout and `NSTextInputClient`; native font fallback/layout rather than a bitmap atlas. | **Recommended.** Write a narrow macOS adapter rather than bend a game window library around a panel. |
| **microui + raylib** | Undecorated, transparent, topmost, hidden and HiDPI flags exist. The desktop GLFW backend creates its own GLFW window; obtaining its Cocoa handle is not an API for adopting an existing panel. Polling belongs to raylib/GLFW. | Public text input is queued Unicode characters, not a marked-text/range/IME-position interface; still need custom editing and Cocoa integration. Load suitable fonts at backing resolution; the default font is not the quality baseline. | Excellent for a quick visual mock-up, not the cleanest production host here. Flags do **not** promise nonactivating editable panels. [4] |
| **microui + SDL3 Renderer** | Same basic visual flags; explicitly supports wrapping a supplied `NSWindow`/`NSView`. Supply our `NSPanel`, not an SDL-created window. SDL's own Cocoa window class is an `NSWindow`, not `NSPanel`. | Best input plumbing of these portable options: committed text, composition events, input-area placement, native Cocoa `NSTextInputClient` adapter. Still no multiline editor widget. | **Runner-up if a vendor GPU renderer is wanted.** Use the Metal `SDL_Renderer`, not SDL GPU. Native panel + SDL responder/event integration must be proven together. [5][6][7] |
| **microui + SDL3 GPU** | Same SDL window/focus issues; GPU does not solve them. | Same input path. Requires shader/pipeline/buffer/texture management instead of the 2D Renderer API. | Extra rendering machinery with no benefit established for one bar. [8] |
| **microui + our Metal renderer** | Direct `CAMetalLayer`/Metal view can live inside our panel, without a second window library. Odin has `vendor:darwin/Metal`, `MetalKit`, `QuartzCore`. | We own batching, blending, scissor, glyph textures/shaping integration and input. | Valid later optimization, not the smallest first implementation. [9] |
| **NanoVG + Fontstash** | Neither owns a window or text input. We still create the panel and graphics surface. Odin's supplied NanoVG backend is OpenGL; it is **not** a ready-made Metal backend. | Path drawing and font atlas/rasterization, not a full editor. Odin Fontstash uses stb_truetype, offers explicit fallback fonts, and leaves GPU vertex/texture management to its caller. Native shaping, grapheme editing and IME are not supplied by that atlas. | Adds a graphics backend while leaving the hard part unchanged. Do not add an outside NanoVG-Metal dependency merely to draw the bar. [10] |

The preference for native drawing is an engineering judgment about this tiny macOS-only UI, **not a measured performance comparison**. Bindings are extra work; see the concrete inventory below. All GPU choices still need correct transparent clearing/blending and rounded geometry: “borderless” alone is not “rounded”.

### Why not a different ready-made IMGUI?

There is no Dear ImGui package in the inspected installed `vendor:` tree. Adding an outside GUI library/binding would spend dependency budget not granted by the map (its outside C library allowance is for Parakeet). Even a richer editor widget would not remove the native panel requirement. Stay with the supplied microui for ordinary controls, and do not fork its single-line textbox into a second general-purpose text system. [11]

## Panel, positioning and focus

The host is independent of renderer choice:

1. Create a real **`NSPanel` subclass**, with borderless + `NSWindowStyleMaskNonactivatingPanel` at construction. Override `canBecomeKeyWindow` to return true and `canBecomeMainWindow` to return false. “Key” means receives typing; “main/frontmost app” is different. Apple documents the nonactivating mask specifically for panels and the need to override key eligibility for a borderless window. Do not put the mask on an arbitrary raylib/SDL `NSWindow` and assume equivalent behaviour. [12]
2. Use nonopaque window/content background, clear background colour, and draw a rounded opaque/translucent bar into transparent surroundings. Use floating window level for normal application windows, not screen-saver level. Hide with `orderOut:` when idle, preserving the view/editor instead of recreating them each time. `hidesOnDeactivate` needs deliberate configuration for a panel used while another app remains frontmost. [13]
3. On opening an Annotation, capture Source and the source app/window **before** changing keyboard focus. Choose the display containing that captured focused window (largest intersection if it straddles displays); fallback to the pointer's display when Source geometry is unavailable. This is a proposed explicit meaning of “active screen”, not an OS-provided universal active-monitor flag. Do not ask jot's new key window which source screen was active.
4. Re-read that `NSScreen.visibleFrame` when showing; place the panel at `x = minX + (width - panel_width)/2`, `y = minY + bottom_margin` in AppKit points. `visibleFrame` excludes Dock/menu areas and must not be cached indefinitely. Reposition on screen removal/configuration changes. [14]
5. Make the editor view first responder and explicitly make the panel key when entering text mode, **without calling app activation APIs**. For mouse entry, handle `needsPanelToBecomeKey` correctly; `becomesKeyOnlyIfNeeded` alone is not a solution. Voice-only display need not take typing focus until editing is requested. On dismiss/Hand-off, order the panel out and validate/restore the captured destination before synthetic paste; ticket #5 owns the complete sequence. [3][12]
6. Treat Spaces/full-screen as an acceptance-test item. `moveToActiveSpace` and `fullScreenAuxiliary` are relevant documented behaviours, not proof of every combination of Stage Manager, other applications' full-screen windows, and multiple displays. Do not combine `canJoinAllSpaces` and `moveToActiveSpace` indiscriminately, or equate “floating” with appearing over every system UI. [15]

**Local focus evidence is partial.** A short native probe successfully made the borderless nonactivating panel key and its custom view first responder, with the `NSWorkspace.frontmostApplication` PID unchanged. However `NSApplication.isActive` read true after event processing. Thus this research does **not** certify “jot is inactive” or real typing/IME routing across apps. Skipping launch completion left the panel non-key, so that is not a valid workaround. The full launch/focus lifecycle needs the focused prototype in ticket #5; record the source app regardless. See the exact probe results below.

## The Note editor is the real work

### Verified microui boundary

In the installed Odin source, `textbox_raw`:

- accepts UTF-8 input, tracks a selection, supports left/right, home/end, backspace/delete and clipboard callbacks;
- maps select/cut/copy/paste to its `CTRL` bit, not macOS Command semantics;
- uses one horizontal offset and one text baseline, does X-only click hit testing, and has no Up/Down key enum entries;
- handles Return by clearing focus and returning `SUBMIT`, not inserting a newline;
- has no marked/preedit range, replacement range or candidate-position protocol. The separate `text` display function's wrapping must not be confused with textbox editing. [1]

Our Odin probe inserted `first\nsecond` into the backing buffer (12 bytes) and then produced `SUBMIT`, focus 0, and the same 12-byte buffer on Return. A stored newline is not a functioning multiline editor.

### Recommended native-services adapter

Use one explicit editor state: text, selection anchor/head and affinity, marked range, scroll offset, preferred X for vertical motion, and undo/edit transactions. Keep UTF-16 ranges at the Cocoa/TextKit boundary; convert deliberately to/from Odin UTF-8 when syncing the Note. Cocoa character ranges, UTF-8 byte offsets, glyph indices and user-perceived graphemes are not interchangeable. [2][3]

Use TextKit for line wrapping, shaping/fallback, glyph placement and hit-test geometry; it works without `NSTextView` (also tested locally). Draw its glyphs into the view's native graphics context and draw selection/caret/marked-text decoration in the immediate-mode pass. Do not implement wrapping by summing per-codepoint widths from Fontstash. TextKit supplies layout, **not** the missing editor behaviour when no `NSTextView` exists. [2][3]

Implement the view's `NSTextInputClient` contract, including:

- `insertText:replacementRange:`;
- `setMarkedText:selectedRange:replacementRange:`, `unmarkText`, `hasMarkedText`, `markedRange`;
- `selectedRange`, `attributedSubstringForProposedRange:actualRange:`, `validAttributesForMarkedText`;
- `firstRectForCharacterRange:actualRange:` in **screen coordinates** and `characterIndexForPoint:`;
- `doCommandBySelector:` for editing/navigation commands.

Feed key events through `NSTextInputContext`/Cocoa key interpretation. Do not turn keycodes or `NSEvent.characters` straight into committed text: that bypasses composition. Implement the macOS editing commands, mouse/Shift selection, wrapped-line Up/Down navigation, paste/cut/copy/select-all, scrolling to the caret, and grapheme-safe deletion. Command-key equivalents need responder/menu handling in a status-item app too. Keep Return-to-save separate from the ordinary newline action; while composing, Return/Escape must reach the input method before jot treats them as save/cancel. [3]

Streaming speech updates must not overwrite an active user selection or composition. Queue/merge transcript edits at defined boundaries on the main thread; this is an application policy to specify, not a renderer feature.

### Which handles IME best?

**Native `NSTextView` wins outright**, but is a retained editor widget. Under the strict immediate-mode requirement, our native `NSTextInputClient` adapter plus native layout is the recommended route, with explicit implementation/testing cost. **SDL3 is the strongest portable input backend**: its Cocoa translator implements marked text and candidate geometry, and `StartTextInput`/`SetTextInputArea` expose the application-facing lifecycle. Consume text-editing events as provisional text, not committed Note bytes. SDL does not supply cursor movement, selection, multiline layout or paste editing policy. Raylib's character queue is a much weaker public interface for this job. [3][4][7]

## Run loop, status item and Retina

**One main-thread AppKit owner.** Run `NSApplication` on the main thread; its status item/menu and custom panel share that loop. Queue input into UI state, perform microui begin/layout/end once per update, consume the command stream while it remains valid, and request view invalidation on state changes. In `drawRect:` render the current state/commands without applying an input twice. AppKit redisplays dirty views on the event loop. Redraw on typing, transcript arrival, mouse changes, resize/backing-scale changes and a caret/recording animation timer; stop those timers when idle. Immediate-mode does not require a permanent 60 Hz poll loop. Audio/inference should only enqueue state updates, not touch AppKit objects. [16]

SDL can coexist, but it is **not just a renderer call**. Its Cocoa registration reuses an existing `NSApp`, avoids replacing an existing delegate, and has native event forwarding; its event pump also calls `nextEventMatchingMask:`. Initialize app/delegate deliberately, wrap the panel via Cocoa window/view properties, avoid activating raise paths, and integrate event draining without two independent competing pumps. A menu/status item is possible, not automatic. This source-level viability has **not** been exercised as an embedded-panel runtime test here. [6][17]

For native drawing, use point-space layout and the view's backing-aware drawing context. Recompute pixel-aligned caret/borders when backing scale changes. Native text layout avoids the supplied microui demo atlas as the font source. The local panel reported scale 2.0; that verifies a Retina environment, not subjective font-quality acceptance. [18]

For SDL/raylib/Metal alternatives, distinguish logical window size from drawable pixels, regenerate raster font resources at the actual scale, and keep measurement and drawing in the same coordinate system. A 1× bitmap enlarged to 2× remains blurry. NanoVG has a device-pixel-ratio input but Fontstash is still an atlas with explicit fonts, not AppKit's full text service. Glyph coverage/shaping, emoji, combining marks and fallback are independent of “HiDPI enabled”. [4][5][10][18]

## What we must write

1. **Native host bindings/subclasses:** extend the existing `core:sys/darwin/Foundation` surface for panel/view properties, responder/event methods, drawing context, colours/paths, screen placement and notifications. Existing `Panel` is only an empty subclass of `Window`; this is not a complete AppKit widget binding. Coordinate with ticket #5's application/status-item bindings. [11][13]
2. **Small microui renderer:** rectangle, clip, text and icon commands; rounded outer chrome outside/around microui's rectangular frame defaults; identical native text measurement/drawing metrics. Bind the necessary AppKit/Quartz operations—installed `core:sys/darwin/CoreGraphics` is not a complete CGContext drawing binding. No shader compiler, atlas uploader or Metal pipeline initially. [1][11]
3. **Note editor and text bridge:** TextKit bindings, input-client methods, ranges/selection, commands, clipboard, undo, scroll, marked text, candidate geometry and edit/voice synchronization described above. This is the largest item, not “a few key handlers”.
4. **Event/render scheduling and ownership:** durable microui context; deterministic text-service object lifetimes/autorelease pools; main-thread handover for transcription; redraw/animation only while needed.
5. **Accessibility:** custom controls/editor do not automatically get `NSTextView`'s accessibility. Bind/implement the required accessibility roles, values, selection and actions; include VoiceOver in the acceptance test. This is another reason to revisit the single-native-editor exception if scope allows.

## Checks performed on this machine

Environment: macOS **27.0 (26A428), arm64**, `odin version dev-2026-09:a2fb372b7`; `odin root` was `/opt/homebrew/Cellar/odin/2026-09/libexec/`. Throwaway files and binaries were kept outside the repository; jot was not built.

| Check | Method | Result / limits |
| --- | --- | --- |
| Vendor availability/linking | `odin run popup-libs.odin -file`; import microui separately, and SDL3, raylib, NanoVG, Fontstash; call SDL `GetVersion` and raylib `SetTraceLogLevel`. | Linked and ran. SDL runtime `3004016` = **3.4.16**; raylib bindings report **6.0**. Not a render/performance test. |
| microui textbox | `odin run popup-probe.odin -file`; initialize context with fake fixed-width metrics and root clipping, focus ID 42, inject multiline text, run frame boundary, inject Return. | `CHANGE`, 12 bytes, selection `[12,12]`; then `SUBMIT`, focus 0, unchanged 12 bytes. Confirms input/submit behaviour; source inspection establishes the single-baseline/no-vertical-navigation limitation. Compiler warned that a stack-local Context is 271,672 bytes; keep the real context durably allocated. |
| Native panel | `clang -fobjc-arc -framework AppKit popup-native.m -o popup-native`; short-lived borderless/nonactivating `NSPanel` subclass, custom `NSView` accepting first responder; order/make-key; process events for 150 ms; hide. | Final launch variant (Prohibited during `finishLaunching`, then Accessory) logged initial active=0, then key=1, first-responder=1, frontmost PID unchanged=1, scale=2.0, **isActive=1**. Accessory launch also yielded isActive=1. Omitting `finishLaunching` yielded key=0, active=0. Focus behaviour remains a prototype gate, not a proven inactive-app lifecycle. |
| Text layout without native widget | In that Objective-C probe, create `NSTextStorage`/`NSLayoutManager`/`NSTextContainer`, no `NSTextView`; system 14-point font, 110-point width, string with newline, emoji, accent and Japanese. | **5 lines, 85-point height, 64 UTF-16 units, 64 glyphs**. Confirms layout can be used separately. Does not prove visual quality, caret correctness or IME behaviour. |
| Installed binding/source audit | Read installed files and corresponding pinned upstream source. | microui selection/clipboard additions verified; no multiline editor. NanoVG's supplied backend is GL. Cocoa text/drawing bindings need extension. |

The native probe was Objective-C only to isolate Apple API behaviour without building a large Odin binding layer; production recommendation remains Odin with Objective-C runtime bindings, **not Swift or an Objective-C application rewrite**.

### Not confirmed / next acceptance probe

Before implementation is called complete, run a small **Odin panel + editor** spike: type into it while another app stays frontmost; dismiss and type back in the source app; test dead keys and Japanese/Chinese IME, marked replacement and candidate positioning, emoji/combining-mark deletion, wrapping/vertical selection, Command-V, multiple Retina/non-Retina screens, Spaces/full-screen/Stage Manager, status-menu tracking while recording, and VoiceOver. No real user typing, full IME session, embedded SDL panel, multi-display movement or graphics quality comparison was tested here. The architectural recommendation does not waive those requirements.

## Primary sources

Odin links below are pinned to the installed compiler commit, rather than assuming the original C microui or current `master` has the same behaviour.

1. [Odin microui at a2fb372b7](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/microui/microui.odin): `Key`, `Context`, `init`, `text`, `textbox_raw` (especially lines 991–1137), command types and frame lifecycle.
2. [Apple: Text System Organization](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/TextSystemArchitecture/ArchitectureOverview.html): separate storage/layout/container/view responsibilities; explicitly describes using all components except `NSTextView`.
3. [Apple: Text Editing](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/TextEditing/TextEditing.html): key-input sequence, first responder, custom text views, complete `NSTextInputClient` contract and marked text.
4. [Odin raylib bindings](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/raylib/raylib.odin); [raylib 6.0 GLFW backend](https://github.com/raysan5/raylib/blob/6.0/src/platforms/rcore_desktop_glfw.c): window flags, `GetWindowHandle`, `glfwCreateWindow`, `PollInputEvents`, `CharCallback`. Source inspection is of the desktop GLFW backend, not a claim that every possible raylib build uses GLFW.
5. [SDL: CreateWindowWithProperties](https://wiki.libsdl.org/SDL3/SDL_CreateWindowWithProperties): visual flags, high pixel density, Cocoa window/view adoption, main-thread requirement.
6. [SDL 3.4.16 Cocoa window implementation](https://github.com/libsdl-org/SDL/blob/release-3.4.16/src/video/cocoa/SDL_cocoawindow.m): `SDL3Window : NSWindow`, `Cocoa_CreateWindow`, external-window setup, floating/transparent properties and activating raise path.
7. [SDL 3.4.16 Cocoa keyboard implementation](https://github.com/libsdl-org/SDL/blob/release-3.4.16/src/video/cocoa/SDL_cocoakeyboard.m); [StartTextInput](https://wiki.libsdl.org/SDL3/SDL_StartTextInput); [SetTextInputArea](https://wiki.libsdl.org/SDL3/SDL_SetTextInputArea).
8. [SDL Renderer API](https://wiki.libsdl.org/SDL3/CategoryRender), [SDL GPU API](https://wiki.libsdl.org/SDL3/CategoryGPU): 2D renderer versus explicit modern GPU pipeline responsibilities.
9. [Installed-version Odin Darwin vendor packages](https://github.com/odin-lang/Odin/tree/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/darwin).
10. [Odin NanoVG GL backend](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/nanovg/gl/gl.odin), [NanoVG interface](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/nanovg/nanovg.odin), [Fontstash](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/fontstash/fontstash.odin).
11. [Odin vendor tree](https://github.com/odin-lang/Odin/tree/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor), [Darwin core bindings](https://github.com/odin-lang/Odin/tree/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin), [CoreGraphics binding](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/CoreGraphics/CoreGraphics.odin).
12. Apple: [nonactivatingPanel](https://developer.apple.com/documentation/appkit/nswindow/stylemask-swift.struct/nonactivatingpanel), [canBecomeKey](https://developer.apple.com/documentation/appkit/nswindow/canbecomekey), [becomesKeyOnlyIfNeeded](https://developer.apple.com/documentation/appkit/nspanel/becomeskeyonlyifneeded).
13. [Odin NSPanel](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSPanel.odin), [NSWindow](https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSWindow.odin); [Apple NSPanel](https://developer.apple.com/documentation/appkit/nspanel).
14. [Apple NSScreen.visibleFrame](https://developer.apple.com/documentation/appkit/nsscreen/visibleframe).
15. Apple: [moveToActiveSpace](https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct/movetoactivespace), [fullScreenAuxiliary](https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct/fullscreenauxiliary).
16. [Apple NSView.needsDisplay](https://developer.apple.com/documentation/appkit/nsview/needsdisplay).
17. [SDL 3.4.16 Cocoa events](https://github.com/libsdl-org/SDL/blob/release-3.4.16/src/video/cocoa/SDL_cocoaevents.m): `Cocoa_RegisterApp`, application delegation, native event forwarding and `Cocoa_PumpEventsUntilDate`.
18. [Apple: High Resolution Guidelines for OS X](https://developer.apple.com/library/archive/documentation/GraphicsAnimation/Conceptual/HighResolutionOSX/Introduction/Introduction.html): points, backing pixels, resolution-aware drawing.
