# OTel Edge Telemetry Protocol

## Hop Matrix

| Hop | Protocol | Sender | Receiver | Success boundary |
| --- | --- | --- | --- | --- |
| Device local | loopback OTLP/HTTP binary protobuf at `POST /v1/metrics` | telemetry_agent | otel-relay | Request bytes and metadata are durably committed to the device spool |
| Device to cloud | private Xmidt relay using `application/vnd.rdk.otel-relay.v1+protobuf` | otel-relay | telemetry gateway | The exact envelope is transactionally admitted to PostgreSQL |
| Cloud upstream | upstream OTLP/HTTP binary protobuf at `POST /v1/metrics` | telemetry gateway | observability backend | The backend returns an OTLP HTTP success status |

The standard hops use `ExportMetricsServiceRequest` and standard OTLP HTTP media and content encodings. The private relay carries a versioned envelope with message ID, signal, media type, encoding, accepted timestamp, SHA-256 digest, and the immutable OTLP body. Xmidt transports the envelope as opaque private-hop content. Neither otel-relay nor the gateway decodes and re-encodes the admitted body for forwarding.

## Ownership And Acknowledgments

The local acknowledgment records durable ownership in otel-relay. telemetry_agent may release its request only after otel-relay has authenticated it, validated its bounded structure, committed its exact bytes and metadata, and returned local success. A lost HTTP response leaves ownership ambiguous and permits the agent to submit the same metric points again.

The cloud acknowledgment records durable ownership in PostgreSQL. A positive Xmidt response means the gateway has authenticated the transport source, authorized resource identity, verified the envelope and digest, and committed the row plus acknowledgment outbox. It does not mean the observability backend has accepted the request. otel-relay may remove the spool record only after an authenticated positive or permanent acknowledgment. Lost cloud responses and reconnects can cause the same envelope to reach admission again; the full authenticated source plus message ID and body digest deduplicate that private-hop retry.

otel-relay registers service `otel-relay` on Parodus client port `9902`, emits `mac:<device-id>/otel-relay`, sends to `event:rdk-otel-relay/v1`, and accepts acknowledgments only from `dns:telemetry-gateway`. Original HermesFS may coexist as service `hermesfs` on port `9901`, but does not own current telemetry.

Changing from legacy `mac:<device-id>/hermesfs` to `/otel-relay` changes the deduplication key. Stop telemetry-agent, drain all pending and in-flight legacy records, stop the legacy owner, and verify the legacy spool offline under exclusive access before new admission. Never move a durable record or reuse its message ID under the other source. Apply the same drain rule before rollback after otel-relay has accepted work.

An authenticated permanent acknowledgment ends automatic device retries and retains bounded terminal metadata. Retryable and unknown outcomes retain device ownership and use bounded backoff. PostgreSQL retains cloud ownership through export retries and expired worker-lease recovery. A permanent upstream rejection moves the cloud row to dead letter without asking the already acknowledged device to resend.

Delivery across each ambiguous network boundary is at-least-once. No end-to-end exactly-once guarantee exists: an upstream request can be accepted even when its response is lost, so a later cloud retry can duplicate final backend delivery. Stable cumulative streams and point timestamps reduce the effect; they do not change the guarantee.

## Metric Catalog Version Lifecycle

The metric catalog version lifecycle is independent of transport envelope evolution. A catalog version owns instrument names, descriptions, UCUM units, point kinds, monotonicity, temporality, approved attributes, value bounds, and cardinality ceilings. Runtime policy may enable a collector or tune its interval, but may not redefine metric identity under an existing version.

A compatible catalog update adds reviewed metrics or collectors without changing existing stream identity. An incompatible descriptor, aggregation, attribute, or temporality change requires a new catalog version, regenerated fixtures, independent protobuf decode checks, staged decoded-metric comparison, and support for the prior version through the fleet retention window. Rollback selects the prior pinned catalog and does not reinterpret queued OTLP bytes.

OpenTelemetry semantic conventions remain pinned to `1.42.0` with schema URL `https://opentelemetry.io/schemas/1.42.0`; the Rust semantic-conventions package remains `0.32.1`. Changing the pin requires one reviewed update to the standards baseline, catalog, generated fixtures, conformance tests, and rollout comparison. Experimental `system.*` definitions remain explicitly reviewed and cannot drift through an unpinned dependency update.

## Compatibility Rules

Relay minor versions may add protobuf fields because receivers preserve unknown fields and validate known required fields. A major envelope change requires simultaneous sender and gateway support before activation. OTLP unknown fields are accepted according to protobuf compatibility, and exact admitted body bytes remain authoritative for upstream forwarding.

Backend credentials and tenant routing exist only in upstream HTTP headers. Local bearer credentials, Xmidt authentication, and cloud backend credentials are independent rotation boundaries. Credentials, provisioning data, and OTLP payload values are excluded from logs and bounded diagnostics.
