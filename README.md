# TCP-AAD: Adaptive Delayed-Acknowledgment Scheme for TCP over Wi-Fi

Bachelor's thesis project, Innopolis University (2024-2025): **"Performance evaluation of a delayed acknowledgment scheme of TCP over Wi-Fi"**.

This repository is a fork of the mainline Linux kernel (v6.12.9) with a modified TCP delayed-ACK (DACK) mechanism, evaluating an adaptive alternative to the stock ACK-timeout (ATO) heuristic under the variable, bursty latency conditions typical of Wi-Fi links.

## Problem

The default Linux DACK implementation derives its ACK timeout (`ato`) from a fixed heuristic based on recent RTT samples (jiffies-granularity, `net/ipv4/tcp_input.c: tcp_event_data_recv()`). Under Wi-Fi, where inter-packet latency is far less stable than on wired links, this heuristic reacts slowly to actual arrival patterns, leading to a suboptimal balance between ACK overhead and delayed-delivery latency.

## Approach

- Replaced the low-resolution `timer_list`-based delayed-ACK timer with a high-resolution `hrtimer`, enabling microsecond-level scheduling (`include/net/inet_connection_sock.h`).
- Widened the `ato` field from a fixed 8-bit bitfield (`ato:ATO_BITS`) to a full `__u32`, since the new algorithm operates in microseconds rather than jiffies.
- Added new per-connection state to track real inter-arrival times (IAT) between received packets: `iat_min`, `iat_curr`, `delayed_segs`, `last_reset_time`.
- Implemented an adaptive ATO estimator that computes the delayed-ACK timeout as a weighted blend of the minimum and current observed IAT (`ato = (iat_min * 0.75 + iat_curr * 0.25) * 1.5`, capped at 500ms), instead of the stock RFC-based heuristic.
- `iat_min` is periodically reset (~every 1s) so the estimator stays responsive as network conditions change.

## Results

Increased throughput by ~9% in baseline conditions and by ~100% under delay-heavy / high-latency Wi-Fi conditions, compared to the default Linux DACK implementation.

## Branches

- **`feature_logs`** (default) — most complete implementation: hrtimer-based adaptive ATO with `pr_info` instrumentation added during benchmarking/debugging. The logging is verbose by design (used to trace IAT/ATO values during experiments) and can be stripped out for a production build.
- `feat` — earlier iteration of the same idea, storing adaptive-ATO state on `tcp_sock` instead of `inet_connection_sock`, still using the original low-resolution timer.
- `master` — clean upstream Linux v6.12.9 baseline, kept for reference/diffing against the modified branches.

## Where to look

- `net/ipv4/tcp_input.c` — `tcp_event_data_recv()` / `tcp_send_ack()`: adaptive ATO computation.
- `include/net/inet_connection_sock.h` — `icsk_ack` struct: new IAT-tracking fields, `hrtimer` conversion.
- `include/net/tcp.h` — `ATO_BITS` / `TCP_DELACK_MAX` assertion, relaxed to support the wider `ato` range.

## Building

Standard Linux kernel build (`make menuconfig`, `make -j$(nproc)`). See the root `README` for general kernel build/config instructions.
