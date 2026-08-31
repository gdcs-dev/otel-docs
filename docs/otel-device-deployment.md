# Device OTLP Service Deployment

The service definitions in `deploy/device` support systemd and LSB SysV init. Both are disabled until `/etc/rdk-otel/enabled` exists. Installing or enabling a service does not activate telemetry by itself.

## Provisioned files

Install these files as root:

| Source | Target | Owner and mode |
|---|---|---|
| `deploy/device/config/otel-relay.yml` | `/etc/rdk-otel/otel-relay.yml` | `root:root 0644` |
| `deploy/device/config/agent.yaml` | `/etc/rdk-otel/agent.yaml` | `root:root 0644` |
| provisioned bearer token | `/etc/rdk-otel/otel-token` | `root:root 0600` |
| provisioned `DEVICE_ID=...` | `/etc/rdk-otel/device.env` | `root:root 0600` |

The token is local-hop authentication only. Do not place it in YAML, command-line arguments, OTLP resources, or logs. Both systemd and SysV copy it into `/run/rdk-otel` with mode `0600` before each start and remove the runtime copy after stop.

otel-relay binds only `127.0.0.1:4318`, registers service `otel-relay` on Parodus client port `9902`, and emits source `mac:<device-id>/otel-relay`. Its spool is `/var/lib/otel-relay/spool`, owned by root with mode `0700`, and survives process and device restarts. Both services receive SIGTERM and have 30 seconds to stop before forced termination. Original HermesFS remains an independent filesystem client on port `9901` and is not a telemetry-agent prerequisite.

## systemd

Install the units under `/etc/systemd/system`, run `systemctl daemon-reload`, and enable both units if they should be boot candidates. The marker still keeps them inactive.

otel-relay uses `Restart=on-failure` to recover its durable spool. The agent uses `Restart=no` to avoid an OOM restart loop. It is ordered after otel-relay but remains alive during a relay restart, relying on its bounded local retry until loopback ingress returns.

Canary activation:

```sh
install -m 0644 /dev/null /etc/rdk-otel/enabled
systemctl start otel-relay.service
systemctl start telemetry-agent-otlp.service
```

Rollback removes the marker and stops the agent before otel-relay. Stopping does not delete the spool:

```sh
rm -f /etc/rdk-otel/enabled
systemctl stop telemetry-agent-otlp.service otel-relay.service
```

## SysV init

Install the scripts under `/etc/init.d` with mode `0755` and register them with the platform service manager. LSB headers order `telemetry-agent-otlp` after `otel-relay`. Start and rollback use the same marker and service order as systemd. The scripts reject a token not owned by UID 0 with mode `0600` and use `TERM/30/KILL/5` stop handling.

## Identity migration

Authorize both `mac:<device-id>/hermesfs` and `mac:<device-id>/otel-relay` at the gateway before migration. Stop telemetry-agent, keep the legacy relay running until it has no pending or in-flight records, stop it, and inspect `/var/lib/hermesfs-otel` offline under exclusive ownership. Start otel-relay only after the legacy non-terminal count is zero. Never copy a pending record or resend its message ID under the other source. Rollback after otel-relay admission requires the symmetric drain under `/otel-relay` before restoring another source.

## Verification

Run the repository checks with:

```sh
tools/validate-device-otel-services.sh
```

On each target image, repeat installation with the marker absent, confirm neither service starts, create the marker, and submit one canonical request. Restart otel-relay during relay and confirm diagnostics report recovered work before starting the agent restart check. Remove the marker and stop both services; verify `/var/lib/otel-relay/spool` and committed files remain.
