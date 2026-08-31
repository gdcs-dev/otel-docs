# OTEL Edge Telemetry Standards Baseline

This file records the dependency and specification versions used by the edge
telemetry pipeline. Updates are intentional compatibility changes and require
the shared Rust/Go fixtures and target builds to pass before rollout.

## Device Toolchain

- Rust: `1.90.0`
- telemetry_agent release target: `aarch64-unknown-linux-musl`
- otel-relay release target: `aarch64-unknown-linux-gnu`
- Original HermesFS verification target: `aarch64-unknown-linux-gnu`
- Minimum supported Rust version of the selected OpenTelemetry crates: `1.75.0`

The telemetry component manifests declare Rust `1.90` so local Cargo validation
and release builders enforce the same supported baseline.

## Rust OpenTelemetry Packages

| Concern | Package | Version | Enabled features |
| --- | --- | --- | --- |
| Metrics API | `opentelemetry` | `0.32.0` | `metrics` |
| Metrics SDK | `opentelemetry_sdk` | `0.32.1` | `metrics` |
| OTLP/HTTP exporter | `opentelemetry-otlp` | `0.32.0` | `metrics`, `http-proto`, `reqwest-blocking-client` |
| OTLP protobuf types | `opentelemetry-proto` | `0.32.0` | `metrics`, `gen-tonic-messages` |
| Protobuf runtime | `prost` | `0.14.4` | default |
| Semantic constants | `opentelemetry-semantic-conventions` | `0.32.1` | `semconv_experimental` |

Default OpenTelemetry features are disabled. telemetry_agent enables only the
metrics API, synchronous periodic metrics export, and OTLP/HTTP binary protobuf.
otel-relay enables only generated OTLP metrics messages because it validates and
relays requests but does not use an OpenTelemetry SDK. Original HermesFS has no
OpenTelemetry dependency.

## Protocol and Semantic Specifications

- OTLP specification: `1.11.0`
- OTLP encoding: binary protobuf over HTTP for standard local and cloud hops
- OpenTelemetry semantic conventions: `1.42.0`
- Schema URL: `https://opentelemetry.io/schemas/1.42.0`

The selected semantic-conventions crate is generated from release `1.42.0`.
The `semconv_experimental` feature is required because applicable `system.*`
metrics remain Development status. Production metric definitions must remain
pinned to this release until a reviewed dependency update changes this file,
the metric catalog, and the compatibility fixtures together.