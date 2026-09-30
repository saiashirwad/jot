# macOS glue for a bare Odin jot

Research for [ticket #5](https://github.com/saiashirwad/jot/issues/5), 2026-09-30. Scope: macOS, unsandboxed, local use; no Swift/SwiftUI, no jot implementation.

## Decision

**Bare binary in tmux: yes, as a development/run mode in the logged-in graphical session; not an unconditional promise that permissions will work under every terminal/tmux launch chain.** A bundle is not required to construct an accessory `NSApplication`, a status item, register global hotkeys, or create a keyboard-capable panel. Actual bare-binary tests, including inside a fresh tmux server, succeeded on this Mac. Microphone and synthetic input remain gated by TCC; this research did **not** obtain consent or demonstrate recording/pasting. The current execution environment reports those permissions denied. See the test ledger below.

Pick:

- **Carbon `RegisterEventHotKey`**, exclusive registrations for ⌥E, ⌥G, ⌃⌘V. It registered all three here while event-listening and event-posting permissions were false. Keep jot **unsandboxed**; there is a documented history of Option-only registration restrictions in sandboxed macOS 15 apps.
- **A nonactivating `NSPanel` subclass**, able to become key but not main, with an input-capable first responder. Capture the Source before opening it. Record a separate Hand-off target at Hand-off time; do not confuse it with the earlier Annotation Source.
- **Accessibility + Microphone** for the complete workflow. **No Input Monitoring requirement** in the chosen hotkey path. No need for Apple Events/Automation just to synthesize ⌘V.
- Start with the bare executable at a stable path. If the real terminal/tmux launch cannot obtain usable permission attribution, use a minimal `.app` launched through LaunchServices. **Ad-hoc signing does not fix permission persistence across rebuilds**; stable certificate-backed code identity is the durable fix. [Apple TN3127](https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements)

## What was actually tested

Environment: `odin version dev-2026-09:a2fb372b7`, `odin root` = `/opt/homebrew/Cellar/odin/2026-09/libexec/`; macOS **27.0 (26A428)**, arm64. SDK from `xcrun --show-sdk-path`. Tests were temporary standalone programs, not jot. No permission settings were changed, no microphone was opened, and no global synthetic keystrokes were sent.

| Test | Result | What it establishes / does not establish |
| --- | --- | --- |
| Odin bare executable imports `core:sys/darwin/Foundation`, gets `NSApplication`, sets `.Accessory` | `accessory true` | AppKit initialization works without an `.app`. |
| Odin runtime sends `systemStatusBar`, `statusItemWithLength:` with `f64(-1)`, then `button` | All three objects non-null; item removed before exit | Status-item construction works. Did not visually inspect the icon or click a menu. |
| Odin Carbon FFI, keys 14, 5, 9; Option / Option / Control+Command; exclusive flag | All returned `OSStatus 0`, non-null references; unregistered on exit | Registration works without input permissions here. Physical callback delivery/dead-key suppression was not tested. |
| Same executable launched in a new detached `tmux -L jot-research-test` server | Same six output lines as below | A tmux child can reach this graphical session's AppKit/Carbon services. Not a test of an old server originally started by a different terminal or over SSH. |
| C/Objective-C bare panel probe, including in that tmux server | Frontmost PID unchanged; panel key = 1; first responder = text view; app-dispatched key-down inserted `x` | A nonactivating panel can own keyboard focus without changing `NSWorkspace.frontmostApplication`. AppKit routing was tested with a local `NSEvent`, not `CGEventPost`. This used `NSTextView` as a probe, not as a decision about jot's immediate-mode renderer. |
| Passive permission checks | `CGPreflightPostEventAccess=false`, `CGPreflightListenEventAccess=false`; `AXIsProcessTrusted=0`; AV audio authorization status = 2 (denied) | No end-to-end mic, AX selection, or synthetic paste claim is justified from these tests. No prompts were requested. |
| Bare Objective-C executable linked with `-Wl,-sectcreate,__TEXT,__info_plist,<plist>` | `NSBundle.mainBundle.bundleIdentifier` changed from null to `dev.jot.research-glue` | Embedded Info.plist is discoverable without a bundle. Does not prove a microphone prompt will be attributed to that identity. |
| `codesign -d -r-` before/after a changed Odin build | Designated requirement changed from `cdhash H"6d9e79…"` to `cdhash H"608623…"` | Local ad-hoc code identity is build-specific, consistent with TN3127. Grant retention itself was not tested. |

Odin/tmux output (after using a proper two-`u32` struct for `EventHotKeyID`):

```text
accessory true
statusbar/item/button true true true
post/listen false false
hotkey 14 0 true
hotkey 5 0 true
hotkey 9 0 true
```

A nuance: the panel probe reported `NSApplication.isActive=1` even though the frontmost PID stayed the previous application's PID. Do not use a single `isActive` boolean as proof of where a paste will go. Verify frontmost application, focused AX element when available, and that jot's panel has relinquished key status.

Reproduction outline: compile the Odin probe with `odin build probe.odin -file -out:<temp>/probe`; initialize AppKit on the main thread, create/remove the status item, call both CG preflight functions, register/unregister the three keys. Run the binary directly and with `tmux -L <isolated-name> new-session -d '<absolute-probe> > <log> 2>&1'`. For the panel probe link Cocoa, AVFoundation and ApplicationServices, create the panel described below, pump AppKit events briefly, and send an in-process key event. All throwaway code remains outside the repository.

## Menu-bar item and run-loop requirements

Use `NSApplication.sharedApplication`, `setActivationPolicy:NSApplicationActivationPolicyAccessory`, a main-thread AppKit event loop (`run`, or a correctly integrated pump), and autorelease pools. Do not replace the UI loop with a blocking stdin/tmux loop. Audio/inference work must not block it. A terminal launch is merely a launch method, not a substitute for an application event loop. Apple's [NSApplication](https://developer.apple.com/documentation/appkit/nsapplication) owns that loop.

Accessory means no Dock icon and no **application menu bar**; it does not prohibit a **status item in the system status bar**. It can still be activated and show windows. `.Prohibited` is wrong because it forbids windows. [Activation policy](https://developer.apple.com/documentation/appkit/nsapplication/activationpolicy-swift.enum/accessory)

Hold a strong/explicitly retained reference to the status item and its target/menu for their lifetimes. Use variable length (`NSVariableStatusItemLength = -1.0`); set the button's image (template image) and title such as `3` for Stack count. Associate an `NSMenu` containing commands and Quit. Update AppKit objects on the main thread; remove the item on shutdown. These are standard [NSStatusBar](https://developer.apple.com/documentation/appkit/nsstatusbar), [NSStatusItem](https://developer.apple.com/documentation/appkit/nsstatusitem), and [NSStatusBarButton](https://developer.apple.com/documentation/appkit/nsstatusbarbutton) APIs, also declared in the installed AppKit headers.

Odin's package name is misleadingly broad: `core:sys/darwin/Foundation` links Cocoa and contains many AppKit bindings. In this installed revision `NSPanel.odin` contains only the class struct (plus modal-result enum), not a usable panel configuration surface. `NSStatusBar`/`NSStatusItem` are absent. **`Foundation.msgSend` is private**: the first probe failed to compile using it. Bind local `@(objc_class="...")` structs and use `base:intrinsics.objc_send` with correctly typed return/argument values. This worked. Runtime class creation and `class_addMethod` are already exported. [Odin source at tested revision](https://github.com/odin-lang/Odin/tree/a2fb372b7/core/sys/darwin/Foundation)

## Global shortcuts: comparison and choice

| API | Permissions | Can reserve/suppress ordinary typing? | Fit |
| --- | --- | --- | --- |
| Carbon `RegisterEventHotKey` | No Accessibility/Input Monitoring grant for the tested registrations | Designed to reserve a shortcut, rather than observe ordinary text. Successful hotkey dispatch should consume the chord before text input. See acceptance test below. | **Choose this**: narrow purpose and small C FFI. |
| `CGEventTapCreate`, `kCGEventTapOptionDefault` modifying tap | Accessibility | Yes: return null for matched events; explicitly handle key-down/up and repeats | Fallback if a required chord cannot work through Carbon on the supported OS. More machinery: CF run-loop source, tap disable/re-enable, fast callbacks. |
| `CGEventTapCreate`, listen-only | Input Monitoring | No | Does not meet the suppression requirement. |
| `NSEvent.addGlobalMonitorForEventsMatchingMask:handler:` | Apple documents Accessibility trust for global key monitoring; modern listening is also subject to input privacy | No; cannot change or prevent delivery; does not observe own-app events | Reject for shortcuts. It would leave the acute dead key/© in the target. |

Permission split comes directly from Apple's [WWDC19 security presentation](https://developer.apple.com/videos/play/wwdc2019/701/): listen-only tap → Input Monitoring; modifying tap and synthetic events → Accessibility. The [Cocoa event-monitor guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/EventOverview/MonitoringEvents/MonitoringEvents.html) explicitly rules out event suppression by a global monitor. Do not infer that all three privacy panes are mandatory merely because jot handles keyboard input.

Carbon setup (SDK `HIToolbox/CarbonEvents.h`, `Events.h`):

- `InstallEventHandler(GetApplicationEventTarget(), callback, …)` for `kEventClassKeyboard`, `kEventHotKeyPressed` (and Released if needed); retrieve the `EventHotKeyID` with `GetEventParameter`, direct object, `typeEventHotKeyID`.
- `RegisterEventHotKey(kVK_ANSI_E=14, optionKey=1<<11, …)`, G = 5, V = 9 with `controlKey=1<<12 | cmdKey=1<<8`. These are **Carbon**, not `NSEvent` or CG flag masks.
- Use `kEventHotKeyExclusive=1`, check every status, keep the references, unregister at shutdown/reconfiguration. Default registration is **not exclusive**: the SDK says multiple apps can receive the same shortcut. Exclusive registration can fail on a conflict; don't announce a working shortcut after an error.
- Registration/handler maintenance belongs on the main thread; the SDK marks registration not thread-safe. [RegisterEventHotKey reference](https://developer.apple.com/documentation/carbon/1507666-registereventhotkey)

On a US layout, ⌥E normally starts an acute-accent dead key and ⌥G types ©. While jot's registration is effective those functions are intentionally unavailable. Carbon consumes hotkeys; unlike a global monitor, it is the appropriate API for this requirement. **The local test established registration, not physical suppression.** Before calling the implementation done, type each chord with a scratch text editor focused; confirm one callback and no dead-key state/©/stray V, then unregister and confirm normal characters return. Test repeats, release order, and ⌃⌘V while a jot panel is open. Registration uses virtual key codes, so those numbers specify US physical key positions; if preferences later promise character-based shortcuts across layouts, add layout translation rather than assuming these constants mean E/G/V everywhere.

Compatibility warning: Apple's [macOS 15 discussion](https://developer.apple.com/forums/thread/763878) describes restrictions on shortcuts using only Option/Shift. The [original reproducible report](https://github.com/feedback-assistant/reports/issues/552) specifically says disabling App Sandbox restored delivery (the report is now marked Fixed). Do not generalize a sandboxed-app report into “Carbon cannot do ⌥E.” Our unsandboxed macOS 27 registrations succeeded. Still handle registration failure and qualify supported OS versions with delivery tests. If required, the fallback is an **Accessibility-authorized modifying session tap**, not silent shortcut changes or private APIs. Secure-input/system-reserved contexts can defeat global input workflows; fail visibly rather than promising “anywhere” literally.

## Focus: Source capture, Note editing, then Hand-off

App activation/frontmost ownership and a key window/first responder are different concepts. AppKit explicitly allows a nonactivating panel to take keyboard focus. [Style mask](https://developer.apple.com/documentation/appkit/nswindow/stylemask-swift.struct), [becomesKeyOnlyIfNeeded](https://developer.apple.com/documentation/appkit/nspanel/becomeskeyonlyifneeded)

Recommended setup:

1. At Annotation hotkey time, before opening any jot UI, capture `NSWorkspace.sharedWorkspace.frontmostApplication` and its PID. Through AX capture focused window/element and the available Source metadata/Excerpt. Keep retained AX references only as long as needed; other apps can close their windows at any time.
2. Create an `NSPanel` subclass with `NSWindowStyleMaskNonactivatingPanel` in its **initial** style mask. Override `canBecomeKeyWindow → YES`, `canBecomeMainWindow → NO`. Do not repeatedly toggle the nonactivating bit on an existing window.
3. Set `becomesKeyOnlyIfNeeded=NO`, `hidesOnDeactivate=NO`, appropriate floating level and positioning. `makeKeyAndOrderFront:`; `makeFirstResponder:` to the Note input view. Do **not** call `activateIgnoringOtherApps:` in this path. A custom immediate-mode view needs `acceptsFirstResponder=YES`, `needsPanelToBecomeKey=YES`, event forwarding and text-input support (`NSTextInputClient` for proper composition/IME); drawing a text box alone does not make it editable. The panel probe used a native text view solely to isolate focus from rendering.
4. On dismissal, `orderOut:` and release keyboard ownership. Include explicit Escape/save handling; don't rely only on “app resigned active” to detect dismissal of a nonactivating panel. Spaces/full-screen behavior needs explicit testing (`canJoinAllSpaces`/`fullScreenAuxiliary` as appropriate), not arbitrary activation calls.

For Hand-off, **capture the destination at the ⌃⌘V event**, not from the oldest Annotation. If jot has activated (menu interaction or an activating fallback panel), remember the last external target before that activation. Hide/order out jot UI first. If necessary activate the captured `NSRunningApplication`, then restore its captured AX window/focused element using supported AX attributes/actions where settable, and wait for actual focus confirmation before posting. Activation alone does not guarantee the correct window or text field. References may be stale, the target may have quit, and activation may be refused. Abort safely and retain the Stack if the destination cannot be established. [NSWorkspace.frontmostApplication](https://developer.apple.com/documentation/appkit/nsworkspace/frontmostapplication), [NSRunningApplication](https://developer.apple.com/documentation/appkit/nsrunningapplication), [AXUIElement](https://developer.apple.com/documentation/applicationservices/axuielement)

A minimal fallback can use an ordinary key window and explicit target restoration, but it is worse than the nonactivating-panel route. Neither route should paste while the Note box still owns keyboard focus.

## Synthetic ⌘V

Use `CGEventCreateKeyboardEvent`, `CGEventSetFlags(kCGEventFlagMaskCommand)`, and paired V down/up events (`CGKeyCode` 9), posted with `CGEventPost` into the session event stream; `CFRelease` created objects. Check `CGPreflightPostEventAccess` and request permission deliberately with `CGRequestPostEventAccess`/Accessibility onboarding, not on every key press. AX trust and posting preflight are distinct checks; don't assume one successful boolean makes all operations work. The SDK's `CoreGraphics/CGEvent.h` declares listen/post preflight and request APIs separately. [CGEvent](https://developer.apple.com/documentation/coregraphics/cgevent), [WWDC19](https://developer.apple.com/videos/play/wwdc2019/701/)

Wait for the user's Control/Command shortcut modifiers to be released before synthesizing ⌘V, so physical Control doesn't turn it into ⌃⌘V again. Avoid repost loops and keep event handling nonblocking. `CGEventPost` returns void: it is **not an acknowledgment that the destination consumed a paste**. Clipboard restoration timing and when to commit clearing the Stack need the Hand-off implementation's own policy; don't clear it merely because a function with no return value was called. No AppleScript is needed, so an Automation grant is not part of this path.

## TCC identity, tmux and packaging

TCC authorization is not simply “whatever PID called the API.” macOS determines **responsible code**. Apple's explanation cautions that this attribution matters for helpers and command-line programs; code identity matters for remembering consent. [Apple DTS: On File System Permissions, responsible-code discussion](https://developer.apple.com/forums/thread/678819), [TN3127](https://developer.apple.com/documentation/technotes/tn3127-inside-code-signing-requirements)

For a normal terminal-launched child, permissions can be attributed to the terminal host rather than the child executable. An independently launched app normally has its own identity. **Do not hard-code “grant tmux” or “grant jot” or promise all services choose the same entry.** A tmux pane is a child of its server, not of the currently attached client's terminal. Server origin, launching host, and responsibility propagation can change the result; attaching from another terminal is not a reliable way to change identity. The exact responsible identity for this machine's existing tmux server was not established by the passive checks. Grant the entry actually named by the system for each requested service, then recheck inside the actual long-lived launch path.

| Capability | Consent needed | Practical handling |
| --- | --- | --- |
| Status item/menu/panel; input delivered to jot's own view | None of these three TCC services | AppKit and graphical login session. |
| Chosen Carbon hotkeys | None for tested unsandboxed registration | Handle conflicts/failure; verify delivery. |
| AX Excerpt/Source inspection and external focus restoration | Accessibility | `AXIsProcessTrustedWithOptions` with prompt option for intentional onboarding; handle AX errors. |
| Synthetic copy/paste | Accessibility/post-event authorization | CG posting preflight; denial must leave data intact. |
| Voice Note audio capture | Microphone | Check authorization, request at first voice use, handle denial/restriction. |
| Alternative listen-only global input | Input Monitoring | Not used in chosen design. |

Microphone usage needs `NSMicrophoneUsageDescription`. Apple documents termination on an authorization/capture attempt without the required purpose string. Check `AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeAudio`, request asynchronously when not determined, marshal results back to the main thread, and only then start the audio backend. The purpose string is not itself consent. No Apple Speech Recognition permission is implied by local Parakeet inference. [Apple media authorization guide](https://developer.apple.com/documentation/bundleresources/requesting-authorization-for-media-capture-on-macos)

A bare Mach-O can embed Info.plist in `__TEXT,__info_plist`; this was checked locally with clang. Include a stable identifier and purpose string rather than relying on a nearby loose plist. Odin linker-flag integration was **not** tested. Embedding metadata does not force TCC to disregard a responsible terminal host. No direct microphone request was made here because the status was already denied and unattended consent would not demonstrate a successful launch path.

**When to use the `.app` fallback:** if the real tmux chain cannot present/obtain the needed permissions, attributes them confusingly, or the user doesn't want to grant broad Accessibility/Microphone rights to a general-purpose terminal. Package the same Odin binary as `Jot.app/Contents/MacOS/jot`, with `Contents/Info.plist` containing `CFBundleExecutable`, stable `CFBundleIdentifier`, name/package type, `LSUIElement=true`, and the microphone purpose string. Sign the completed bundle and launch via `open /absolute/path/Jot.app` (LaunchServices), rather than directly executing `Contents/MacOS/jot` as another terminal child. This is packaging, not a Swift rewrite. Keep logs available to tmux, but the app is then independently launched, not literally its foreground child.

For local use an ad-hoc signature (`codesign --force --sign - Jot.app`) is a reasonable starting point, **not a solution to rebuild churn**. Apple TN3127 explicitly says ad-hoc designated requirements are tied to a particular version; the observed Odin `cdhash` requirements changed after rebuilding. When the terminal is the responsible identity, rebuilding a child may leave its terminal grants usable, but that is not a portable guarantee. For persistent independent jot grants, use consistent certificate-backed signing and a stable identifier/path; use Developer ID/notarization for distribution as appropriate. If Hardened Runtime is enabled, add the audio-input entitlement; do not enable App Sandbox as an incidental consequence of packaging. [Audio input entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.device.audio-input)

## Bindings to write (and existing pieces to reuse)

Use thin, typed platform bindings, not another external C library. Framework linking does not consume the one outside inference-library allowance.

| Surface | Required bindings / reuse |
| --- | --- |
| Status item | New Objective-C class wrappers for `NSStatusBar`, `NSStatusItem`, `NSStatusBarButton`; `systemStatusBar`, `statusItemWithLength:`, `removeStatusItem:`, `button`, `setMenu:`, button `setTitle:`, `setImage:`, tooltip/accessibility label as needed. Reuse `NSMenu`, `NSMenuItem`, `NSImage` methods where present; custom target/action callbacks through runtime methods. |
| App/panel/view | Reuse Application/Window/View classes, style-mask constants, window first-responder calls and runtime subclass helpers. Add missing NSPanel property sends, key/main overrides, responder/text-input methods for chosen renderer. Manage retain/release/autorelease and C callback contexts explicitly. |
| Carbon | `EventHotKeyID` = struct of two `u32`; opaque `EventHotKeyRef`, event/handler/target refs; `EventTypeSpec`; `RegisterEventHotKey`, `UnregisterEventHotKey`, `InstallEventHandler`, `RemoveEventHandler`, `GetApplicationEventTarget`, `GetEventParameter`, event/key/modifier constants. C ABI callbacks and `OSStatus` are not Odin-context callbacks. |
| Quartz posting | Opaque CG event/source refs; `CGKeyCode` (`u16`), flags (`u64`), enum/mask types; create keyboard event, set flags, post, posting preflight/request, key-state query if used to wait for modifier release. CoreFoundation release. Tap bindings only if fallback is needed. |
| Source/target/focus | Missing `NSWorkspace`/`NSRunningApplication` wrappers, frontmost app/PID/activation and notifications; AX trust/prompt, create application element, copy/set attributes, query settable, perform action, error/constants, CF ownership. Reuse installed CoreFoundation types. |
| Microphone consent | AVFoundation framework, `AVCaptureDevice` class authorization/request methods, `AVMediaTypeAudio`, authorization enum, completion block taking BOOL. Odin has `NSBlock.odin` helpers including a one-parameter callback; verify lifetime and callback signature rather than passing an Odin closure directly. Audio streaming bindings are a separate research concern. |

Installed source audit references: [NSApplication.odin](https://github.com/odin-lang/Odin/blob/a2fb372b7/core/sys/darwin/Foundation/NSApplication.odin), [NSPanel.odin](https://github.com/odin-lang/Odin/blob/a2fb372b7/core/sys/darwin/Foundation/NSPanel.odin), [NSTypes.odin](https://github.com/odin-lang/Odin/blob/a2fb372b7/core/sys/darwin/Foundation/NSTypes.odin), [objc.odin](https://github.com/odin-lang/Odin/blob/a2fb372b7/core/sys/darwin/Foundation/objc.odin), [NSBlock.odin](https://github.com/odin-lang/Odin/blob/a2fb372b7/core/sys/darwin/Foundation/NSBlock.odin). Local SDK headers are the ABI authority; API documentation is not a substitute for copying exact widths, struct layout and calling conventions.

## Remaining implementation acceptance checks

These are validation tasks, not reasons to guess that permissions succeeded:

1. On the user's **actual** terminal + existing tmux server, obtain Accessibility and Microphone deliberately; note which named identities are granted. Verify recording and synthetic paste in a disposable editor. Repeat after a changed build and restart, and after terminal/tmux restart.
2. Verify Carbon delivery and swallowing of all three exact chords, particularly ⌥E's dead-key state, with jot inactive and its panel open. Include conflicts and unsupported/secure-input cases. No fallback to a global monitor.
3. Verify physical typing, IME/dead-key composition, Escape, focus release, same-app multiple windows, app switching during a Note, and full-screen/Spaces with the chosen immediate-mode view. The native-view probe does not certify a future renderer.
4. Confirm destination identity before Hand-off and preserve the Stack on focus/permission failure. The Annotation Source is not the Hand-off destination.
5. If bare attribution fails, repeat with the minimal independently launched `.app`. Do not describe ad-hoc signing as persistent identity or reset unrelated applications' TCC grants during debugging.
