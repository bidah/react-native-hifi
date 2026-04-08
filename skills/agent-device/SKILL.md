---
name: agent-device
description: Automates interactions for iOS simulators/devices and Android emulators/devices. Use when navigating apps, taking snapshots/screenshots, tapping, typing, scrolling, or extracting UI info on mobile targets.
---

## Mobile Automation with agent-device

For exploration, use snapshot refs. For deterministic replay, use selectors.
For structured exploratory QA bug hunts and reporting, use [../dogfood/SKILL.md](../dogfood/SKILL.md).

## Start Here (Read This First)

Use this skill as a router, not a full manual.

1. Pick one mode:
   - Normal interaction flow
   - Debug/crash flow
   - Replay maintenance flow
2. Run one canonical flow below.
3. Open references only if blocked.

## Decision Map

- No target context yet: `devices` -> pick target -> `open`.
- Normal UI task: `open` -> `snapshot -i` -> `press/fill` -> `diff snapshot -i` -> `close`
- Debug/crash: `open <app>` -> `logs clear --restart` -> reproduce -> `network dump` -> `logs path` -> targeted `grep`
- Replay drift: `replay -u <path>` -> verify updated selectors
- Remote multi-tenant run: allocate lease -> point client at remote daemon base URL -> run commands with tenant isolation flags -> heartbeat/release lease
- Device-scope isolation run: set iOS simulator set / Android allowlist -> run selectors within scope only

## Target Selection Rules

- iOS local QA: use simulators unless the task explicitly requires a physical device.
- iOS local QA in mixed simulator/device environments: run `ensure-simulator` first and pass `--device`, `--udid`, or `--ios-simulator-device-set` on later commands.
- Android local QA: use `install` or `reinstall` for `.apk`/`.aab` files, then relaunch by installed package name.
- Android React Native + Metro flows: prefer `open <package> --remote-config <path> --relaunch`.
- In mixed-device environments, always pin the exact target with `--serial`, `--device`, `--udid`, or an isolation scope.
- For session-bound automation runs, prefer a pre-bound session/platform instead of repeating selectors on every command: set `AGENT_DEVICE_SESSION`, set `AGENT_DEVICE_PLATFORM`, and the daemon will enforce the shared lock policy across CLI, typed client, and RPC entry points.
- Use `--session-lock reject|strip` (or `AGENT_DEVICE_SESSION_LOCK`) only when you need to override the default reject behavior. Lock mode applies to nested `batch` steps too.

## Canonical Flows

### 1) Normal Interaction Flow

```bash
agent-device open Settings --platform ios
agent-device snapshot -i
agent-device press @e3
agent-device diff snapshot -i
agent-device fill @e5 "test"
agent-device close
```

### 1a) Local iOS Simulator QA Flow

```bash
agent-device ensure-simulator --platform ios --device "iPhone 16" --boot
agent-device open MyApp --platform ios --device "iPhone 16" --session qa-ios --relaunch
agent-device snapshot -i
agent-device press @e3
agent-device close
```

Use this when a physical iPhone is also connected and you want deterministic simulator-only automation.

### 1b) Android React Native + Metro QA Flow

```bash
agent-device reinstall MyApp /path/to/app-debug.apk --platform android --serial emulator-5554
agent-device open com.example.myapp --remote-config ./agent-device.remote.json --relaunch
agent-device snapshot -i
agent-device close
```

Do not use `open <apk|aab> --relaunch` on Android. Install/reinstall binaries first, then relaunch by package.

### 1c) Session-Bound Automation Flow

```bash
export AGENT_DEVICE_SESSION=qa-ios
export AGENT_DEVICE_PLATFORM=ios
export AGENT_DEVICE_SESSION_LOCK=strip

agent-device open MyApp --relaunch
agent-device snapshot -i
agent-device batch --steps-file /tmp/qa-steps.json --json
agent-device close
```

Use this for orchestrators that must preserve one bound session/device across many plain CLI calls without a wrapper script.

### 1d) Android Emulator Session-Bound Flow

```bash
export AGENT_DEVICE_SESSION=qa-android
export AGENT_DEVICE_PLATFORM=android

agent-device reinstall MyApp /path/to/app-debug.apk --serial emulator-5554
agent-device --session-lock reject open com.example.myapp --relaunch
agent-device snapshot -i
agent-device close --shutdown
```

### 2) Debug/Crash Flow

```bash
agent-device open MyApp --platform ios
agent-device logs clear --restart
agent-device network dump 25
agent-device logs path
```

Logging is off by default. Enable only for debugging windows.
`logs clear --restart` requires an active app session (`open <app>` first).

### 3) Replay Maintenance Flow

```bash
agent-device replay -u ./session.ad
```

### 4) Remote Tenant Lease Flow (HTTP JSON-RPC)

