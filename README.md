# WaemoScan V3

WaemoScan is a high-performance Network Diagnostic and Protocol Auditing Framework built for controlled SOC validation, protocol-state evidence generation, and low-level network-path diagnostics.

Current validated status: **V3 through Sprint 3V — Firewall Validation Workflow**.

## Architecture

- **C# / .NET 10**: CLI, orchestration, profiles, policy, reporting, JSON artifacts.
- **Rust native layer**: raw socket lifecycle, packet utilities, native transport boundary, ABI contract.
- **Protocol diagnostics**: deterministic IPv4 fragmentation scenarios, gateway-routed L2 binding, live TX, response capture, correlation evidence.

## Current Feature Set

| Area | Status | Notes |
|---|---:|---|
| Scan / fingerprint | Complete | `scan`, `fp`, banner follow-up, JSON/export/report. |
| Full range scan | Complete | 1-65535 with open-only report projection. |
| External L2 gateway | Complete | `--profile external-l2-gateway`, gateway MAC as Ethernet destination. |
| Fragment catalog | Complete | `tcp8`, `tiny-first-fragment`, `tcp-options-split`. |
| Readiness gate | Complete | Verifies route, gateway MAC, source IP/MAC, offsets, CAP_NET_RAW. |
| Controlled live TX | Complete | Exact frames from readiness artifact; no mutation or range loop. |
| Fragment response probe | Complete | Classifies `SynAck`, `Rst`, `Icmp`, `Timeout`. |
| Firewall validation | Complete | Baseline TCP path vs fragmented TCP path in one artifact. |
| KVM Windows Server lab | Pending | Sprint 3W. |
| Company release pack | Pending | Sprint 3X. |

## Quick Start

```bash
waemoscan --help
waemoscan diagnose --help
```

Daily SOC flow:

```bash
waemoscan scan -t 192.168.33.105 -p 18000-18080 --out /tmp/mint-scan.json
waemoscan report /tmp/mint-scan.json

waemoscan fp -t 192.168.33.105 -p 18080 -B --out /tmp/mint-fp.json
waemoscan report /tmp/mint-fp.json
```

Full range scan:

```bash
waemoscan scan -t 192.168.33.105 -p 1-65535 --out /tmp/mint-full.json
waemoscan report /tmp/mint-full.json
```

## Build, Test, Package, Install

```bash
cargo test --manifest-path WaemoScan.Muscle/Cargo.toml

dotnet test WaemoScan.Brain/tests/WaemoScan.Tests/WaemoScan.Tests.csproj

make package-release
sudo make install-release

waemoscan --version
waemoscan --help
```

## Execution Profiles

| Profile | Purpose |
|---|---|
| `internal-l3` | Routed/private internal L3 path. |
| `internal-l2` | Same-segment internal L2 path. |
| `external-l3` | Public/hostname baseline L3 path. |
| `external-l2-gateway` | Gateway-routed L2 path; Ethernet destination is gateway MAC. |
| `lab` | Namespace/veth lab execution profile. |

## Fragment Scenarios

| Scenario | ScenarioId | Fragments | Offsets | Purpose |
|---|---|---:|---|---|
| `tcp8` | `tcp8_syn` | 2 | `0x2000`, `0x0001` | First fragment carries only first 8 bytes of TCP segment. |
| `tiny-first-fragment` | `tiny_first_fragment_syn` | 3 | `0x2000`, `0x2001`, `0x0002` | Tiny initial fragment layout. |
| `tcp-options-split` | `tcp_options_split_syn` | 4 | `0x2000`, `0x2001`, `0x2003`, `0x0005` | TCP options/control visibility split. |

## Protocol Diagnostic Workflows

### 1. Offline compose

```bash
waemoscan diagnose fragment \
  --scenario tcp8 \
  --out /tmp/tcp8-offline.json

waemoscan report /tmp/tcp8-offline.json
```

### 2. Gateway readiness

```bash
sudo waemoscan diagnose fragment \
  --scenario tcp8 \
  --profile external-l2-gateway \
  -i <IFINDEX> \
  -t scanme.nmap.org \
  -p 80 \
  --out /tmp/tcp8-ready.json

waemoscan report /tmp/tcp8-ready.json
```

### 3. Controlled live TX

```bash
sudo waemoscan diagnose fragment-tx \
  /tmp/tcp8-ready.json \
  -i <IFINDEX> \
  --inter-frame-delay-us 500 \
  --out /tmp/tcp8-live-tx.json

waemoscan report /tmp/tcp8-live-tx.json
```

### 4. Fragment response probe

