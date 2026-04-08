# Performance Metrics (`perf` / `metrics`)

Use this reference when you need to measure launch performance in agent workflows.

## Quick flow

```bash
agent-device open Settings --platform ios
agent-device perf --json
```

Alias: `agent-device metrics --json`

## What is measured today

- Session-scoped `startup` timing only.
- Sampling method: `open-command-roundtrip`.
- Unit: milliseconds (`ms`).
- Source: elapsed wall-clock time around each session `open` command dispatch.

## Output fields to use

- `metrics.startup.lastDurationMs`: most recent startup sample.
- `metrics.startup.lastMeasuredAt`: ISO timestamp.
- `metrics.startup.sampleCount`: number of retained samples.
- `metrics.startup.samples[]`: recent startup history.
- `sampling.startup.method`: sampling method identifier.

## Platform support

- iOS simulator: supported for startup sampling.
- iOS physical device: supported for startup sampling.
- Android emulator/device: supported for startup sampling.
- `fps`, `memory`, and `cpu`: currently placeholders (`available: false`).

## Interpretation guidance

- Treat startup values as command round-trip timing, not true app first-frame telemetry.
- Compare like-for-like runs (same device target, same app build, same workflow/session steps).
- Use multiple runs and compare trend/median, not one-off samples.

## Common pitfalls

- Running `perf` before any `open` yields no startup sample.
- Comparing values across different devices/runtimes introduces large noise.
- Interpreting current `startup` as CPU/FPS/memory would be incorrect.
