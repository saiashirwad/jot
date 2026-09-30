# An Odin style guide for jot

Research for [ticket #7](https://github.com/saiashirwad/jot/issues/7), 2026-09-30. This is a recommended house style, not a claim that all Odin programmers agree. Domain names follow [CONTEXT.md](../../CONTEXT.md); product constraints follow [map #1](https://github.com/saiashirwad/jot/issues/1). No jot application was built.

## Decision in brief

Write a small procedural program: one application package split by subject, plain domain structs, explicit lifetime boundaries, and a thin macOS/FFI boundary. Pass `^App` into ordinary application procedures; allow one main-thread callback bridge pointer, not freely shared global state. Keep the Stack and UI on the main thread, PCM in a bounded single-producer/single-consumer ring, and worker results in an explicitly owned mailbox/channel. Use Odin's installed `core:` and `vendor:` packages before adding dependencies. Do not build an OOP framework, custom scheduler, or generic platform abstraction for v1.

These recommendations combine Ginger Bill's own [website generator][gb-code] (passed state, permanent/scratch arenas, explicit string clones), his [lifetime-based allocation article][gb-memory], Karl Zylinski's [game template][karl] (plain state and per-update temporary cleanup), and the actual Odin runtime/bindings below. They do **not** settle jot's final run-loop, renderer, or Parakeet library choice.

## 1. Packages and build: files are organization, packages are boundaries

Odin compiles directory-based packages, not a hand-maintained list of source files. Relative imports work without a collection; collections name import roots and are not package managers. See the official [package and build overview][overview]. Start with a layout like this (proposal, not files created by this research):

```text
src/                       # package jot, including main()
  main.odin                # startup and shutdown
  app.odin                 # App and state transitions
  annotation.odin          # Annotation = Excerpt + Note + Source
  stack.odin               # one ordered Stack; persistence and Hand-off policy
  capture.odin             # application-level capture orchestration
  audio.odin               # device lifetime and PCM transport
  inference.odin           # worker lifetime, jobs and result ownership
  ui.odin                  # immediate-mode pop-up drawing/input
  annotation_test.odin
  macos/                   # package macos: missing native bindings + narrow adapters
    status_item.odin
    accessibility.odin
    hotkeys.odin
    clipboard.odin
  parakeet/                # package parakeet: adapter over chosen C API
third_party/<chosen-lib>/  # pinned source/license/build instructions and native archive
build/                     # generated output, not committed
```

Keep the domain in `src` initially. Extract a pure domain package when multiple consumers or tests justify it; do not create one package per struct. Native packages must not import the application package back: pass values, opaque user data, or a narrowly typed callback. This is a jot-specific application of the [directory semantics][overview], not a language-mandated layout. The larger showcase app [Todool][todool-main] likewise groups many topical files in one application package, with a few real subpackages.

A small shell build script can create `build/`, build the selected C dependency separately, then run:

```sh
odin build src -out:build/jot -debug
odin build src -out:build/jot -o:speed
odin test src
# Only if a named import root is useful:
odin build src -out:build/jot -collection:deps=third_party
```

`import ma "vendor:miniaudio"` means the compiler's installed vendor collection, **not** `third_party/`. Keep external C declarations next to their library import in the binding package; put higher-level ownership/error adaptation above them. [Miniaudio's actual binding][ma-common] uses a relative archive path and `foreign import`; [Foundation][objc] imports system frameworks. For example, the *shape* of a future adapter is `foreign import backend "../../third_party/<chosen-lib>/lib/<archive>.a"`, followed by a `foreign backend` block of `proc "c"` declarations. Do not invent the archive name, C signatures, transitive linker flags, or dylib deployment policy before the inference decision. Preserve upstream ABI names/types in low-level bindings; expose Odin names in the adapter.

**Checked here:** a two-file package built with `odin build src -collection:deps=deps -out:probe -debug`, importing a tiny `deps:probe` binding with `foreign import libc "system:c"`. Running it printed `collection + foreign import passed`. The installed miniaudio archive also linked and ran in a separate ring-buffer test. This does not verify any Parakeet dependency.

## 2. State: data and procedures, not objects and managers

Ginger Bill's `Website` is a plain struct passed as `^Website`; Karl's `Game_Memory` is a plain struct accessed through a global pointer. Both are real Odin styles, so "no globals ever" is not a community rule. [Sources: generator][gb-code], [template][karl]. For jot prefer passed pointers because tests and thread ownership become easier to see:

