# OTel Edge Telemetry Operations

This runbook covers the normalized alert contract in `deploy/observability/otel-pipeline-alerts.yaml`. Poll telemetry-agent and otel-relay diagnostics on-device and the restricted gateway diagnostics endpoint in cloud operations. Export only the listed bounded counters and gauges; never export credentials, OTLP bodies, attribute values, or provisioning records.

Counter rules use non-negative deltas over the rule interval. Treat a reset as zero. Queue utilization is the greater of byte and record utilization. Oldest age is the greater of the device spool and cloud unfinished queue ages. Absence of every source is a separate telemetry-scrape failure, not a healthy zero.

<a id="queue-saturation"></a>
## Queue Saturation

Confirm whether bytes or record count is limiting and whether backpressure is increasing. Check Xmidt connectivity and gateway readiness. Restore relay capacity or backend flow before changing limits; increasing the spool only postpones exhaustion. Do not delete queued records.

<a id="oldest-age"></a>
## Oldest Age

Locate the oldest owner: otel-relay before cloud acknowledgment, or PostgreSQL after acknowledgment. For device ownership, inspect spool utilization, oldest-record age, relay attempts, Parodus connectivity, and acknowledgment state. For cloud ownership, inspect worker leases, export retries, and upstream health. Restart a stuck worker only after confirming leases recover automatically.

<a id="clock-invalidity"></a>
## Clock Invalidity

Check wall-clock synchronization, boot sequencing, and time-service health on affected devices. Metrics rejected before valid wall time cannot be reconstructed. Do not fabricate timestamps or switch cumulative streams to delta temporality.

<a id="cardinality-overflow"></a>
## Cardinality Overflow

Identify the metric name and approved attribute whose values expanded. The overflow series preserves accounting under `otel.metric.overflow=true`. Correct policy or collector normalization; do not raise limits until memory and request-size impact is measured.

<a id="authentication-failure"></a>
## Authentication Failure

Separate loopback bearer, Caduceus webhook, and upstream credential failures. Check recent rotations and file permissions, then compare failure scope by component. Never print a token during diagnosis. Revoke suspected credentials and rotate one boundary at a time.

<a id="identity-mismatch"></a>
## Identity Mismatch

Compare the authenticated Xmidt source with the provisioning-map entry and decoded envelope host IDs. Quarantine repeated mismatches. Correct provisioning only after ownership is independently verified; payload identity is never authoritative authentication.

<a id="relay-retry"></a>
## Relay Retry

Check Xmidt reachability, gateway readiness, and acknowledgment publication. A retry can be normal after an ambiguous response and may create duplicate delivery. Verify one durable cloud row per `(device_id, message_id)` rather than expecting exactly-once transport.

<a id="dead-letter"></a>
## Dead Letter

Use restricted message-state lookup with authenticated device ID and message ID. Classify permanent upstream rejection versus exhausted retry window. Retain the original row for remediation and audit. Replay only after correcting the cause and preserving the exact admitted body and metadata.

<a id="permanent-drop"></a>
## Permanent Drop

Distinguish authenticated permanent acknowledgment, retention expiry, and local administrative removal using otel-relay diagnostics. Confirm cloud ownership before any manual device cleanup. Expired or permanently rejected records are terminal and must remain in conservation accounting.

<a id="backend-rejection"></a>
## Backend Rejection

Inspect the bounded HTTP reason in message state and the upstream service release history. Validate media type, encoding, tenant headers, and catalog compatibility without logging the body. A permanent 4xx becomes cloud dead letter and must not ask otel-relay to resend an already acknowledged envelope.

## Alert Verification

Run the deterministic injection check whenever rules, panels, or this runbook change:

```sh
python3 tools/validate-otel-alerts.py
```

The check proves all ten required rules remain linked to a panel and procedure, remain quiet for a normal sample, and fire for the declared injected sample. Cluster promotion should additionally inject one real condition per environment, beginning with a synthetic backend 400 and a paused export worker, and confirm notification routing before restoring service.
