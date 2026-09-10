# log_pipeline_test

End-to-end test cases for the syslog path:

```
switches (RFC 3164, UDP 514)  ─┐
NTP/GPS clocks (RFC 5424, TCP) ─┼─> Fluent Bit ─forward:24224─> Fluentd ─tcp:601─> NESS
local files (tail)             ─┘
```

Three questions, three phases:

1. Did the injected line get in? — Fluent Bit metrics delta
2. Did it arrive at Fluentd? — permanent receipt tap, counted by marker
3. Is the copy going to NESS valid RFC 5424? — raw wire tap, asserted field by field

## Test case matrix

| ID | Phase | What it asserts | Fails when |
|----|-------|-----------------|-----------|
| TC-01 | preflight | `fluent-bit.service` running + enabled | agent down or masked |
| TC-02 | preflight | config parses (`--dry-run`, no restart) | bad config staged but not yet loaded |
| TC-03a | preflight | TCP syslog listener accepting | port not bound |
| TC-03b | preflight | UDP syslog socket bound (`ss`) | `wait_for` can't test UDP; this can |
| TC-04 | preflight | collector reaches Fluentd :24224 | firewall/segmentation break |
| TC-05 | inject | N lines into the tailed file | — |
| TC-06 | inject | N RFC 3164 frames over UDP as a switch | — |
| TC-07 | inject | N RFC 5424 frames + SD over TCP as a clock | — |
| TC-08 | fluentbit | forward output `proc_records` advanced | ingested but not shipped |
| TC-09 | fluentbit | no new `errors` / `retries_failed` | backpressure or TLS failure |
| TC-10 | fluentd | marker present in the receipt tap | forward path broken |
| TC-11 | fluentd | count at Fluentd == count injected | silent loss |
| TC-12 | fluentd | all three sources contributed N each | one dead input |
| TC-13 | fluentd | no plugin in retry (monitor_agent, optional) | queued not delivered |
| TC-14 | format | all frames present on the NESS egress | dropped by a Fluentd filter |
| TC-15 | format | full RFC 5424 grammar match | framing wrong |
| TC-16 | format | `<PRI>` in 0–191, decodes to expected facility/severity | Fluentd stamped its own default |
| TC-17 | format | VERSION == 1 | still emitting 3164 |
| TC-18 | format | HOSTNAME is the device, not the collector | **the usual failure** |
| TC-19 | format | APP-NAME preserved | tag not mapped |
| TC-20 | format | STRUCTURED-DATA is `-` or `[SD-ID ...]` | empty field instead of NILVALUE |
| TC-21 | format | timestamp within skew of injection | 3164 year/TZ inferred wrong |
| TC-22 | format | MSG body intact | truncation |

## Prerequisites

The test role reads three things it does not create. Deploy them from your
pipeline roles first — TC-preflight fails loudly if they are missing:

| Template | Where | Purpose |
|----------|-------|---------|
| `fluent-bit-probe-input.conf.j2` | collectors | tail input + `HTTP_Server On` |
| `fluentd-rfc5424-builder.conf.j2` | Fluentd | assembles the 5424 frame |
| `fluentd-probe-tap.conf.j2` | Fluentd | receipt tap + wire tap |
| `rfc5424-format.conf.j2` | Fluentd | the one shared `<format>` block |
| `rsyslog-wire-capture.conf.j2` | Fluentd | raw-byte listener on 127.0.0.1:5514 |

This split is deliberate. If the test role edited Fluentd config and reloaded
the service, it would be testing a configuration that only exists during the
test. Making the taps permanent means what you validate is what runs.

## Running

```bash
# read-only, generates nothing
ansible-playbook -i inventory.ini test_log_pipeline.yml --tags preflight

# canary first
ansible-playbook -i inventory.ini test_log_pipeline.yml --limit collector-01

# fleet, 20% at a time
ansible-playbook -i inventory.ini test_log_pipeline.yml
```

## Before the first run

- **Route `pipeline-probe` in NESS.** Every probe is real billable ingest and a
  real alert candidate. Give the APP-NAME a drop rule or a throwaway index
  first. 500 collectors × 15 probes is 7,500 events per sweep.
- **Check the UDP receive buffer.** TC-11 shortfalls on the 3164 path are
  usually `net.core.rmem_max`, not Fluent Bit. Watch `ss -lunm`.
- **Confirm which framing NESS expects.** If it wants RFC 6587 octet counting,
  set `probe_expect_octet_counting: true` — otherwise TC-15 fails on a frame
  that is actually correct.
- **Set `probe_sd_id`** to your own private enterprise number. `32473` is the
  RFC's example value and must not go to production.

## What this does not cover

- **TLS.** If Fluent Bit → Fluentd or Fluentd → NESS is TLS, the wire tap is
  plaintext on loopback and proves nothing about cert validation, chain, or
  revocation. Test that separately.
- **Semantic facility mapping.** TC-16 proves PRI decodes to what the probe
  declared. It does not prove your switches' `local4` is the facility NESS
  expects to see for switches — that's a policy decision, not a format one.
- **Sustained-rate behaviour.** 15 events per host proves the path works. It
  says nothing about buffer behaviour at 5,000 eps or during a Fluentd
  restart. Load testing is a different exercise.
- **Real device output.** TC-06 and TC-07 impersonate a switch and a clock
  faithfully at the byte level, but a real Cisco IOS `%SYS-5-CONFIG_I` frame
  has its own quirks. Pair this with a capture from one real device of each
  type, replayed through the same path.
- **Ordering.** Nothing here asserts that events arrive in the order sent.
