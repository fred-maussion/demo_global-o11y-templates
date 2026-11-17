# OVHcloud Network Operations

Elastic Observability scenario for a mid-size European enterprise operating across 3 OVHcloud datacenters (Gravelines, Strasbourg, Beauharnois). Generates realistic synthetic telemetry — metrics, logs, and traces — for OVH-native network infrastructure, with 12 fault injection channels and an AI agent for incident response demos.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Services](#services)
- [Fault Channels](#fault-channels)
- [Executive KPIs](#executive-kpis)
- [AI Agent](#ai-agent)
- [Demo Scripts](#demo-scripts)
- [Log Namespaces](#log-namespaces)
- [Directory Structure](#directory-structure)

## Overview

| Property | Value |
|----------|-------|
| Scenario ID | `ovh` |
| Namespace | `ovhnet` |
| Schema version | 1.2 |
| Services | 6 |
| Channels | 12 (8 HITL + 4 Auto-Remediate) |
| Datacenters | GRA11 (Gravelines), SBG5 (Strasbourg), BHS5 (Beauharnois) |
| Currency | EUR |
| KPI emitter | ovh-load-balancer |

## Architecture

```
                        ┌─────────────────────────────────────────────┐
                        │              OVHcloud Network               │
                        │                                             │
  GRA11 (Gravelines)    │    SBG5 (Strasbourg)    BHS5 (Beauharnois) │
 ┌───────────────────┐  │  ┌──────────────────┐  ┌─────────────────┐ │
 │ vrack-network-mgr │──┼──│  anti-ddos-vac   │  │ ovhcloud-connect│ │
 │ ovh-load-balancer │  │  │  gateway-nat     │  │ managed-k8s     │ │
 └───────────────────┘  │  └──────────────────┘  └─────────────────┘ │
                        └─────────────────────────────────────────────┘
```

**Service call topology:**

- vrack-network-manager → ovh-load-balancer, gateway-nat, anti-ddos-vac
- ovh-load-balancer → vrack-network-manager, gateway-nat
- anti-ddos-vac → vrack-network-manager, ovh-load-balancer
- ovhcloud-connect → vrack-network-manager, managed-kubernetes
- managed-kubernetes → ovh-load-balancer, vrack-network-manager

## Services

| Service | Subsystem | Language | Region | Description |
|---------|-----------|----------|--------|-------------|
| vrack-network-manager | core_network | Go | GRA11 | vRack private networking — VLAN management, subnet IP monitoring, cross-DC latency checks |
| ovh-load-balancer | traffic_management | Java | GRA11 | HAProxy-based LB — request distribution, backend health, SSL termination, connection stats |
| anti-ddos-vac | security | Rust | SBG5 | Anti-DDoS VAC scrubbing — packet filtering, attack volume, false positive detection |
| gateway-nat | network_access | Python | SBG5 | Gateway NAT (SNAT) — port allocation, conntrack, outbound routing |
| ovhcloud-connect | connectivity | Go | BHS5 | Dedicated/provider interconnect — BGP peering, circuit monitoring, optic Rx levels |
| managed-kubernetes | orchestration | Go | BHS5 | Managed Kubernetes Service (MKS) — node pools, pod lifecycle, HPA scaling |

Each service emits:
- **Metrics** with `if_active` degradation during fault injection
- **INFO logs** for operational telemetry (the baseline noise)
- **WARN logs** for threshold warnings (MTU mismatch, queue depth, evictions)
- **ERROR logs** for unrelated faults (DNS failures, CNI errors, LACP timeouts)

The vrack-network-manager additionally emits high-volume `VRACK-VLAN-SEGMENTATION` ERROR logs that form the dominant category during CH-01 fault injection.

## Fault Channels

### HITL (Human-in-the-Loop) — Channels 01-08

These require manual SRE approval before remediation executes.

| # | Channel | Error Code | Affected Services | Cascade |
|---|---------|------------|-------------------|---------|
| 01 | vRack Cross-DC VLAN Segmentation | `VRACK-VLAN-SEGMENTATION` | vrack-network-manager, ovh-load-balancer | managed-kubernetes |
| 02 | vRack Subnet IP Exhaustion | `VRACK-SUBNET-EXHAUSTION` | vrack-network-manager, gateway-nat | ovh-load-balancer |
| 03 | Anti-DDoS VAC False Positive | `VAC-FALSE-POSITIVE-FILTERING` | anti-ddos-vac, ovh-load-balancer | ovhcloud-connect |
| 04 | OVHcloud Connect Circuit Degradation | `CONNECT-CIRCUIT-DEGRADATION` | ovhcloud-connect, vrack-network-manager | managed-kubernetes, gateway-nat |
| 05 | LB Backend Drain Storm | `LB-BACKEND-DRAIN-STORM` | ovh-load-balancer, managed-kubernetes | gateway-nat |
| 06 | Gateway NAT SNAT Exhaustion | `GW-SNAT-EXHAUSTION` | gateway-nat, vrack-network-manager | ovh-load-balancer |
| 07 | Floating IP Reassignment Conflict | `VRACK-FLOATINGIP-CONFLICT` | vrack-network-manager, ovh-load-balancer | managed-kubernetes |
| 08 | OVHcloud Connect BGP Peer Flap | `CONNECT-BGP-FLAP` | ovhcloud-connect, vrack-network-manager | gateway-nat |

### Auto-Remediate — Channels 09-12

These self-heal without human intervention.

| # | Channel | Error Code | Affected Services | Cascade |
|---|---------|------------|-------------------|---------|
| 09 | MKS Node NotReady Cascade | `MKS-NODE-NOTREADY` | managed-kubernetes, ovh-load-balancer | vrack-network-manager |
| 10 | LB SSL Certificate Renewal | `LB-SSL-CERT-RENEWAL` | ovh-load-balancer, vrack-network-manager | anti-ddos-vac |
| 11 | MKS Pod OOMKill Recovery | `MKS-POD-OOMKILL` | managed-kubernetes, ovh-load-balancer | gateway-nat |
| 12 | LB Health Check Flap | `LB-HEALTHCHECK-FLAP` | ovh-load-balancer, managed-kubernetes | vrack-network-manager |

Each channel includes:
- `investigation_notes` with real OVH CLI / API commands
- `error_message` and `stack_trace` templates with parameterized placeholders
- `fault_params` with random generators for realistic variation
- `rca_clues` per affected service
- `business_impact` referencing executive KPIs

## Executive KPIs

22 KPIs across 4 sections, scaled for a mid-size European enterprise (EUR):

| Section | KPIs |
|---------|------|
| Traffic | LB requests/min, latency p50, backend health %, 5xx error rate, throughput Gbps, connection rate |
| Network & Security | vRack cross-DC latency, DDoS mitigation events/min, Gateway SNAT utilization, Floating IP failover, private network packet loss, Connect bandwidth |
| SLO & Reliability | Uptime %, API success rate %, SLA compliance %, MTTD, MTTR, availability index |
| Commercial | Cloud spend EUR/min, committed resources utilization %, cost per M requests EUR, resource efficiency %, carbon intensity, SLA breach count |

## AI Agent

**OVHcloud Network Analyst** — a configured Elastic AI Assistant persona with:
- Knowledge of all 6 OVH services and their interdependencies
- Awareness of all 12 error codes and their root causes
- Ability to reference `ovh-cli` and `openstack` CLI commands
- `body.text` search guidance for unstructured log investigation

## Demo Scripts

| Language | File | Format |
|----------|------|--------|
| English | [OVH Demo Script - EN.html](script/OVH%20Demo%20Script%20-%20EN.html) | Presenter walkthrough |
| French | [OVH Demo Script - FR.html](script/OVH%20Demo%20Script%20-%20FR.html) | Presenter walkthrough |

**Recommended demo flow (~12 min):**
1. Inject CH-01 (VRACK-VLAN-SEGMENTATION, HITL) + CH-12 (LB-HEALTHCHECK-FLAP, Auto)
2. Application Cockpit — show SLO burning and KPI impact
3. Discover — ES|QL on `logs.otel.ovhnet` namespace, ML categorization
4. AI Assistant — explain log entry, start conversation
5. Agent creates tracking dashboard and observability case with runbook
6. Reveal auto-remediation of CH-12, approve HITL remediation for CH-01
7. Watch resolution on cockpit

## Log Namespaces

To scope queries to this scenario only:

```
FROM logs.otel.ovhnet, logs.otel.ovhnet.*, logs.ecs.ovhnet, logs.ecs.ovhnet.*
```

Additional filters: `cloud.provider: "ovh"` or `service.namespace: "ovhnet"`.

## Directory Structure

```
ovh/
  README.md
  scenario.yaml              # Master config: hosts, clusters, agent, KPIs
  services/
    vrack-network-manager.yaml
    ovh-load-balancer.yaml
    anti-ddos-vac.yaml
    gateway-nat.yaml
    ovhcloud-connect.yaml
    managed-kubernetes.yaml
  channels/
    01-vrack-vlan-segmentation.yaml
    02-vrack-subnet-exhaustion.yaml
    03-vac-false-positive.yaml
    04-connect-circuit-degradation.yaml
    05-lb-backend-drain-storm.yaml
    06-gw-snat-exhaustion.yaml
    07-floatingip-conflict.yaml
    08-connect-bgp-flap.yaml
    09-mks-node-notready.yaml
    10-lb-ssl-cert-renewal.yaml
    11-mks-pod-oomkill.yaml
    12-lb-healthcheck-flap.yaml
  script/
    OVH Demo Script - EN.html
    OVH Demo Script - EN.md
    OVH Demo Script - FR.html
    OVH Demo Script - FR.md
```
