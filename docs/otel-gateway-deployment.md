# Telemetry Gateway Deployment

`deploy/gateway/kubernetes` is the production baseline. Set an immutable image digest or release tag in `kustomization.yaml`; do not deploy the `replace-me` policy or image values.

## Required inputs

Create `telemetry-gateway-secrets` through the cluster secret manager with these keys:

- `database-url`
- `argus-basic-auth`
- `webhook-secret`
- `diagnostics-token`
- `scytale-basic-auth`
- `otlp-upstream-endpoint`
- `otlp-upstream-authorization`

Do not commit rendered Secret resources. Replace `telemetry-gateway-device-policy` with the non-production provisioning export before applying the Deployment. Each authenticated Xmidt source needs its authorized `host_ids` and upstream tenant.

Every pod runs `/telemetry-gateway-migrate` before the gateway. Migrations use a PostgreSQL advisory lock and ledger, so rolling pods may execute the idempotent command concurrently. Migration failure prevents the pod from accepting traffic.

PostgreSQL is the authoritative relay journal. Prometheus does not replace it: time-series storage cannot represent message deduplication, original request bytes, acknowledgment state, leases, retry schedules, or dead letters. Cleanup only removes rows where export is `exported`, acknowledgment is `published`, and `accepted_at` is strictly older than `now - GATEWAY_DEDUPE_RETENTION`. Dead letters and incomplete rows are never removed automatically. `GATEWAY_RETENTION_CLEANUP_INTERVAL` defaults to `1h`, and `GATEWAY_RETENTION_CLEANUP_BATCH_SIZE` defaults to `500`.

## Development observability services

The development Collector receives OTLP/HTTP on port `4319` and synchronously sends Prometheus Remote Write to `127.0.0.1:9090/api/v1/write`. Its sending queue, retry helper, and batch processor must remain disabled so an OTLP success means Prometheus accepted the write; PostgreSQL remains the only durable retry owner. Prometheus enables its Remote Write receiver, retains seven days of samples, and accepts out-of-order samples up to eight days old. Prometheus listens only on loopback. Restrict Collector port `4319` and Grafana port `3000` with the host firewall when the host is not isolated.

Install the checked-in service configuration with:

```sh
install -m 0644 deploy/gateway/otel-collector.yaml /etc/otelcol-dev/config.yaml
install -m 0644 deploy/gateway/systemd/otelcol-dev.service /etc/systemd/system/otelcol-dev.service
install -m 0644 deploy/gateway/prometheus.yaml /etc/prometheus/prometheus.yml
install -m 0644 deploy/gateway/prometheus.default /etc/default/prometheus
cp -R deploy/gateway/grafana/provisioning/. /etc/grafana/provisioning/
install -m 0644 deploy/gateway/grafana/dashboards/telemetry-gateway.json /var/lib/grafana/dashboards/telemetry-gateway.json
systemctl daemon-reload
systemctl enable --now prometheus.service otelcol-dev.service grafana-server.service
```

Validate the active development path with:

```sh
/usr/local/bin/otelcol-dev validate --config=/etc/otelcol-dev/config.yaml
promtool check config /etc/prometheus/prometheus.yml
systemctl is-active prometheus.service otelcol-dev.service grafana-server.service
bash tools/validate-prometheus-delivery.sh
```

The gateway diagnostics response exposes payload-free `cleanupAttempts`, `cleanupFailures`, and `cleanupDeletedRecords` totals. A cleanup failure is logged and retried on the next cleanup interval without stopping admission, acknowledgment, or export workers.

## Rollout

For the development stack, enable the Prometheus Remote Write receiver and out-of-order window first, deploy the synchronous Collector configuration second, verify the live validation script, and deploy the gateway cleanup worker last. For Kubernetes, apply the secret and policy first, then the kustomization. The baseline starts two replicas, keeps `maxUnavailable: 0`, permits one surge pod, and requires readiness only after Argus registration. The PDB retains one available replica during voluntary disruption. The HPA scales from 2 to 20 replicas at 65% CPU; queue depth and oldest age remain the authoritative saturation signals.

Before production promotion, run:

```sh
TEST_DATABASE_URL='postgres:///root?host=/var/run/postgresql' \
  tools/validate-gateway-deployment.sh
```

In the non-production cluster, admit a canonical envelope while upstream export is paused, restart the Deployment, resume upstream, and verify the same `(device_id, message_id)` row exports with its original body. Observe no second work item and no loss during the rolling update.

## Rollback

Use `kubectl rollout undo deployment/telemetry-gateway` for a compatible image regression. Database migrations in this change are additive; do not run migration down while any current or prior gateway pod or retained device envelope can reference the schema.

For a development Collector rollback, restore a compatible Prometheus scrape target before replacing the Remote Write Collector. Keep the Prometheus Remote Write receiver enabled until no deployed Collector uses it. Rolling back the gateway binary stops scheduled cleanup but does not require deleting the additive retention index or any journal row.

To stop new admission without deleting durable cloud work, remove or disable the Argus callback registration and scale gateway admission to zero only after workers have drained or a worker-only maintenance deployment is running. Never delete `relay_messages` as part of image rollback. Keep the deployed envelope major version until the maximum device retention and dedupe window have elapsed.