```odin
// Sketch: omit platform/worker fields until their designs are settled.
Source :: struct {
	app, window_title, url: string,
	captured_at_unix_ns: i64,
}
Annotation :: struct {
	excerpt, note: string,
	source: Source,
}
Stack :: struct {
	annotations: [dynamic]^Annotation,
}
App :: struct {
	stack: Stack,
}
// Procedures such as annotation_begin(app: ^App) and hand_off(app: ^App).
```

The strings in this sketch are views; the struct alone says nothing about ownership. Document their backing storage. Keep an allocation-owning wrapper separate if using an arena per Annotation. Avoid `AnnotationManager`, inheritance, virtual-method tables and getter/setter scaffolding. Native Objective-C objects necessarily follow Apple's class model; that is a boundary constraint, not a reason to model the Stack with classes.

A callback user-data pointer is preferable to a global when the API provides one. If a main-thread Objective-C bridge needs `app: ^App`, keep it private to that boundary, establish its lifetime before registration, unregister callbacks before freeing it, and never let the audio callback mutate it. These are jot recommendations rather than properties enforced by Odin's type system.

## 3. Memory: specify a lifetime before choosing an allocator

Ginger Bill explicitly distinguishes permanent, transient/cycle, and scratch allocations [in his article][gb-memory]. His generator stores retained strings in a permanent arena and brackets scratch work with `arena_temp_begin/end` [in code][gb-code]. Apply that distinction to jot:

| Lifetime | Recommended owner and release point |
|---|---|
| App/config/model | Explicit app/backend ownership; destroy during orderly shutdown. C model handles use the C library's destroy function, not Odin `free`. |
| One accepted Annotation | Clone Excerpt, Note and Source strings into owned storage. A small arena per Annotation makes deletion/cancellation bulk cleanup; stable heap allocation for the arena-owning wrapper. Destroy after removal or successful Hand-off, consistent with persistence semantics. |
| Draft edits | Reusable builder/general allocator, or a draft arena periodically rebuilt. Do not endlessly append every keystroke/version to a long-lived arena. |
| One UI update/event batch | `context.temp_allocator` or an explicitly scoped scratch arena; reset only after all consumers finish. Never retain formatted temp strings in the Stack. |
| One inference job/result | Worker scratch dies at job end; result text must be copied to separately owned storage before publication. Receiver frees/adopts it, including stale/cancelled results. |
| Audio PCM | Preallocated ring/chunks; allocate before starting the device and release only after callback shutdown. |

Per-Annotation arenas are a recommendation, **not** mandatory Odin practice. Individually allocated strings plus one `annotation_destroy` procedure can be simpler for tiny records. A single Stack arena is viable only if individual deletion/editing does not need reclamation before the whole Stack dies. Start simple and measure.

Important implementation facts from [`core:mem/virtual`][arena]:

- A zero-initialized `virtual.Arena` can lazily grow; `virtual.arena_allocator(&arena)` holds a pointer to that arena. Do not copy/move the initialized arena or its owner through a reallocating `[dynamic]Owner`; use stable owners and store pointers/IDs in the Stack.
- This arena's `.Free` operation returns `.Mode_Not_Implemented`. Individual `delete` is not reclamation. Use `arena_temp_end`, `arena_free_all` or `arena_destroy` at the appropriate boundary; `arena_destroy` releases backing blocks.
- The implementation uses a mutex. An arena is not automatically an allocation-free or lock-free audio-callback allocator.

A tested lifetime pattern (imports: `core:mem/virtual`, `core:strings`, `core:fmt`):

```odin
arena: virtual.Arena
// This example owns the arena only for this scope.
defer virtual.arena_destroy(&arena)
a := virtual.arena_allocator(&arena)
owned := strings.clone(fmt.tprintf("Note %d", 7), a)
free_all(context.temp_allocator)
assert(owned == "Note 7")
```

`defer` runs on scope exit, not as an exception unwinder; place it directly after acquiring a resource [overview][overview]. For the UI, "per frame" means a completed drawing/event unit, not a mandatory busy game loop. AppKit can re-enter through callbacks/modal loops: resetting shared scratch inside a nested callback can invalidate the outer callback's data. Prefer local scratch marks/arenas where nesting is possible. Objective-C retain/release and autorelease pools are a **separate** lifetime system; resetting an Odin arena does not release Cocoa objects [Apple][apple-threads], [Odin pool wrapper][pool].

