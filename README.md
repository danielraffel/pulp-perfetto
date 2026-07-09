# pulp-perfetto

Perfetto-based DSP tracing for [Pulp](https://github.com/danielraffel/pulp) audio
plugins — see, per audio block, exactly which DSP node ate the time. A scalar CPU
meter tells you *that* you're at 40%; the trace tells you *why*.

This is a showcase of the tracing tooling built into Pulp. Everything here is real
output from real traces — no mockups.

---

## What you get

- **`pulp trace` CLI** — `start` / `stop` / `query` / `explain` / `snapshot` /
  `doctor` / `open` / `fetch`, plus one-line presets (`slowest-frames`, `xruns`,
  `dsp-hotspots`, `layout-vs-paint`).
- **Offline DSP flamegraphs** — a deterministic, deadline-free render of your
  plugin, instrumented with per-node `dsp.node` spans, flushed to a `.pftrace` you
  open in [ui.perfetto.dev](https://ui.perfetto.dev).
- **SQL over your trace** — the same `trace_processor` engine the Perfetto UI uses,
  from the command line: `pulp trace query "<sql>" --trace run.pftrace`.
- **Zero-install** — `pulp trace` fetches a pinned, hash-verified
  `trace_processor` (Perfetto v57.2) on first use; nothing to `brew install`.
- **RT-safe by design** — only the *offline* render path is Perfetto-traced. The
  live `process()` callback is never instrumented (a `TRACE_EVENT` takes a lock at
  buffer rollover and is not real-time-safe); live plugins use a separate
  fixed-slot telemetry path instead.

---

## Install

Tracing is a **dev-only, opt-in** build option (never linked into a shipping
plugin). Build the tracing examples:

```bash
git clone https://github.com/danielraffel/pulp && cd pulp
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DPULP_TRACING=ON
cmake --build build --target pulp-trace-plugin-chain -j"$(getconf _NPROCESSORS_ONLN)"
```

The `pulp trace` CLI ships with the SDK. On first `query` it fetches the pinned
`trace_processor` automatically (or run `pulp trace fetch` up front).

---

## Use case 1 — "The load meter said 40%. The trace said *why*."

`examples/trace-demo` renders an offline poly-synth: three cheap sine voices plus
one deliberately expensive 8×-oversampled lead. The average load looks calm, but
the per-block flamegraph shows one node eating every block.

```bash
./build/examples/trace-demo/pulp-trace-demo 0.06 /tmp/demo.pftrace
pulp trace query \
  "select name, count(*) blocks, sum(dur)/1000.0 total_us, round(avg(dur)/1000.0,2) us_per_block \
   from slice where name in ('voice_oscillators','lead_oversampler','mix_bus') \
   group by name order by total_us desc" \
  --trace /tmp/demo.pftrace
```

The `lead_oversampler` node costs **~6× the oscillators and ~87× the mix bus** —
invisible to an average, obvious in the trace.

![Perfetto timeline + per-node cost](assets/perfetto-timeline-and-breakdown.png)

![Per-node cost in Perfetto's SQL view](assets/perfetto-sql-breakdown.png)

---

## Use case 2 — "Which plugin in my chain is expensive?" (and why the average lies)

`examples/trace-plugin-chain` profiles a **real** effect chain — `PulpGain` →
`PulpEffect` (a biquad filter) → `PulpCompressor` — driven through each plugin's
actual `process()` code, one `dsp.node` span per plugin per block.

The trace catches a truth a CPU meter inverts:

![Real plugin chain: average vs steady state](assets/perfetto-real-plugin-chain-sql.png)

`gain`'s **average** (33 µs) looks like the worst offender — but nearly all of it
is a one-time cold-start spike on the very first block (the first `process()` call
warms caches and touches fresh pages). Its **steady-state** cost is 0.21 µs. The
real per-block hot node is the biquad filter (5.7 µs), then the compressor
(4.1 µs). The per-block trace separates one-time warmup from the cost that
actually recurs — the whole reason to reach for a trace over an average.

---

## How it works, briefly

- Wrap an offline render in a session: `pulp::runtime::Tracing::start({"dsp","dsp.node"}, path)`
  … render … `Tracing::stop()`.
- Annotate DSP stages with `PULP_TRACE_SCOPE_NAMED("dsp.node", "my_stage")` — a
  no-op when `PULP_TRACING=OFF`, so it costs nothing in shipping builds.
- Open the `.pftrace` in [ui.perfetto.dev](https://ui.perfetto.dev), or query it
  from the CLI, or ask `pulp trace explain "why is my plugin slow?"` for a
  plain-English answer.

Full guide: [`docs/guides/tracing.md`](https://github.com/danielraffel/pulp/blob/main/docs/guides/tracing.md).

---

*Built with [Pulp](https://github.com/danielraffel/pulp) · traces rendered in
Perfetto v57.2.*
