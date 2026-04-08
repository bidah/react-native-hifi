# Permissions and Setup

## iOS snapshots

iOS snapshots use XCTest and do not require macOS Accessibility permissions.

## iOS physical device runner

For iOS physical devices, XCTest runner setup requires valid signing/provisioning.
Use Automatic Signing in Xcode, or provide optional overrides:

- `AGENT_DEVICE_IOS_TEAM_ID`
- `AGENT_DEVICE_IOS_SIGNING_IDENTITY`
- `AGENT_DEVICE_IOS_PROVISIONING_PROFILE`
- `AGENT_DEVICE_IOS_BUNDLE_ID` (optional runner bundle-id base override)

Security guidance:
- These variables are optional and only needed for physical-device XCTest setup.
- Treat values as sensitive host configuration.
- Prefer Xcode Automatic Signing when possible.

If setup/build takes long, increase:
- `AGENT_DEVICE_DAEMON_TIMEOUT_MS` (default `90000`)

If daemon startup fails with stale metadata, clean:
- `~/.agent-device/daemon.json`
- `~/.agent-device/daemon.lock`

## iOS permission alerts (simulator only)

```bash
agent-device alert wait          # block until an alert appears (default 10s timeout)
agent-device alert accept        # accept the frontmost alert
agent-device alert dismiss       # dismiss the frontmost alert
agent-device alert get           # read alert title/message without acting
```

`alert accept` and `alert dismiss` include a built-in 2s retry window.

**Preferred pattern for clean simulator sessions:**

```bash
agent-device open MyApp --platform ios
agent-device alert wait 5000     # wait up to 5s for the permission prompt
agent-device alert accept        # accept; retries internally if not yet actionable
```

`alert` is only supported on iOS simulators.

## iOS: "Allow Paste" dialog

iOS 16+ shows an "Allow Paste" prompt. Under XCUITest this prompt is suppressed by the testing runtime. Use `xcrun simctl pbcopy booted` to set clipboard content directly.

## Simulator troubleshooting

- If snapshots return 0 nodes, restart Simulator and re-open the app.
