# OTLP Pipeline Performance Baseline

This baseline makes pipeline capacity decisions reproducible. It is not a substitute for target-device endurance testing or a production-sized PostgreSQL load test.

Run all probes with:

```sh
TEST_DATABASE_URL='postgres:///root?host=/var/run/postgresql' \
  tools/benchmark-otel-pipeline.sh
```

The PostgreSQL URL must name an isolated database. The integration benchmarks create and remove private schemas.

## Reference results

The reference run used optimized Rust builds and Go `linux/arm64` in the Ubuntu 24.04 development container, with PostgreSQL 16 reached over a local Unix socket. Values are average throughput, not tail latency.

| Path | Workload | Reference result |
|---|---|---:|
| Collection and catalog aggregation | Parse `/proc/meminfo`; emit 8 mapped metrics | 104,252 collections/s; 834,019 metrics/s |
| OTLP assembly and gzip | Two cumulative metrics per request | 49,754 requests/s; gzip ratio 0.583 |
| Durable commit | 1,024 8 KiB records, group 1 | 994 records/s |
| Durable commit | Same records, group 8 | 4,748 records/s |
| Durable commit | Same records, group 32 | 9,233 records/s |
| Spool recovery | 1,024 committed 8 KiB records | 39-45 ms |
| Private relay preparation | Encode and enqueue an 8 KiB envelope to a synthetic sender | 1,507,859 messages/s |
| Gateway envelope validation | Canonical protobuf envelope and SHA-256 | 414 ns/op; 1.09 GiB/s |
| PostgreSQL admission | Fresh durable rows | 2,592 requests/s |
| Fleet reconnect | 16 concurrent devices replay distinct accepted rows | 8,456 requests/s |
| PostgreSQL export | Claim, local OTLP HTTP success, complete; 1 worker | 1,037 requests/s |
| PostgreSQL export | Same path; 4 workers | 2,255 requests/s |

The relay preparation result excludes Xmidt network latency. Live canary measurements own that budget; the synthetic probe only detects local encoding or queueing regressions.

## Budgets

Device release gates on each target platform are:

- Agent RSS stays below the configured 48 MiB fail-safe ceiling.
- Representative collection plus aggregation and OTLP assembly each remain below 10 ms p95.
- A one-request durable commit remains below 100 ms p95 and recovery of 1,024 records remains below 1 second.
- Gzip remains enabled only while representative request size is at most 90% of identity and compression remains below 5 ms p95.
- The 64 MiB spool is the flash budget. Endurance tests must include synchronized appends, state transitions, and whole-segment reclamation.

Cloud reference gates per gateway replica are 2,000 fresh admissions/s, 5,000 duplicate reconnect admissions/s at 16 concurrent clients, and 2,000 successful exports/s with four workers. The reference run passes these gates. Production load tests must repeat them with networked PostgreSQL and the real Xmidt and OTLP services before rollout.

## Selected defaults

| Control | Default | Rationale |
|---|---:|---|
| Agent export interval | 60 s | Bounds request rate and follows the OTLP design baseline. |
| Agent observation queue | 8,192 batches | Existing measured memory envelope; overflow is explicit. |
| Agent compression | gzip | 0.583 reference ratio at about 20 microseconds/request. |
| Agent local retry | 1 s initial, 30 s maximum, 5 min elapsed | Covers transient local restart without competing with durable otel-relay retention. |
| otel-relay spool | 64 MiB, 16,384 records | Hard flash cap plus enough record slots for 10,080 one-minute exports over seven days. |
| otel-relay retention | 7 days | Maximum outage age; byte or record saturation can occur first and returns backpressure. |
| otel-relay segment | 4 MiB | Limits recovery and reclamation units while amortizing file metadata. |
| otel-relay commit | Immediate, one request per synchronized group | Lowest acknowledgment ambiguity for one agent exporting once per minute; group 8 and 32 remain measured tuning options for higher-rate deployments. |
| otel-relay relay | 4 in flight; 30 s acknowledgment timeout | Bounds device memory and duplicate window. |
| otel-relay retry | 1 s initial, 60 s maximum, 24 h elapsed | Fast reconnect with bounded retry pressure; records remain until acknowledgment, expiry, or explicit loss. |
| Gateway dedupe retention | 8 days | Covers seven-day device retention plus one-day operations and clock margin. |
| Gateway workers | 4 acknowledgment and 4 export workers | Four export workers more than doubled reference throughput and passed the 2,000/s gate. |
| Gateway worker timing | 30 s lease; 250 ms idle poll | Recovers interrupted work without hot polling. |
| Gateway upstream retry | 1 s initial, 60 s maximum, 24 h elapsed | Bounded backend retry with durable ownership retained. |

At a 60-second interval, seven days produces 10,080 records. The 64 MiB byte budget supports that duration only when average encoded record size, including spool overhead, stays below about 6.5 KiB. Larger requests reach byte backpressure earlier; the age setting is not a promise of seven days at the 512 KiB request maximum.

All throughput defaults retain at-least-once behavior. Neither worker concurrency nor deduplication makes final backend delivery exactly once.

## Conservation soak

Run the deterministic saturation and terminal-accounting suite with:

```sh
TEST_DATABASE_URL='postgres:///root?host=/var/run/postgresql' \
  tools/soak-otel-pipeline.sh
```

The device soak repeats 100 fill, saturation, retry, expiry, restart-recovery, and segment-reclamation cycles. Its reference result was:

```text
submitted=6500 accepted=6400 backpressured=100 retries=1600
cloud_accepted=3200 dead_lettered=1600 expired=1600
```

It asserts `submitted = accepted + backpressured` and `accepted = cloud_accepted + dead_lettered + expired`, then verifies the recovered spool is empty. Retry count is additional attempt accounting, not a terminal outcome.

The cloud soak durably admits 300 rows and drives 100 direct successes, 100 retry-then-success responses, and 100 permanent backend rejections. It asserts 200 exported rows, 100 dead letters, zero unfinished rows, and 400 recorded export attempts. The same suite verifies agent cardinality overflow accounting plus gateway envelope, identity, and replay-conflict rejection paths.
