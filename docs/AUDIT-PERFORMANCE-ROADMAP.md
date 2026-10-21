# TaoClip Desktop — Audit & Performance Roadmap

Status: initial static audit of the public fork. Runtime build/tests have not yet been executed in this environment.

## Baseline

- Source: `hosyminh/taoclip-desktop`, forked to `JamesPham79/taoclip-desktop`.
- Baseline branch: `main`; working branch: `james/performance-audit`.
- App: Electron + React + Vite; video pipeline uses FFmpeg/FFprobe.
- Package version: `0.4.1`.
- Existing scripts: `npm run build`, `npm test`, `npm run test:ai`, `npm run test:dl`, `npm run test:face`, `npm run dist`.
- Lockfile exists; dependencies should be installed with `npm ci` for reproducible validation.
- MIT license is present. Runtime-downloaded models/tools and external services have separate terms; do not assume they are covered by the repository license.

## Initial findings from static inspection

1. The project already has a bounded cache queue (concurrency 2), deduplicated in-flight thumbnail/waveform work, cache fingerprints based on path/size/mtime and parameters, and priority promotion for selected media.
2. Preview proxies are transcoded to H.264 at up to 1280×720 with the ultrafast preset and CRF 30. This is a sensible compatibility/performance tradeoff, but the cost of generating proxies for long/high-resolution media should be measured.
3. Export has CPU and hardware encoder paths (NVENC, Quick Sync, AMF). Hardware availability and output correctness need validation on actual Windows hardware; a successful small encoder probe does not guarantee every driver/device combination is stable.
4. The current npm test suite includes project-model/renderer logic and FFmpeg integration tests. AI, downloader, and face-swap tests are separate and may require network access or optional model/tool assets.
5. The Windows installer configuration uses electron-builder/NSIS. CI should validate both the general build/tests and Windows packaging; successful Vite build alone does not prove the installer is healthy.
6. No existing GitHub Actions CI workflow was found at `.github/workflows/ci.yml` during this audit.

## Validation sequence

1. Install dependencies reproducibly: `npm ci`.
2. Run `npm run build`.
3. Run `npm test` (unit/project tests plus FFmpeg integration tests).
4. Run optional suites separately when prerequisites are available: `npm run test:ai`, `npm run test:dl`, `npm run test:face`.
5. Build the Windows installer with `npm run dist` on a Windows runner.
6. Smoke-test the packaged app on Windows: launch, import video/audio/image/GIF, seek and scrub timeline, save/reopen project, export MP4, cancel export, relink media, and verify AI/model download errors are recoverable.

## Performance roadmap (priority order)

### P0 — Establish a measurable baseline
- Capture build/test status and installer artifact size.
- Benchmark a representative project: 1080p and 4K sources, 10/50/100 timeline clips, audio waveforms, thumbnails, proxy generation, and export.
- Record CPU, RAM, GPU/VRAM, timeline interaction latency, proxy generation time, and export FPS. Do not claim improvement without before/after measurements.

### P1 — Timeline responsiveness
- Profile React renders while scrubbing and selecting clips; memoize only components proven to re-render unnecessarily.
- Ensure expensive waveform/thumbnail/proxy work stays off the UI thread and remains bounded by the existing queue.
- Confirm stale asynchronous results are discarded after relink/project switch and that selected media receives cache priority.
- Consider virtualizing timeline clip rendering only if profiling shows DOM/render cost is a bottleneck.

### P1 — Preview and media cache
- Measure proxy creation time and cache hit rate on large media.
- Keep cache keys invalidated by file changes; add tests for source replacement, deletion, and concurrent duplicate requests.
- Consider user-selectable proxy resolution/quality only after benchmarks establish the tradeoff.
- Avoid unbounded memory growth from image data URLs, waveforms, and thumbnails; inspect cleanup on project close/reload.

### P1 — Export reliability and speed
- Benchmark CPU versus each supported hardware encoder on supported Windows devices.
- Validate cancellation, temporary-file cleanup, audio sync, frame rate, aspect ratios, and output replacement on failure.
- Keep a reliable CPU fallback and surface actionable errors when hardware encoding is unavailable.

### P2 — Packaging, security, and maintainability
- Validate Windows x64 packaging and installer launch in CI.
- Review Electron IPC input validation, external URL handling, download sources, and dependency audit findings before release.
- Document external model/tool licenses and any cloud data transfer clearly.
- Pin and update dependencies deliberately; review security fixes without blindly applying breaking upgrades.

## Acceptance criteria for an optimization

- Baseline and optimized runs use the same sample media, project, device, and settings.
- No regression in output correctness, audio/video sync, project compatibility, or cancellation.
- Unit/integration tests pass.
- Quantified improvement is recorded (e.g. lower scrub latency, shorter proxy generation, lower memory use, or higher export FPS).
