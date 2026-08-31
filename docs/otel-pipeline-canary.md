# OTel Pipeline Canary

Use `deploy/rollout/otel-pipeline-canary.yaml` as the promotion contract. This order is mandatory: authorize both relay identities and deploy the gateway first, drain and offline-validate any legacy spool, deploy otel-relay with telemetry_agent still off, then enable telemetry_agent on selected devices. Expand one device model and percentage step at a time.

For each canary, record the baseline and a minimum 30-minute enabled window. Decode the source fixture and recorded backend request independently and compare every resource, instrumentation scope, metric descriptor, attribute, timestamp, and value listed in the plan. Exact request bytes must match from otel-relay admission through gateway forwarding.

During the same window, record telemetry_agent RSS, otel-relay spool byte/record utilization and oldest age, Parodus connectivity, gateway CPU, and terminal outcome deltas. Do not promote on missing samples. Stop when any configured maximum is exceeded or identity mismatch, permanent drop, or backend rejection increases.

Run the local non-production proof before a fleet stage:

```sh
TEST_DATABASE_URL='postgres:///root?host=/var/run/postgresql' \
  bash tools/validate-otel-canary.sh
```

Rollback first disables callback registration so no new cloud admission begins, then disables telemetry_agent before otel-relay. Keep otel-relay running under `/otel-relay` until every accepted record is terminal; only then may another source resume admission. Retain both device state directories and gateway policies through their retention windows. Keep gateway workers and PostgreSQL rows until cloud work is exported or dead-lettered. Image rollback and service disablement must never delete either durable queue.