```bash
sudo waemoscan diagnose fragment-probe \
  /tmp/tcp8-ready.json \
  -i <IFINDEX> \
  --inter-frame-delay-us 500 \
  --response-ms 3000 \
  --receive-timeout-ms 100 \
  --max-frames 128 \
  --out /tmp/tcp8-probe.json

waemoscan report /tmp/tcp8-probe.json
```

Response classifications:

| ResponseStatus | Meaning |
|---|---|
| `SynAck` | Target accepted/reassembled the fragmented SYN and responded. |
| `Rst` | The path reached a TCP decision point but the port/session was refused. |
| `Icmp` | ICMP path response observed. Inspect type/code. |
| `Timeout` | No correlated response in the configured response window. |
| `NotExecuted` | Dry-run only. |

### 5. Firewall validation

```bash
sudo waemoscan diagnose firewall-validate \
  --scenario tcp8 \
  --profile external-l2-gateway \
  -i <IFINDEX> \
  -t scanme.nmap.org \
  -p 80 \
  --response-ms 3000 \
  --out /tmp/firewall-validation.json

waemoscan report /tmp/firewall-validation.json
```

Interpretation codes:

| Code | Meaning |
|---|---|
| `BothReached` | Normal path and fragmented path both reached the target. |
| `FragmentedOnlyReached` | Fragmented path reached the target while normal path did not. Visibility-gap candidate. |
| `NormalOnlyReached` | Normal path reached the target while fragmented path did not. |
| `BothNoTargetResponse` | Neither path produced target-reach evidence. |
| `ValidatedDryRun` | Baseline measured; fragmented path validated without live TX. |
| `FragmentReadinessBlocked` | Fragmented TX was blocked by readiness. |

## Artifact Reference

| ArtifactKind | Command | Evidence |
|---|---|---|
| `waemoscan.protocol.fragment.diagnostic.v1` | `diagnose fragment` | Fragment plan, scenario, readiness, gateway binding. |
| `waemoscan.protocol.fragment.live-tx.v1` | `diagnose fragment-tx` | TX frames, bytes, timestamps, correlation keys. |
| `waemoscan.protocol.fragment.capture.v1` | `diagnose fragment-capture` | RX observed/matched/missing fragments. |
| `waemoscan.protocol.fragment.txrx.v1` | `diagnose fragment-txrx` | Armed TX/RX evidence in one artifact. |
| `waemoscan.protocol.fragment.probe.v1` | `diagnose fragment-probe` | Target-side response classification. |
| `waemoscan.protocol.firewall.validation.v1` | `diagnose firewall-validate` | Baseline vs fragmented path comparison. |

## Current Validation Snapshot

Validated workflows:

- Full range scan against Mint target: 1-65535 with open-only projection.
- `fragment-probe` installed execution: fragmented `tcp8` SYN produced target `SYN/ACK`.
- `firewall-validate` installed execution: `BothReached` against `scanme.nmap.org:80` with `PotentialVisibilityGap=false`.

## Roadmap Before Final Video

1. **Sprint 3W — KVM Windows Server Lab**
   - Windows Server target VM.
   - Controlled Windows Firewall policy.
   - Baseline scan vs fragmented firewall validation.

2. **Sprint 3X — Company Release Pack**
   - Final README/runbook.
   - SOC operator command sheet.
   - SOC manager demo script.
   - Known limits and acceptance matrix.

3. **Final video**
   - SOC user workflow.
   - Management-level evidence workflow.
   - Advanced fragmentation/firewall validation workflow.

## Known Operational Notes

- Raw socket paths require `CAP_NET_RAW` or `sudo`.
- Gateway-routed L2 requires a resolvable route device and gateway MAC.
- Wi-Fi adapters may not mirror outbound frames into the local RX queue; use observer-side capture or `fragment-probe` for target response evidence.
- `BothReached` is not a bypass claim; it means both paths reached the target.
- `FragmentedOnlyReached` is the primary visibility-gap candidate condition.

## Minimal Demo Sequence

```bash
waemoscan scan -t 192.168.33.105 -p 18000-18080 --out /tmp/demo-scan.json
waemoscan report /tmp/demo-scan.json

waemoscan fp -t 192.168.33.105 -p 18080 -B --out /tmp/demo-fp.json
waemoscan report /tmp/demo-fp.json

sudo waemoscan diagnose firewall-validate \
  --scenario tcp8 \
  --profile external-l2-gateway \
  -i <IFINDEX> \
  -t scanme.nmap.org \
  -p 80 \
  --response-ms 3000 \
  --out /tmp/demo-firewall-validation.json

waemoscan report /tmp/demo-firewall-validation.json
```# WaemoScanV3-README.md
