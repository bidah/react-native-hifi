# Session Management

## Named sessions

```bash
agent-device --session auth open Settings --platform ios
agent-device --session auth snapshot -i
```

Sessions isolate device context. A device can only be held by one session at a time.

## Best practices

- Name sessions semantically.
- Close sessions when done.
- Use separate sessions for parallel work.
- For orchestrated QA runs, prefer a pre-bound session/platform over repeating per-command selectors.
- For remote tenant-scoped automation, run commands with:
  `--tenant <id> --session-isolation tenant --run-id <id> --lease-id <id>`
- In iOS sessions, use `open <app>`. `open <url>` opens deep links.
- For dev loops with persistent runtime state, use `open <app> --relaunch` to restart the app process.
- Use `--save-script [path]` to record replay scripts on `close`.
- Use `close --shutdown` on iOS simulators or Android emulators to shut down the target as part of session teardown.

## Session-bound automation

```bash
export AGENT_DEVICE_SESSION=qa-ios
export AGENT_DEVICE_PLATFORM=ios
export AGENT_DEVICE_SESSION_LOCK=strip

agent-device open MyApp --relaunch
agent-device snapshot -i
agent-device press @e3
agent-device close
```

- `AGENT_DEVICE_SESSION` and `AGENT_DEVICE_PLATFORM` provide the default binding.
- A configured `AGENT_DEVICE_SESSION` enables lock policy enforcement by convention.
- `--session-lock reject|strip` controls whether conflicting selectors fail or are ignored.
- Lock policy applies to nested `batch` steps too.

## Scoped device isolation

```bash
agent-device devices --platform ios --ios-simulator-device-set /tmp/tenant-a/simulators
agent-device devices --platform android --android-device-allowlist emulator-5554,device-1234
```

- Scope is applied before selectors.
- Out-of-scope selectors fail with `DEVICE_NOT_FOUND`.

## Listing sessions

```bash
agent-device session list
```

## Replay within sessions

```bash
agent-device replay ./session.ad --session auth
agent-device replay -u ./session.ad --session auth
```