## 4. Errors: return values, with propagation visible

Use `(value, ok: bool)` for an ordinary yes/no result with no needed explanation. Use `(value, Error)` when callers must distinguish failure causes. Choose an enum with `.None = 0` for a small stable set; use a tagged union when different failures need different payloads (e.g. OS error versus backend error). Do not encode absence as success or make one universal string-only error. These are house rules based on the [language's multiple returns, unions and `or_return`][overview] and the [arena's real allocation/error API][arena].

**Tested enum propagation:**

```odin
Capture_Error :: enum {None, Permission_Denied}
capture :: proc(fail: bool) -> (string, Capture_Error) {
	if fail { return "", .Permission_Denied }
	return "excerpt", .None
}
propagate :: proc(fail: bool) -> (text: string, err: Capture_Error) {
	s := capture(fail) or_return
	return s, .None
}
```

Surprise worth keeping: with multiple return values, the enclosing procedure needs **named results** for `or_return`. The compiler rejected `(string, Capture_Error)` in `propagate`; naming them fixed it. It also propagates false `ok` values and non-nil error forms as documented; do not treat it as "all errors are booleans". Use an explicit `if err != ...` when translating domains or attaching details. Ginger Bill's generator uses `@(require_results)`, explicit early returns, and `or_return` in short helpers [source][gb-code].

Use `@(require_results)` on important fallible operations. Reserve `assert`/`panic` for programmer invariants and unrecoverable failures, not denied permissions, missing models, device loss or failed persistence. Odin error flow is explicit rather than exception-based [overview][overview]. For jot: failed Hand-off must not clear the Stack; surface the error at the application/UI boundary and preserve recoverable state. Cleanup on partial initialization belongs in the initializer's explicit failure path, not in an imagined destructor.

## 5. Threads: one owner for each mutable state

Recommended topology, pending the dedicated integration decision:

```text
main/AppKit thread: Stack + draft + UI + native window/pasteboard coordination
       | jobs/commands                      ^ owned result messages
       v                                    |
inference worker: backend/model state + job scratch
       ^
       | bounded SPSC PCM ring
       |
audio callback: copy incoming PCM, commit write, record overflow
```

- Use `core:thread` to create/start/join workers, and `core:sync` mutexes/condition variables or `core:sync/chan` for ordinary worker messaging. A channel does not require a custom scheduler. [`Chan` contains a mutex and condition variables][chan], so even `try_send` is **not** a promise of lock-free real-time operation.
- Prefer installed [`miniaudio.pcm_rb_*`][ma-ring] if miniaudio is chosen. Its [documentation][ma-doc] specifies **single producer, single consumer**, acquire/commit and potentially partial contiguous spans. Handle wrap/full conditions; never assume one acquire returns the requested size. Set a bounded overflow policy (drop incoming PCM and report a discontinuity is a reasonable starting proposal), rather than silently corrupting transcript continuity. Do not add a second producer/consumer without changing the design.
- No allocation, logging, AppKit messages, blocking channel operations or inference inside the audio callback. It is a narrow C ABI procedure with preallocated user data. Stop the device before freeing anything its callback uses.
- Result messages containing `string` copy the descriptor, not the bytes [string representation][overview]. Transfer an owned allocation or copy into a receiver-owned buffer; never publish worker temp memory. Include an Annotation/job generation ID so an old result cannot overwrite a newer draft. Drain/free stale messages during cancellation/shutdown.
- Publish first, then wake the main loop. A buffered channel alone does not wake AppKit. Apple documents `postEvent:atStart:` from secondary threads and main-thread event handling [source][apple-threads]; exact wake-up integration remains unprototyped here. Keep all jot UI operations on main, a deliberately stricter policy than the exceptions in Apple's historical thread-safety guide.
- Request stop, stop producers, wake blocked workers, join, drain messages, then destroy queues/backend/arenas. Never join a worker from main while that worker is synchronously waiting for main; do not make forced thread termination normal shutdown.

[`core:thread.Thread.init_context` documentation][thread] matters: leaving it unset gives the worker a fresh temporary allocator that is cleaned up on thread exit. Supplying a context makes allocator management your responsibility. Do not copy main's short-lived allocator/logger state into a long-lived worker. Reset worker scratch between jobs, not merely when the thread exits.

