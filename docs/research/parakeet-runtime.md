# Parakeet Unified from Odin on Apple Silicon

Checked 2026-09-30. Resolves [jot #3](https://github.com/saiashirwad/jot/issues/3), under [map #1](https://github.com/saiashirwad/jot/issues/1). Research only: no jot implementation, microphone recording, or throwaway code committed.

## Decision

**Use direct Core ML from Odin, reusing Sendpoint's demonstrated pipeline, with the INT8 320 ms Unified encoder and published Core ML preprocessor.** No Swift or Python is needed at runtime. Odin can invoke the Objective-C API using its existing Foundation/Objective-C machinery; this is independently rerun here, not merely an FFI conjecture. Keep **transcribe.cpp / Metal / Q8_0** as the one-external-C-library fallback if owning the small Core ML orchestration layer proves undesirable.[S], [P], [10]

This updates the initial transcribe.cpp preference after the user supplied [Sendpoint's prior research at `4b753c2`](https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt.md). Reuse its work rather than porting RNNT/mel code again: the 174-line pure-Odin prototype already covers the Core ML preprocessor, encoder, predictor, joint decision, vocabulary, partial text and final flush. We inspected that code and its raw results, verified all 15 existing cache files against its manifest, compiled it unchanged with the installed Odin, and ran real audio through it.[S], [P], [M]

**Why this choice:** on the same M5, same 11 s audio and same 320 ms `(left,chunk,right)=(5600,160,160)` geometry, direct Core ML took **1.14–1.16 s**, versus **2.48 s** summed compute for transcribe.cpp. Both transcribed the words correctly; final punctuation differed. Core ML also avoids building an outside inference library. This is a narrow local comparison, not a corpus-wide claim that it is universally fastest. Direct Core ML does require productionizing the reused Odin orchestration, whereas transcribe.cpp supplies that lifecycle behind a C API.

### Constraints and differences from Sendpoint

- jot permits **at most one outside C library**; the selected route uses **zero**, because Core ML/Foundation/Objective-C are Apple system frameworks and the host logic is Odin. “One allowed” is treated as a ceiling, not an obligation to add a dependency. The fallback uses one vendored ASR package including its bundled ggml implementation; it is not literally one independently authored codebase.
- jot's application code remains Odin, with **no Swift/SwiftUI**. Sendpoint's optional FluidAudio Swift dylib/helper is **not** carried forward. Core ML does not require Swift.
- jot uses `vendor:miniaudio` capture rather than redoing Sendpoint's CoreAudio HAL/AVAudioEngine research. This also avoids the two-argument Objective-C block adapter that AVAudioEngine taps need.
- jot uses immediate-mode UI, not Sendpoint's AppKit text-widget implementation. Reuse worker ownership/cancellation ideas, but publish copied Note snapshots to jot's UI.
- Sendpoint already has a FluidAudio model cache. jot must install its own pinned, integrity-checked assets in **`~/.cache/jot`**, not depend on that sibling app's installation. Existing cache was read-only during this research.
- Retain the Apache-2.0 attribution/license when adapting Sendpoint's FluidAudio-derived orchestration. Do not assume converted weights have been relicensed: see the license discrepancy below.[S], [P]

## What was measured here

Machine: Apple **M5**, 32 GB, macOS arm64, Odin Homebrew **2026-09** (`odin root`: `/opt/homebrew/Cellar/odin/2026-09/libexec/`). Shared workstation, not controlled for thermal/power/competing work. Tests used transcribe.cpp's real **11.000 s, 16 kHz mono `samples/jfk.wav`**, converted losslessly from PCM16 to normalized float32 for the Core ML prototype.

| Path | Geometry / precision | Result |
|---|---|---|
| **Odin → Core ML**, unchanged Sendpoint prototype | 320 ms, 70/2/2 INT8 encoder; Core ML preprocessor | Two runs: decode **1155.257 / 1141.356 ms** (~9.5–9.6× real-time), tail **17.708 / 16.617 ms**; load **7235.480 / 133.567 ms** |
| transcribe.cpp Metal | Same 320 ms geometry, Q8_0 | **2479.4 ms** summed mel/encode/decode (~4.4× real-time); load 351.21 ms |
| transcribe.cpp Metal | 1120 ms, 70/7/7 Q8_0 | **1123.6 ms** summed compute (~9.8×); load 220.76 ms |
| transcribe.cpp Metal | 160 ms, 70/1/1 Q8_0 | **5432.3 ms** summed compute (~2.0×); load 356.92 ms |
| transcribe.cpp Metal offline | Q8_0, `-r 3`, no timestamps | Final warm iteration **89.5 ms / 123×**; not a three-run average |

Direct Core ML's final transcript:

> And so, my fellow Americans ask not what your country can do for you. Ask what you can do for your country.

The 320 ms transcribe.cpp result had the same words without the last period. Both yielded incremental output and successful finalization. These are **file-fed throughput tests**, not microphone or key-release-to-visible-text latency. Direct timing includes host loop/logging; transcribe.cpp reports summed inference components, so the timing scopes are not perfectly identical. The direct route remains faster in this limited same-geometry comparison despite its broader timing scope.

Cold-start warning: first transcribe.cpp Metal initialization took **15.1 s**, mainly embedded shader compilation (subsequent cached loads ~0.2–0.36 s). Core ML's first rerun here took **7.24 s** to load, then 134 ms. Prepare/warm models before recording and display readiness; do not confuse warm throughput with startup latency.

### Prior evidence reused, not rerun unnecessarily

Sendpoint's raw [direct-stream result](https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt/results/direct-stream.txt) records 797.300 ms decode / 16.868 ms tail for a different 7.435 s clip. Its report gives three-run direct range 764–803 ms. Its sherpa 240 ms INT8 results were **23.75–26.88 s CPU** and **34.17–36.46 s packaged CoreML provider** on that clip, i.e. slower than real time. We read the prior report/results and checked our sherpa source/export/API observations against them; we did not redownload/rebenchmark that already-established losing preset. These do not rule out other thread counts, larger chunks or newer CoreML provider options.[S]

Sendpoint's low warm process RSS is **not total model memory**: Core ML/ANE caches/services may live outside the process. Neither report establishes total system RAM, idle power, ANE placement, or sustained microphone performance. Do not claim those from compute-unit configuration or model file size.

## Streaming: yes, but buffered, not cache-aware

The exact model is **Unified FastConformer-RNNT**, not Parakeet TDT v2/v3 or Nemotron. It recomputes a left/chunk/right audio window with chunked attention and dynamic chunk convolution, while retaining RNNT state. NVIDIA's 160–2080 ms is **chunk + right context**, not computation or end-to-end latency.[5]

The **selected Core ML export is genuinely streaming at 320 ms**: 70/2/2 encoder frames, 80 ms each. The compiled encoder has fixed geometry; changing latency means selecting/reconverting the appropriate encoder, not changing a host integer. The conversion supports configured contexts and FluidAudio publishes 320/640/1120/2080 ms tiers. Other tiers, including a new 160 ms Core ML export, were **not tested here**. The selected route is not limited to offline transcription.[10], [11], [P]

Reused pipeline:

1. 16 kHz mono f32 PCM → Core ML preprocessor, CPU-only, `audio_signal [1,N]`, `audio_length [1]` → 128-bin mel and length. Its dynamic input supports the 94,720-sample window, so **no new Odin FFT/mel implementation is needed**.
2. INT8 streaming encoder, **CPU+ANE allowed**, fixed mel `[1,128,593]` → encoder `[1,1024,75]`. Allowing ANE does not prove per-op execution placement. Avoid INT8 GPU/`.all`: FluidInference documents an MPSGraph failure.[10]
3. CPU RNNT predictor: token and persistent `h/c [2,1,640]`, blank ID **1024**. CPU joint decision takes one encoder step plus predictor output, returns next token. Blank advances time; nonblank updates recurrent state and emits a piece; cap 10 symbols/frame.
4. Vocabulary pieces map `▁` to spaces. Retain left context; first advance 5120 samples, subsequent advances 2560, with two future encoder frames withheld. On finish flush once, including exact-boundary endings. The existing prototype has a narrow exact-boundary regression in Sendpoint; do not treat that as a complete lifecycle test suite.[S], [P]

**Model-card inconsistency found and tested:** NVIDIA's 560 ms recommendation uses chunk 160 + right 400 ms, i.e. `(70,2,5)` frames. The checkpoint's listed right-context menu is `{0,1,2,3,4,7,13}`. transcribe.cpp rejects `(70,2,5)` with `invalid argument`, confirmed on the real GGUF. Sherpa's exporter explicitly generates that tuple anyway. Therefore do not promise that every published latency row is interchangeable across runtimes. Selected 70/2/2, fallback 70/7/7 and 70/1/1 all ran successfully.[1], [3], [5], [6]

## Converted models and conversion recipe

### Selected Core ML assets

Use the [FluidInference published repository](https://huggingface.co/FluidInference/parakeet-unified-en-0.6b-coreml), not a runtime `.nemo` loader. For the 320 ms route install only:

- `parakeet_unified_encoder_streaming_70_2_2_int8.mlmodelc/`
- `parakeet_unified_decoder.mlmodelc/`
- `parakeet_unified_joint_decision_single_step.mlmodelc/`
- `vocab.json`, model/config metadata
- `parakeet_unified_preprocessor.mlmodelc/`

The first group of 15 cached files totals **608,330,968 bytes**, verified against Sendpoint's per-file SHA-256 [manifest], [M]; the encoder's weight file alone is **589,486,784 bytes**. The additional preprocessor is about 623 KB. We independently downloaded the preprocessor from revision **`d32e972dd4315f1dc3f6be28fb2aab0ab3e80358`** and reran it. The original encoder-cache download revision remains unconfirmed; pin its **verified hashes**, and resolve/validate the corresponding published files before making a production installer. Do not imply the cached encoder was proved to come from that preprocessor revision.[S], [M]

If conversion is necessary, follow the [pinned Mobius recipe], [10], not generic NeMo export. In its model-specific directory:

```sh
uv sync
uv pip install --no-deps --force-reinstall \
  'nemo_toolkit @ git+https://github.com/NVIDIA-NeMo/NeMo.git@95f92737cfb8ee0123bb328b07a2d24c6d859aff'
curl -fL -o parakeet-unified-en-0.6b.nemo \
  https://huggingface.co/nvidia/parakeet-unified-en-0.6b/resolve/main/parakeet-unified-en-0.6b.nemo
uv run --no-sync python convert-coreml.py \
  --output-dir ./build/parakeet_unified_coreml --streaming-context 70,2,2
uv run --no-sync python quantize_int8.py
uv run --no-sync python stage_hf.py
```

This recipe is **source-reviewed, not executed here**; use its scripts' output layout and compare-model/benchmark validation before publishing. The pinned conversion docs report NeMo 2.7.3 PyPI cannot restore the new chunked fields; `--no-sync` prevents undoing the Git overlay. FP16 encoder packages are ~1.1 GB each, INT8 ~565 MB each in the publisher's rounded units, plus decoder ~14 MB and joint ~3.3 MB. `stage_hf.py` compiles packages to `.mlmodelc`. Python/NeMo/torch/coremltools are **development-time only**.[10]

Download/install policy: `~/.cache/jot`, pinned manifest + per-file hashes, temporary staging, safe extraction if using archives, atomic promotion, preserve old valid model on failure. No models in Git or a mutable signed bundle. Compiled Core ML assets may need OS/runtime compatibility validation on the deployment target; this test only establishes this Mac.

**Weight license discrepancy:** current NVIDIA card says **NVIDIA Open Model License**, while converted Core ML/GGUF cards say **CC-BY-4.0**. Sendpoint repeats the latter. Conversion does not itself grant relicensing rights: preserve NVIDIA provenance/terms and resolve redistribution terms before bundling weights. This corrects rather than blindly inherits the sibling's license statement.[5], [10]

## Odin API and build surface

### Selected route: system Objective-C API, no external C ABI shim

Reuse [Sendpoint `coreml-stream.odin`], [P] and its retained [FluidAudio license](https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt/LICENSE-FluidAudio.txt). The Apple runtime is callable through Odin; no outside C library or Swift bridge is necessary:

```odin
import "base:intrinsics"
import ns "core:sys/darwin/Foundation"
@(require) foreign import coreml "system:CoreML.framework"
send :: intrinsics.objc_send
@(objc_class="MLModel")
Model :: struct {using _: ns.Object}
```

Bind exact selector/type signatures for `MLModelConfiguration` (`setComputeUnits:`), `MLModel` (`modelWithContentsOfURL:configuration:error:`, `predictionFromFeatures:error:`), `MLMultiArray` (`initWithShape:dataType:error:`, `dataPointer`, `strides`), `MLDictionaryFeatureProvider` and feature values. Use the compiler-backed `objc_send`, **not an unsafe arbitrary variadic `objc_msgSend` cast**. Foundation bindings supply the Objective-C runtime C boundary. `nil` denotes void return in these calls. No static/dylib vendoring or Homebrew inference runtime is needed.[P], [A]

Compile reproduction, using the sibling's pinned source in a temporary checkout:

```sh
odin build docs/research/odin-stt/coreml-stream.odin -file -o:speed -out:coreml-stream
./coreml-stream /path/to/verified-model-cache \
  /path/to/parakeet_unified_preprocessor.mlmodelc /path/to/16k-mono.f32le
```

Do not ship the prototype unchanged. Replace assertions/process exits with typed errors, own one worker/session, retain/release objects across autorelease pools, inspect data types/shapes/**strides**, reuse buffers, and provide begin/feed/finish/cancel/reset semantics. Core ML predictions here are synchronous; cancellation invalidates the take immediately but must wait for an in-flight call before freeing its objects. Copy whole transcript snapshots to the UI; do not append partial strings as deltas. These production needs are already identified in Sendpoint; reuse its ownership/cancellation checklist rather than inventing a second one.[S], [P]

### Fallback: transcribe.cpp C API / static archive

Pinned and built **`e85b30edac87533168863283c1e595bf39bd7d15`**, version 0.2.4. Download [Q8_0 GGUF](https://huggingface.co/handy-computer/parakeet-unified-en-0.6b-gguf/tree/d5249700b2382bf5c5024c2421d101b8db54a629): ~731 MB (F16 1.24 GB, F32 2.47 GB, Q4_K_M 477 MB). Tested Q8 SHA-256: `4b50b6dd862bf6e346929aaf4f5eaacec003bfa3f56462d6c874b41ef2f38795`. Same file supports offline and buffered streaming. Conversion recipe: `uv run --project scripts/envs/parakeet scripts/convert-parakeet.py nvidia/parakeet-unified-en-0.6b`, then `transcribe-quantize INPUT-F32.gguf OUTPUT-Q8.gguf --quant Q8_0`.[1]

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
  -DTRANSCRIBE_BUILD_TESTS=OFF -DTRANSCRIBE_BUILD_TOOLS=ON \
  -DGGML_METAL_EMBED_LIBRARY=ON
cmake --build build -j 8
libtool -static -o libjot-transcribe.a \
  build/src/libtranscribe.a build/ggml/src/libggml.a \
  build/ggml/src/libggml-cpu.a \
  build/ggml/src/ggml-metal/libggml-metal.a \
  build/ggml/src/libggml-base.a
```

This build succeeded locally with AppleClang 21; CMake was missing, so `uvx --from cmake cmake` supplied the build tool without changing Homebrew. Odin successfully linked the combined archive, called `transcribe_version()` (0.2.4) and returned status 3 for a deliberately nonexistent model. `otool -L` on the CLI showed **only Apple system libraries/frameworks**, no Python/ORT/Homebrew/ggml dylib. Combining archives satisfies one shipped archive, not a prohibition on all bundled third-party code. Keep all licenses. Default native CPU flags target the build host; choose an M1-compatible baseline before distribution.[4]

```odin
foreign import tc {
    "libjot-transcribe.a", // relative to binding; absolute path was rejected here
    "system:c++", "system:Accelerate.framework",
    "system:Foundation.framework", "system:Metal.framework",
    "system:MetalKit.framework",
}
foreign tc {
    transcribe_version :: proc() -> cstring ---
    transcribe_model_load_file :: proc(path: cstring, params: rawptr,
                                       model: ^rawptr) -> i32 ---
}
```

Real binding: typed opaque model/session handles; header-matched structs and initializers. Bind `transcribe_model_load_file`, `session_init`, `stream_begin`, `stream_feed`, `stream_get_text`, `stream_finalize`, `stream_reset`, `session_free`, `model_free` (all with `transcribe_` prefix). Configure `transcribe_parakeet_buffered_stream_ext` and `stream_params.family`; feed f32 mono 16 kHz. C enums/status/int are 32-bit, C bool is one-byte (`b8`); verify struct sizes/offsets and initialize `struct_size` with the library functions. Returned strings are borrowed; copy before the next mutating call. One worker owns session calls. Default static build is preferred; shared mode also creates ggml shared components, not just one dylib.[2], [3], [4]

## Other candidates and why not

| Candidate | Evidence and disposition |
|---|---|
| **sherpa-onnx** | Exact Unified online recognizer, decoder and export scripts; v1.13.8 arm64/universal2 static/shared no-TTS builds. Published int8 offline/240/560/1120 ms archives ~501 MB compressed. ONNX int8 encoder ~624 MiB plus decoder ~6.9/joiner ~1.7 MiB; FP32 external weights ~2.3 GiB. `run-streaming.sh` installs NeMo Git and calls `export_onnx_streaming.py`. Easy C ASR API, but Sendpoint's low-latency preset was slower than real time; packaged `coreml` did not solve it. Keep as reference, not default.[6], [7], [S] |
| **ONNX Runtime C directly** | Use sherpa's exported graphs; bind `OrtGetApiBase()->GetApi(...)` and API-table `CreateEnv`, `CreateSession`, `CreateTensorWithDataAsOrtValue`, `Run`, release functions. Would still implement frontend, RNNT decoding, tokens and buffering in Odin. CoreML EP presence is not proof of ANE acceleration/graph coverage. More integration work than reusing either demonstrated pipeline.[6], [8], [S] |
| **MLX / mlx-c** | mlx-c is an array API, not an ASR C interface. parakeet-mlx exposes Python RNNT/TDT/CTC and local-attention streaming, chiefly illustrated with TDT v3. No confirmed drop-in Unified dynamic-chunked-convolution streaming C implementation found. Python at runtime is disallowed; porting a network to mlx-c needlessly repeats solved work.[9] |
| **transcribe.cpp / ggml** | Best conventional C-library fallback: actual Unified support, Metal, bundled features/decoder, one GGUF, dynamic validated tuples. Reports 188–228× offline M4 Max Metal speed, but that is not streaming throughput. Local 320 ms path lost to direct Core ML; still far faster than real time and simpler host lifecycle.[1] |

Sherpa bindings, if needed, are `SherpaOnnxCreateOnlineRecognizer`, `CreateOnlineStream`, `OnlineStreamAcceptWaveform`, `IsOnlineStreamReady`, `DecodeOnlineStream`, `GetOnlineStreamResult`, `OnlineStreamInputFinished`, and paired destroys (all prefixed `SherpaOnnx`). Match complete versioned structs; do not pass offline export to online recognizer. Reuse the already-executed sibling probe rather than rewriting the ABI.[12], [S]

## Microphone capture: `vendor:miniaudio`

The installed package handles capture, resampling and channel/sample conversion; request **client format** 16 kHz mono f32, not an assumption that the physical mic runs at 16 kHz. This exact configuration and C callback signature compiled, linked, and ran locally, printing `capture config 16000 1 f32`:[15]

```odin
import ma "vendor:miniaudio"
callback :: proc "c" (device: ^ma.device, output, input: rawptr, frames: u32) {
    // Copy frames f32 samples into preallocated single-producer queue.
    // No inference, allocation, UI, I/O, or blocking locks here.
}
cfg := ma.device_config_init(.capture)
cfg.capture.format = .f32
cfg.capture.channels = 1
cfg.sampleRate = 16000
cfg.dataCallback = callback
// ma.device_init(nil, &cfg, &device); ma.device_start(&device)
```

For mono `frames` equals sample count. Copy input before callback returns. An Odin `core:`-based bounded SPSC queue carries PCM to the inference worker; for Core ML accumulate until the next 5120/2560-sample readiness boundary, or let the fallback C stream accept 10–20 ms blocks. Detect overflow explicitly. Stop capture outside the callback, settle callbacks, drain accepted audio, then finalize once. Cancel invalidates the take, discards stale partials, and waits for inference ownership to settle before reuse. Keep live model/session calls off the audio and UI threads.[S], [P], [15]

The mic probe deliberately **did not initialize/start a device or request consent**. Physical-device negotiation, privacy permissions, hotplug, resampling quality and end-to-end release latency remain implementation tests. Add `NSMicrophoneUsageDescription` and test the chosen binary/app permission identity; a working file-fed CLI does not settle packaging/TCC.

## Remaining gates, not reasons to redo this research

1. Adapt the existing direct-CoreML pipeline into typed Odin worker-owned state, not a new DSP/network port; test empty/short/exact-boundary input and repeated begin/finish/cancel.
2. Pin model installation against verified published bytes; settle weight redistribution license mismatch and minimum macOS support.
3. Integrate miniaudio and jot's immediate-mode transcript display; test permissions, queue overflow, route changes and no live mic after teardown.
4. Measure first-audio→partial and release→visible-final median/p95 over live takes, whole-system memory/idle power, sustained thermal performance and corpus parity. No ANE-placement or total-RAM claim yet.
5. Retain the tested transcribe.cpp path as a fallback, not a second simultaneously loaded engine. No full app or distribution validation was done in this ticket.

## Sources

[S]: https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt.md
[P]: https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt/coreml-stream.odin
[M]: https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt/results/coreml-cache-manifest.json
[A]: https://developer.apple.com/documentation/coreml/mlmodel
[1]: https://github.com/handy-computer/transcribe.cpp/blob/e85b30edac87533168863283c1e595bf39bd7d15/docs/models/parakeet-unified-en-0.6b.md
[2]: https://github.com/handy-computer/transcribe.cpp/blob/e85b30edac87533168863283c1e595bf39bd7d15/include/transcribe.h
[3]: https://github.com/handy-computer/transcribe.cpp/blob/e85b30edac87533168863283c1e595bf39bd7d15/include/transcribe/parakeet.h
[4]: https://github.com/handy-computer/transcribe.cpp/blob/e85b30edac87533168863283c1e595bf39bd7d15/CMakeLists.txt
[5]: https://huggingface.co/nvidia/parakeet-unified-en-0.6b/blob/main/README.md
[6]: https://github.com/k2-fsa/sherpa-onnx/tree/040afe360a38e25daaa325ce8889abf93ea02609/scripts/nemo/parakeet-unified-en-0.6b
[7]: https://github.com/k2-fsa/sherpa-onnx/releases/tag/v1.13.8
[8]: https://onnxruntime.ai/docs/get-started/with-c.html
[9]: https://github.com/ml-explore/mlx-c
[10]: https://github.com/FluidInference/mobius/blob/864ef8050f2f281d0761de26e3a03108f9f1ce73/models/stt/parakeet-unified-en-0.6b/coreml/README.md
[11]: https://github.com/FluidInference/FluidAudio/blob/8145085136df11758cc1303ab54d8e032c12bd41/Sources/FluidAudio/ASR/Parakeet/Unified/benchmark.md
[12]: https://github.com/k2-fsa/sherpa-onnx/blob/040afe360a38e25daaa325ce8889abf93ea02609/sherpa-onnx/c-api/c-api.h
[15]: https://github.com/odin-lang/Odin/tree/dev-2026-09/vendor/miniaudio

Additional source inspections: [actual ggml buffered-stream implementation](https://github.com/handy-computer/transcribe.cpp/blob/e85b30edac87533168863283c1e595bf39bd7d15/src/arch/parakeet/model.cpp), [parakeet-mlx Python API](https://github.com/senstella/parakeet-mlx/blob/master/README.md), [Sendpoint reproduction script](https://github.com/saiashirwad/sendpoint/blob/4b753c2c96d3b45ec2eeebcc129dcdd150161e16/docs/research/odin-stt/run.sh). Local measurements above are new probes from this session, not numbers taken from those sources.
