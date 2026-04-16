# WasmVideoPlayer — Agent Guide

> **Context**: This file is for AI coding agents working on the WasmVideoPlayer project. The project README (`README.md`) is written in Chinese; source code comments and identifiers are in English.

---

## Project Overview

WasmVideoPlayer is a browser-based H.265/HEVC video player proof-of-concept built around **FFmpeg 3.3 compiled to WebAssembly** via Emscripten. It demonstrates soft-decoding of H.265 (and H.264/AAC) in the browser, rendering decoded YUV frames with **WebGL** and playing PCM audio through the **Web Audio API**. The project does not rely on any modern frontend framework; it is plain HTML5 + vanilla JavaScript.

Key capabilities:
- Playback of MP4/FLV files and HTTP-FLV streams.
- Chunked download with HTTP `Range` requests or an optional WebSocket transport.
- Audio/video synchronization using the audio clock.
- Seeking support (pause, seek, resume with buffer refill).

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Decoder (native) | C + FFmpeg 3.3 (`libavformat`, `libavcodec`, `libavutil`, `libswscale`) |
| WASM toolchain | Emscripten (`emcc`, `emconfigure`, etc.) |
| Glue / worker | Vanilla JavaScript Web Worker (`decoder.js`) |
| Video rendering | WebGL (YUV → RGB shader) (`webgl.js`) |
| Audio playback | Web Audio API (`pcm-player.js`) |
| Network | `XMLHttpRequest` / `fetch` / WebSocket (`downloader.js`) |
| UI | Plain HTML5 + CSS (`index.html`, `styles/style.css`) |
| Optional server | Node.js + `ws` module for WebSocket video serving (`ws/ws_server.js`) |

There is **no `package.json` at the project root**. The main application is pure static files. Only the optional WebSocket server under `ws/` has a `package-lock.json` and depends on the `ws` npm package.

---

## Project Structure

```
.
├── index.html              # Demo page / UI entry point
├── common.js               # Message constants + Logger utility
├── player.js               # Main thread: Player logic, A/V sync, buffering, UI control
├── decoder.js              # Web Worker: loads WASM and drives the C decoder
├── downloader.js           # Web Worker: handles HTTP/WS downloads
├── pcm-player.js           # Web Audio API PCM player
├── webgl.js                # WebGL YUV420P renderer
├── decoder.c               # C source: FFmpeg demux/decode wrapper
├── libffmpeg.js            # Emscripten-generated JS glue (loads libffmpeg.wasm)
├── libffmpeg.wasm          # Compiled FFmpeg + decoder.c WASM binary
├── build_decoder.sh        # Build script: configure & compile FFmpeg static libs
├── build_decoder_wasm.sh   # Build script: compile decoder.c into WASM
├── dist/                   # Output of FFmpeg build (headers + static libs)
│   ├── include/
│   └── lib/
├── video/                  # Sample H.265 video file (`h265_high.mp4`)
├── styles/                 # CSS
├── img/                    # UI icons
└── ws/                     # Optional Node.js WebSocket server
    ├── ws_server.js
    └── package-lock.json
```

---

## Architecture

The player uses a **3-thread model implemented with Web Workers** because pthread support in WASM was experimental at the time of writing.

### Threads / Roles

1. **Main Thread (`Player` in `player.js`)**
   - UI event handling (play, pause, stop, fullscreen, seek).
   - Download pacing and buffer management.
   - Audio/video synchronization.
   - Schedules video rendering via `requestAnimationFrame`.
   - Forwards downloaded data to the Decoder Worker.

2. **Decoder Worker (`decoder.js`)**
   - Loads `libffmpeg.js` (Emscripten glue) which in turn loads `libffmpeg.wasm`.
   - Exposes C functions via `Module._xxx` and uses `Module.addFunction` to register JS callbacks for decoded frames.
   - Processes requests from the main thread: init, open, feed data, start/pause decode, seek.