**Checked here:** a real worker sent `42` through a capacity-one `chan.Chan(int)`; main received it, joined/destroyed the worker, and an empty `try_recv` returned false. A separate test linked miniaudio, initialized an eight-frame mono-f32 PCM ring, wrote/read four samples and uninitialized it. This proves API/link compatibility, **not** race-freedom, callback deadlines, wraparound or overflow behavior under load.

## 6. Objective-C: imitate the installed bindings, not handwritten message-send casts

The installed [`Foundation` implementation][objc] already links Foundation/Cocoa. [`NSTypes.odin`][ns-types] privately aliases `msgSend` to `intrinsics.objc_send`; use the intrinsic in jot's added binding package, rather than relying on that private alias. [`NSString.odin`][ns-string] demonstrates `@(objc_class)`, embedded superclass storage, `@(objc_type, objc_name)` and `proc "c"` wrappers. Keep exact selector colons, ABI types and superclass relationships.

This minimal custom wrapper compiled and ran against macOS Foundation:

```odin
import "base:intrinsics"
import ns "core:sys/darwin/Foundation"

@(objc_class="NSString")
Probe_String :: struct {using _: ns.Object}
@(objc_type=Probe_String, objc_name="length")
probe_length :: proc "c" (self: ^Probe_String) -> ns.UInteger {
	return intrinsics.objc_send(ns.UInteger, self, "length")
}
```

The probe allocated `ns.String` with `initWithCString("jot", .UTF8)`, sent `length` through this wrapper, asserted `3`, released the string and drained its autorelease pool. It also checked `intrinsics.objc_find_class("NSString")` and `objc_find_selector("length")`. **`objc_find_class` takes the class name string, not the Odin type**; an initial type-argument attempt failed compilation. In contrast, `objc_send` accepts the annotated type as receiver for class messages, as Foundation's alloc wrappers show.

`@(objc_class)` describes a binding; it does not by itself implement/register a new runtime subclass. For delegates or new classes, study [`register_subclass`/`alloc_user_object`][objc-helper] and the C callback trampolines in [`NSApplication.odin`][ns-app]. Those restore an Odin `context` before calling Odin procedures. Foreign callbacks do not receive Odin's implicit context argument. Keep that bridge explicit and avoid borrowing main-thread scratch on another thread.

One sharp edge: `String_initWithOdinString` in the installed binding calls `initWithBytesNoCopy:...freeWhenDone:false` [source][ns-string]. Do not assume it copies or can outlive the Odin bytes. The probe deliberately used the copying C-string initializer. Retain/release owned Cocoa objects, use autorelease pools around Foundation work on worker threads, and clone borrowed native text before storing it in an Annotation [Apple][apple-threads]. No delegate registration, menu/status-item lifecycle, AppKit event pump or microphone permissions were tested here.

## 7. Naming, comments, file size and tests

Recommended house style, supported by the named examples rather than an asserted universal style standard:

- `Upper_Snake_Case` types (`Capture_Error`, `App`); `snake_case` procedures, locals and fields (`annotation_destroy`, `source.window_title`); uppercase genuine constants. Keep the agreed domain words, not `Clip`, `Entry`, or `Export`. Use `int` for ordinary indexing and exact ABI-sized types at native boundaries [overview][overview], [generator][gb-code].
- Tabs, braces on the declaration line, grouped imports, trailing commas in multiline structs, and localized alignment for related fields. Do not mechanically rename upstream C/Objective-C bindings; their naming is useful for comparison with headers [Foundation][ns-string], [miniaudio][ma-ring].
- Keep types beside their operations, split files by topic once navigation suffers. No arbitrary line limit: the generator keeps a cohesive pipeline together, while [Todool][todool-main] splits application concerns across files. Neither is a reason to copy an entire game/app architecture into jot.
- Comments explain ownership, thread affinity, callback reentrancy, invariants and native API quirks. At every returning-string boundary say "borrowed until …", "caller frees using …", or "owned by …". Record why a workaround exists and link its API/source. Avoid comments that only restate assignments.
- Write `@(test) name :: proc(t: ^testing.T)` and run `odin test src`; use `-all-packages` only when intentionally including imported package tests. Odin's test runner is multithreaded and tracks Odin allocator leaks/bad frees by default [official testing guide][testing]. It does not automatically audit C/Objective-C allocations or make AppKit safe on test threads.

```odin
import "core:testing"
@(test)
capture_denial_is_preserved :: proc(t: ^testing.T) {
	_, err := propagate(true)
	testing.expect(t, err == .Permission_Denied)
}
```