```bash
export AGENT_DEVICE_DAEMON_BASE_URL=http://mac-host.example:4310
export AGENT_DEVICE_DAEMON_AUTH_TOKEN=<token>

# Allocate lease
curl -sS "${AGENT_DEVICE_DAEMON_BASE_URL}/rpc" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"jsonrpc":"2.0","id":"alloc-1","method":"agent_device.lease.allocate","params":{"runId":"run-123","tenantId":"acme","ttlMs":60000}}'

# Use lease in tenant-isolated command execution
agent-device \
  --tenant acme \
  --session-isolation tenant \
  --run-id run-123 \
  --lease-id <lease-id> \
  session list --json

# Heartbeat and release
curl -sS "${AGENT_DEVICE_DAEMON_BASE_URL}/rpc" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"jsonrpc":"2.0","id":"hb-1","method":"agent_device.lease.heartbeat","params":{"leaseId":"<lease-id>","ttlMs":60000}}'
curl -sS "${AGENT_DEVICE_DAEMON_BASE_URL}/rpc" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"jsonrpc":"2.0","id":"rel-1","method":"agent_device.lease.release","params":{"leaseId":"<lease-id>"}}'
```

## Command Skeleton (Minimal)

### Session and navigation

```bash
agent-device devices
agent-device devices --platform ios --ios-simulator-device-set /tmp/tenant-a/simulators
agent-device devices --platform android --android-device-allowlist emulator-5554,device-1234
agent-device ensure-simulator --device "iPhone 16" --ios-simulator-device-set /tmp/tenant-a/simulators
agent-device ensure-simulator --device "iPhone 16" --runtime com.apple.CoreSimulator.SimRuntime.iOS-18-4 --ios-simulator-device-set /tmp/tenant-a/simulators --boot
agent-device open [app|url] [url]
agent-device open [app] --relaunch
agent-device close [app]
agent-device install <app> <path-to-binary>
agent-device install-from-source <url> [--header "name:value"]
agent-device reinstall <app> <path-to-binary>
agent-device session list
```

Use `boot` only as fallback when `open` cannot find/connect to a ready target.

### Snapshot and targeting

```bash
agent-device snapshot -i
agent-device diff snapshot -i
agent-device find "Sign In" click
agent-device press @e1
agent-device fill @e2 "text"
agent-device is visible 'id="anchor"'
```

`press` is canonical tap command; `click` is an alias.

### Utilities

```bash
agent-device appstate
agent-device clipboard read
agent-device clipboard write "token"
agent-device keyboard status
agent-device keyboard dismiss
agent-device perf --json
agent-device network dump [limit] [summary|headers|body|all]
agent-device push <bundle|package> <payload.json|inline-json>
agent-device trigger-app-event screenshot_taken '{"source":"qa"}'
agent-device get text @e1
agent-device screenshot out.png
agent-device settings permission grant notifications
agent-device settings permission reset camera
agent-device trace start
agent-device trace stop ./trace.log
```

### Batch (when sequence is already known)

```bash
agent-device batch --steps-file /tmp/batch-steps.json --json
```

### Performance Check

- Use `agent-device perf --json` (or `metrics --json`) after `open`.
- For detailed metric semantics, see [references/perf-metrics.md](references/perf-metrics.md).

## Guardrails (High Value Only)

- Re-snapshot after UI mutations (navigation/modal/list changes).
- Prefer `snapshot -i`; scope/depth only when needed.
- Use refs for discovery, selectors for replay/assertions.
- `find "<query>" click --json` returns `{ ref, locator, query, x, y }` — all derived from the matched snapshot node.
- Use `fill` for clear-then-type semantics; use `type` for focused append typing.
- Use `install` for in-place app upgrades, and `reinstall` for deterministic fresh-state runs.
- App binary format support: Android `.apk`/`.aab`, iOS `.app`/`.ipa`.
- Android `.aab` requires `bundletool` in `PATH`.
- `network dump` is best-effort and parses HTTP(s) entries from the session app log file.
- For AndroidTV/tvOS selection, always pair `--target` with `--platform`.
- Permission settings are app-scoped and require an active session app:
  `settings permission <grant|deny|reset> <camera|microphone|photos|contacts|notifications> [full|limited]`
- iOS simulator permission alerts: use `alert wait` then `alert accept/dismiss` — retries internally for up to 2s.

## Common Failure Patterns

- `Failed to access Android app sandbox for /path/app-debug.apk`: Android relaunch flow received an APK path instead of package name. Use `reinstall` first, then `open <package> --relaunch`.
- `Failed to terminate iOS app`: flow may have selected a physical iPhone. Re-run with `ensure-simulator`, then pin with `--device` or `--udid`.

## Common Mistakes

- Mixing debug flow into normal runs (keep logs off unless debugging).
- Continuing to use stale refs after screen transitions.
- Using URL opens with Android `--activity` (unsupported combination).
- Treating `boot` as default first step instead of fallback.

## Security and Trust Notes

- Prefer a preinstalled `agent-device` binary over on-demand package execution.
- If install is required, pin an exact version.
- Keep logging off unless debugging and use least-privilege environments for autonomous runs.

## References

- [references/snapshot-refs.md](references/snapshot-refs.md)
- [references/logs-and-debug.md](references/logs-and-debug.md)
- [references/session-management.md](references/session-management.md)
- [references/permissions.md](references/permissions.md)
- [references/video-recording.md](references/video-recording.md)
- [references/coordinate-system.md](references/coordinate-system.md)
- [references/batching.md](references/batching.md)
- [references/perf-metrics.md](references/perf-metrics.md)
- [references/remote-tenancy.md](references/remote-tenancy.md)