3. **Downloader Worker (`downloader.js`)**
   - Fetches file metadata (`Content-Length` via HEAD-like XHR).
   - Downloads file chunks via HTTP `Range` or via a custom WebSocket protocol.
   - Reports chunks back to the main thread using `postMessage` with `Transferable` ArrayBuffers.

### Communication

- All inter-thread communication uses `postMessage`.
- Large buffers (video frames, download chunks) are transferred with **Transferable Objects** to avoid memory copies.
- Message types are integer constants defined in `common.js` (e.g., `kInitDecoderReq`, `kVideoFrame`, `kFileData`).

### Data Flow

```
[Network] -> Downloader Worker -> Main Thread -> Decoder Worker -> WASM/FFmpeg
                                                           |
                                                           v
                                            [YUV frames] -> WebGL
                                            [PCM frames] -> Web Audio
```

### Buffering Strategy

Because FFmpeg’s synchronous read callbacks must always return data, the player maintains two critical buffers:
- **File data buffer**: For non-streaming playback, data is written to an in-memory MEMFS file (`FORCE_FILESYSTEM=1`). For streams, an `AVFifoBuffer` is used in C.
- **Decoded frame buffer**: The main thread buffers decoded frames. When the buffered time reaches `maxBufferTimeLength` (1 second), decoding is paused. When it drops below 0.5 s, decoding resumes.

### A/V Synchronization

- Audio is the master clock.
- `PCMPlayer.getTimestamp()` returns `audioCtx.currentTime`.
- Video frames are rendered immediately if they are behind audio; otherwise they are deferred via `setTimeout` scheduling inside `displayLoop`.

---

## Build Process

### Prerequisites

1. **Emscripten** installed and activated in the shell (tested with 1.39.5 at the time of creation).
2. **FFmpeg 3.3 source** cloned into a sibling directory named `../ffmpeg`:
   ```bash
   git clone https://git.ffmpeg.org/ffmpeg.git
   cd ffmpeg
   git checkout release/3.3
   ```

### Build Commands

Run the full build from the project root:

```bash
./build_decoder.sh
```

What happens:
1. `build_decoder.sh` runs `emconfigure` + `make` + `make install` for FFmpeg, outputting headers and static libraries to `dist/`.
   - Configuration is heavily stripped: only `hevc`, `h264`, `aac` decoders and `mov`/`flv` demuxers are enabled.
2. `build_decoder_wasm.sh` compiles `decoder.c` together with the FFmpeg static libraries into `libffmpeg.js` and `libffmpeg.wasm`.

**Exported C functions** (see `build_decoder_wasm.sh`):
- `_initDecoder`, `_uninitDecoder`, `_openDecoder`, `_closeDecoder`, `_sendData`, `_decodeOnePacket`, `_seekTo`, `_main`, `_malloc`, `_free`

**Important Emscripten flags**:
- `-s WASM=1`
- `-s TOTAL_MEMORY=67108864` (64 MB)
- `-s FORCE_FILESYSTEM=1` (enables MEMFS for file-backed buffering)
- `-s EXTRA_EXPORTED_RUNTIME_METHODS="['addFunction']"`
- `-s RESERVED_FUNCTION_POINTERS=14`

---

## Running / Testing

No automated test suite exists. Testing is manual via a browser:

```bash
# Using any static HTTP server, e.g.:
npx http-server -p 8080 .
# Or Python:
python3 -m http.server 8080
```

Then open `http://127.0.0.1:8080` in a browser.

### Optional WebSocket Server

If you want to test WebSocket-based delivery:

```bash
cd ws
npm install   # installs the `ws` package
node ws_server.js
```

This serves the sample video from `../video/h265_high.mp4` on `ws://0.0.0.0:8080`.

---

## Code Organization & Conventions

### File Responsibilities