Prioritize pure tests for Annotation ownership, Stack ordering, persistence round-trip, XML escaping, failed Hand-off retaining the Stack, stale-result rejection and cancellation cleanup. Keep native UI/permissions smoke tests in an explicit main-thread executable. Add ring wrap/full/overflow tests and worker shutdown tests before integrating live audio. Use `mem.Tracking_Allocator` in debug ownership tests; Karl's [release entry point][karl-main] provides a concrete setup/inspection example. Do not equate lack of reported Odin leaks with correctness of foreign allocations.

## Evidence and limits

### Local checks (all throwaway code outside the repository)

Machine: macOS `arm64`; `odin version` reported `dev-2026-09:a2fb372b7`; `odin root` was `/opt/homebrew/Cellar/odin/2026-09/libexec/`.

| Probe | Result |
|---|---|
| `odin test <probes> -out:<temporary-path> -define:ODIN_TEST_FANCY=false` | Three tests passed: enum propagation + arena/scratch lifetime; real worker/channel; miniaudio PCM ring. No Odin test-runner memory warnings. |
| `odin run <objc-probe> -out:<temporary-path>` | Printed `objc length: 3`; custom annotated wrapper, class/selector lookup and manual native cleanup ran successfully. |
| Directory build + custom collection + C import | Built with `-debug`, executed successfully; two `.odin` files in the application package. |
| Negative compilation checks | Unnamed multi-results with `or_return` rejected; type argument to `objc_find_class` rejected. Corrected forms then passed. |

Temporary source and binaries were not committed. Examples above are extracted from tested patterns; the proposed domain/layout and full concurrency design are recommendations, not a compiled jot skeleton.

### What the source sample establishes—and what it does not

Read real code from Ginger Bill's generator, Karl Zylinski's template, Odin core/vendor, and the official showcase's open-source Todool. The generator supplies especially direct evidence for the requested "gingerbill style": passed state, two lifetimes, explicit cloning and early-return procedures. Karl demonstrates that global state pointers are also idiomatic; Todool demonstrates a real tool organized by topical files. Todool is older code and was **not** compiled on this toolchain; its shared worker data/forced-termination patterns are not being endorsed as jot's synchronization model.

The [official showcase][showcase] lists native macOS Monitor and JangaFX applications. Their presence is evidence of real Odin applications, not evidence of their private package layout or concurrency internals. I did not inspect JangaFX application source and make no claim to reproduce its internal style. The exact renderer, AppKit wake mechanism, inference C ABI, audio latency, signed-app/permission behavior, and production memory budgets still need their own prototype/research decisions. No jot build, dependency installation, user permission changes or live audio capture was performed.

## Primary sources

Links to repository code are pinned to revisions inspected (Odin is the installed compiler revision); web documentation was read on 2026-09-30.

[overview]: https://odin-lang.org/docs/overview/
[testing]: https://odin-lang.org/docs/testing/
[gb-memory]: https://www.gingerbill.org/article/2019/02/01/memory-allocation-strategies-001/
[gb-code]: https://github.com/gingerBill/gingerBill.org/blob/591ff149327fc036e2c821cee5f11a8db79392bf/generator/generator.odin
[karl]: https://github.com/karl-zylinski/odin-raylib-hot-reload-game-template/blob/901bb85274b76272fce317a5b7b136d592b77ea6/source/game.odin
[karl-main]: https://github.com/karl-zylinski/odin-raylib-hot-reload-game-template/blob/901bb85274b76272fce317a5b7b136d592b77ea6/source/main_release/main_release.odin
[todool-main]: https://github.com/Skytrias/todool/blob/dccef1c3a08aa3d0c9f6dafc7e9170cf0808a761/src/main.odin
[showcase]: https://odin-lang.org/showcase/
[arena]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/mem/virtual/arena.odin
[thread]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/thread/thread.odin
[chan]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sync/chan/chan.odin
[ma-common]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/miniaudio/common.odin
[ma-ring]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/miniaudio/data_conversion.odin
[ma-doc]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/vendor/miniaudio/doc.odin
[objc]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/objc.odin
[ns-types]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSTypes.odin
[ns-string]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSString.odin
[pool]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSAutoreleasePool.odin
[objc-helper]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/objc_helper.odin
[ns-app]: https://github.com/odin-lang/Odin/blob/a2fb372b76e81ef31fbbc8a2cf2b4fdf5ac6c924/core/sys/darwin/Foundation/NSApplication.odin
[apple-threads]: https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html
