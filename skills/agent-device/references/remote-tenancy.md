# Remote Tenancy and Lease Admission

Use this reference for remote daemon HTTP flows that require explicit tenant/run admission control.

## Transport prerequisites

- Start daemon in HTTP mode (`AGENT_DEVICE_DAEMON_SERVER_MODE=http|dual`).
- Point remote clients at the host with `AGENT_DEVICE_DAEMON_BASE_URL=http(s)://host:port[/base-path]`.
- Use `AGENT_DEVICE_DAEMON_AUTH_TOKEN` / `--daemon-auth-token` for authentication.
- Prefer an auth hook (`AGENT_DEVICE_HTTP_AUTH_HOOK`) for caller validation and tenant injection.

## Lease lifecycle (JSON-RPC)

Use `POST /rpc` with JSON-RPC 2.0 methods:

- `agent_device.lease.allocate`
- `agent_device.lease.heartbeat`
- `agent_device.lease.release`

Example allocate:

```bash
curl -sS "${AGENT_DEVICE_DAEMON_BASE_URL}/rpc" \
  -H "content-type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"jsonrpc":"2.0","id":"alloc-1","method":"agent_device.lease.allocate","params":{"tenantId":"acme","runId":"run-123","ttlMs":60000}}'
```

## Command admission contract

For tenant-isolated command execution, pass all four flags:

```bash
agent-device \
  --tenant acme \
  --session-isolation tenant \
  --run-id run-123 \
  --lease-id <lease-id> \
  session list --json
```

## Failure semantics

- Missing tenant/run/lease fields: `INVALID_ARGS`
- Lease not active or wrong scope: `UNAUTHORIZED`
- Method mismatch: JSON-RPC `-32601` (HTTP 404)

## Operational guidance

- Keep TTL short and heartbeat only while a run is active.
- Release lease immediately on run completion/error paths.
- For remote debug, inspect logs on the remote host.
