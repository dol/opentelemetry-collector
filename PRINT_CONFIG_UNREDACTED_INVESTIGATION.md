# Print-Config Unredacted Investigation

## Scope
Investigate the second bug around `otelcol print-config --mode unredacted` for tail-sampling config rendering.

The observed issue is separate from the contrib-side marshaling bug in `tailsamplingprocessor.PolicyCfg`.

## Confirmed Findings

### 1. Contrib has a real redacted-path marshaling bug
In `opentelemetry-collector-contrib/processor/tailsamplingprocessor/config.go`, `PolicyCfg` embeds an unexported `sharedPolicyCfg` with `mapstructure:",squash"`.

That unexported field carries:
- `name`
- `type`
- the real policy payloads such as `probabilistic`, `status_code`, `latency`, `string_attribute`, etc.

`confmap`'s generic encoder only serializes struct fields for which `field.CanInterface()` is true. In `go.opentelemetry.io/collector/confmap@v1.56.0/internal/mapstructure/encoder.go`, see:

- `encodeStruct`
- the `if field.CanInterface()` guard

This means generic `confmap.Marshal` called from outside the `tailsamplingprocessor` package drops the embedded unexported `sharedPolicyCfg` entirely.

That directly explains the broken `print-config` output for the redacted path.

### 2. Repro for the redacted-path bug is complete
A tiny program run from the nested module `processor/tailsamplingprocessor` that does `confmap.Marshal` on `[]PolicyCfg` reproduces this exact broken structure:

```yaml
policies:
  - and:
      and_sub_policy: []
    composite:
      composite_sub_policy: []
      max_total_spans_per_second: 0
      policy_order: []
      rate_allocation: []
    drop:
      drop_sub_policy: []
    not:
      not_sub_policy: {}
```

So the redacted-path issue is fully explained by contrib config marshaling.

## The Remaining Core-Side Bug

The second bug is that `--mode unredacted` also produces broken or unexpectedly interpreted output for tail sampling.

This should not happen according to core source intent.

### Core implementation inspected
In `opentelemetry-collector/otelcol/command_print.go`:

- `printRedactedConfig()`:
  - gets typed config via `getPrintableConfig()`
  - calls `confmap.Marshal(cfg)`
- `printUnredactedConfig()`:
  - if validation is enabled, it validates a typed config first
  - then creates a fresh resolver with `confmap.NewResolver(...)`
  - calls `resolver.Resolve(...)`
  - prints `conf.ToStringMap()`

That means unredacted mode is intended to print the resolved raw confmap before typed marshaling.

### Core test expectation inspected
In `opentelemetry-collector/otelcol/command_print_test.go`, the `default field value` test explicitly says:

> Since the structure is empty before interpretation, no default is expanded.

And the expected unredacted output is just:

```yaml
e: null
```

So upstream core intent is clear: unredacted mode should not go through typed default expansion.

## Runtime Evidence From The Real Binary

Using the real `otelcol-contrib-0.150.1` binary from:

- `/home/dominicluechinger/project/observability/otelcol-gateway-infra/main/.ci-tools/otelcol-contrib-0.150.1`

`go version -m` shows:

- `go.opentelemetry.io/collector/otelcol v0.150.0`
- `go.opentelemetry.io/collector/confmap v1.56.0`

### Minimal repro file
A minimal config file was created with only a tail-sampling processor and two simple policies:

- `status_code`
- `probabilistic`

Running:

```bash
./.ci-tools/otelcol-contrib-0.150.1 print-config --mode unredacted --validate=false --config /tmp/tail-sampling-repro-YrN1.yaml
```

still produced:

- defaulted top-level tail sampling fields such as:
  - `decision_wait: 30s`
  - `num_traces: 50000`
  - `sample_on_first_match: false`
  - `sampling_strategy: trace-complete`
- and the same broken anonymous policy-union rendering

That is inconsistent with the core test expectation for unredacted mode.

## What Is Proven vs Inferred

### Proven
- The contrib redacted-path bug is real and rooted in unexported squashed fields.
- The actual shipped binary also shows incorrect behavior in `--mode unredacted` on a minimal repro.
- Core source says unredacted mode should print resolved raw confmap, not typed/defaulted config.

### Inferred but not yet pinned to exact line
The shipped `0.150.x` binary appears to have a second issue where the `print-config --mode unredacted` path is no longer behaving like the source contract/tests suggest.

Possible explanations:
- the release/distribution wiring is not actually using the plain resolver path expected by upstream source
- a converter in resolver settings mutates config into a more interpreted/defaulted form before printing
- the exact `0.150.0` source used in the binary differs from the nearby locally inspected source snapshot
- the issue arises in `confmap.Resolve` or subsequent config conversion before `ToStringMap()`

## Suggested Next Investigation Steps

1. Pin the exact `otelcol v0.150.0` source used by the shipped binary.
2. Reproduce the unredacted bug inside the core repo with a focused unit test under `otelcol/command_print_test.go`.
3. Trace the actual resolver settings and converters active in `print-config --mode unredacted` for the contrib distribution.
4. Determine whether the unexpected defaults come from:
   - resolver conversion
   - release/distribution setup
   - a divergence between source and shipped artifact
5. If confirmed, fix core so `--mode unredacted` preserves raw resolved config semantics.

## Candidate Test To Add
A focused core regression test should:

- define a minimal tail-sampling config file with two explicit policies
- run `print-config --mode unredacted --validate=false`
- assert that output contains:
  - `name`
  - `type`
  - `probabilistic.sampling_percentage`
  - `status_code.status_codes`
- assert that unconfigured typed defaults are not expanded if the design contract still stands

If the intended contract has changed, the test should be updated to the new desired behavior and the command documentation should be aligned.

## Prompt For Next Session
Use this worktree to investigate the core-side `print-config --mode unredacted` bug.

Start from the evidence in this file and do the following:

1. Reproduce the bug inside the core repo with an automated test.
2. Identify the exact code path that causes unredacted mode to emit typed/defaulted tail-sampling output.
3. Decide whether the bug is in:
   - `otelcol/command_print.go`
   - resolver/converter setup
   - release/distribution wiring
   - or an interaction with `confmap`
4. Implement the smallest defensible fix.
5. Add regression coverage.
6. Summarize whether the contrib marshaling fix and the core unredacted fix should land independently.