| File | Responsibility |
|------|----------------|
| `common.js` | Shared message-type constants and a simple `Logger` helper. |
| `player.js` | ~1160 lines. Core orchestrator. Avoid making this larger; prefer extracting helpers. |
| `decoder.js` | ~260 lines. Thin JS wrapper around the WASM module. |
| `downloader.js` | ~220 lines. Abstracts `XMLHttpRequest` and WebSocket fetching. |
| `pcm-player.js` | ~195 lines. Self-contained PCM → Web Audio renderer. |
| `webgl.js` | ~148 lines. Self-contained WebGL YUV renderer. |
| `decoder.c` | ~990 lines. Single C file with FFmpeg decoder logic. |

### Style Guidelines

- **Indentation**: 4 spaces in C; mixed 2/4 spaces in JS (existing files use a variety). When editing, match the indentation of the surrounding code.
- **Naming**:
  - C: `camelCase` for functions; `kErrorCode_Xxx` for enum constants.
  - JS: `camelCase` for methods/variables; `PascalCase` for constructors (`Player`, `Decoder`, `PCMPlayer`).
- **Logging**: Use the provided `Logger` class in JS (`new Logger("ModuleName")`) and `simpleLog()` in C. The C logger respects a `logLevel` setting (`kLogLevel_None`, `kLogLevel_Core`, `kLogLevel_All`).
- **Comments**: Keep English comments in source files to match the existing style.

### State Constants

- `decoderStateIdle / decoderStateInitializing / decoderStateReady / decoderStateFinished`
- `playerStateIdle / playerStatePlaying / playerStatePausing`

When adding new states, add them here and update the switch blocks in `player.js`.

---

## Important Implementation Details

### WASM / C Boundary

- Data is copied into WASM heap with `Module.HEAPU8.set(typedArray, cacheBuffer)` before calling C functions.
- Decoded frame callbacks copy data out of the WASM heap with `Module.HEAPU8.subarray(...)` and then transfer ownership via `postMessage(objData, [objData.d.buffer])`.

### Video Format Constraints

- The WebGL shader only supports **YUV420P**. If FFmpeg outputs another pixel format, `processDecodedVideoFrame` in `decoder.c` logs an error and returns `kErrorCode_Invalid_Format`.

### Seeking

- Seeking is implemented in C via `avformat_seek_file`.
- On the JS side, seeking pauses playback, clears the frame buffer, resets the download offset, and refills the buffer before resuming.
- The `accurateSeek` flag discards frames with timestamps earlier than the seek target.

### Memory

- The WASM module is compiled with a fixed 64 MB memory pool (`TOTAL_MEMORY`).
- Large video frames + the in-memory file buffer can exhaust this quickly for high-resolution content. If you change resolutions or buffering sizes, you may need to increase `TOTAL_MEMORY` in `build_decoder_wasm.sh`.

---

## Security Considerations

- The player fetches arbitrary URLs provided by the user in `index.html`. No URL whitelist or CSP is present.
- The WebSocket server (`ws/ws_server.js`) reads local files based on a hardcoded path (`../video/h265_high.mp4`). It does not validate input paths; it is intended for local demo use only.
- No input sanitization is performed on the video bitstream before passing it to FFmpeg. Malicious media files could theoretically trigger decoder vulnerabilities.

---

## Common Pitfalls for Agents

1. **Do not add a `package.json` or bundler to the root unless explicitly requested.** The project is intentionally dependency-free in the browser.
2. **Do not rename `libffmpeg.js` or `libffmpeg.wasm` without updating both `decoder.js` and the HTML.** The WASM file name is embedded inside the Emscripten glue.
3. **Changing FFmpeg configure flags** in `build_decoder.sh` changes which codecs are available. If you add a new codec, ensure the corresponding decoder flag is passed to `--enable-decoder=...`.
4. **Seek and buffering logic is tightly coupled.** Changes to `maxBufferTimeLength` or `chunkInterval` in `player.js` can break A/V sync or cause FFmpeg read errors.
5. **The project does not use ES modules.** All JS files are loaded with `<script>` tags or `importScripts()` inside workers. Use the same pattern for new files.
