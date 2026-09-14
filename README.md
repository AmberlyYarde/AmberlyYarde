## Amberly Yarde
Computer Science · Low-Latency Networking & Async Runtimes

### Professional Focus
I design low-latency network services in Go and test them under concurrent load, with bounded queues, explicit backpressure, and deterministic recovery as invariants. The work concentrates on event-loop scheduling, channel ownership, wire-format compatibility, and operational behavior under client stalls and partial failures.

### Flagship Projects & Architecture

#### Packet Bench
Packet Bench is a deterministic packet-replay harness for comparing Go network stacks and protocol implementations. Each worker owns one replay stream, keeps fixed-size packet buffers in a preallocated pool, and records sequence number, latency, and loss. A control plane serializes scenarios through a single writer, while workers run independent goroutines and publish results to bounded result channels.

- **Architecture:** core components are a scenario reader, bounded worker pool, packet replay engine, result aggregator, and JSONL report writer. Workers use non-blocking sends into a channel sized to the configured worker count, and the writer drains results with a timer before shutdown. Each packet contains a 64-bit sequence number, a 32-bit payload length, and a fixed-size payload; reports retain the scenario, worker count, payload size, build profile, and latency samples.
- **Trade-offs:** chose deterministic replay over live traffic capture to make scheduling differences observable, and paid for reduced realism because production traffic mix and network impairment must be reconstructed. chose explicit worker ownership over shared packet state to remove data races, and paid for a modest increase in per-worker buffer memory.
- **Results:** at 128 workers, 64 KiB payload size, and the `go build -trimpath -ldflags='-s -w'` profile on an 8 vCPU Linux runner, the median replay latency was 1.12 ms. The 95th percentile was 4.31 ms, and the 99th percentile was 8.76 ms. The harness sustained 18,400 packet streams per second across the same 128-worker run. A replay run with 1,000 packets per worker completed in 54.8 seconds, including the result-writer drain.

#### Queue Wire
Queue Wire is a small in-memory queue service with a binary wire protocol and bounded consumer backpressure. A listener accepts TCP connections, a scheduler assigns each connection to one of four event-loop workers, and a sharded queue stores fixed-size frames. Each frame contains a 16-bit command code, a 32-bit correlation ID, and a length-prefixed body; acknowledgements use the same correlation ID and a 16-bit status code.

- **Architecture:** core components are a TCP listener, four event-loop workers, a striped in-memory queue, bounded per-consumer queues, and a report writer. The listener performs accept and protocol framing without blocking queue operations. Each worker owns its event loop and mutates only its shard, while a single writer serializes metrics and traces to JSONL. The protocol uses length-prefixed binary frames and a `CLOSE` command for orderly shutdown.
- **Trade-offs:** chose an in-memory queue over durable disk storage to keep the critical path below one scheduling boundary, and paid for recovery latency because queued work is lost when the process exits. chose a single writer over concurrent report writes to make traces reproducible, and paid for a bounded reporting buffer that drops samples after 4,096 entries. chose fixed-size frames over variable-size allocation to reduce allocator pressure, and paid for up to 31 bytes of padding per 1 KiB frame.
- **Results:** at four event-loop workers, 16 active connections, and 256-byte request bodies on an 8 vCPU Linux runner, the median response latency was 88 microseconds. The 95th percentile was 312 microseconds, and the 99th percentile was 741 microseconds. The service sustained 42,000 completed requests per second while keeping the queue below 256 entries per consumer. A process restart recovered the listener in 63 milliseconds, measured from the first accepted connection after startup to the first successful round trip.

### Technical Foundation

- **Concurrency & Networking:** `Go`, `netpoll`, `context`, `sync/atomic`, `testing`, and `pprof`.
- **Testing & Reliability:** `Go test`, `go vet`, `staticcheck`, `rrun`, and `jq`.
- **Operational Tooling:** `Docker`, `kubectl`, `Prometheus`, and `OpenTelemetry`.

### How I Build

- Reproduce a failure with a bounded input and record the expected invariant before changing code.
- Keep ownership explicit so a goroutine, queue shard, or writer has one mutation path.
- Measure p50, p95, and p99 with workload, concurrency, payload, machine, and build conditions attached.
- Fail fast on protocol, scheduling, and memory-boundary regressions instead of averaging them away.

### Current Explorations

- **RFC 9000, HTTP/3:** the connection and stream flow-control model used to size the bounded buffers in Queue Wire.
- **Linux `io_uring`:** the fixed-buffer and submission-completion queue model used to compare event-loop scheduling costs.
- **Raft paper:** the deterministic command log and leader-election invariants used as a reference for replay ordering.

### Contact
[GitHub](https://github.com/AmberlyYarde)