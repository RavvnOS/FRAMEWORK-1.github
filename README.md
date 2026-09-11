# FRAMEWORK-1.github
Changes and Efficiency 
# RavvnOS Framework v2 — Cross-Platform Core (Design Doc)

> Scope: this document covers only the **new framework redesign discussion** — not the original v1 Bash core. It defines the architecture for a cross-platform (Linux/Arch + Windows) core, its tech stack, known risks with mitigations, and the build roadmap.

---

## 1. Why a New Framework

The original core was Bash-based and Linux-only — it does not run natively on Windows. Since cross-platform compatibility (Arch Linux **and** Windows) is now a hard requirement, the core needs a **redesign and rewrite**, not just an extension of the existing scripts.

### Requirements

- Full compatibility and feasibility on **Arch Linux** and **Windows**.
- Architecture that makes it easy to add new features later without destabilizing the whole system.
- Threshold-based monitoring/detection for RAM, network, other core-level conditions, and malicious/suspicious activity.
- Automatic **PDF alert report generation + email delivery** when a serious threshold or core-level problem is detected.
- A technology stack with strong performance, minimal delay, and efficient execution.
- Rename pending: current working name **"Core Insight"** will be replaced with a final name later.

---

## 2. High-Level Architecture

```text
                    ┌─────────────────────────┐
                    │      CLI / Interface     │
                    │  (ravvn commands, later  │
                    │   GUI/dashboard too)     │
                    └───────────┬─────────────┘
                                │ (local API / IPC)
                    ┌───────────▼─────────────┐
                    │       Core Engine         │
                    │  (single background       │
                    │   daemon process)          │
                    └───────────┬─────────────┘
        ┌───────────────────────┼───────────────────────┐
        ↓                       ↓                        ↓
┌────────────────┐     ┌───────────────────┐     ┌──────────────────┐
│  Metrics Core    │     │ Detection Engine   │     │ Config + Logging │
│ (OS-abstracted)  │     │ (thresholds +      │     │   (cross-OS)     │
│                  │     │  suspicious         │     └──────────────────┘
│  Linux backend   │     │  activity rules)    │
│  Windows backend │     └─────────┬───────────┘
└────────┬─────────┘               │
         │                         ↓
         │               ┌───────────────────┐
         │               │   Event Engine     │
         │               │ (state machine:    │
         │               │ NORMAL→WARNING→    │
         │               │ CRITICAL→RESOLVED) │
         │               └─────────┬──────────┘
         │                         ↓
         │               ┌───────────────────┐
         │               │  Report Engine     │
         │               │  (PDF generation)  │
         │               └─────────┬──────────┘
         │                         ↓
         │               ┌───────────────────┐
         │               │ Notification       │
         │               │ Engine (Email)     │
         │               └───────────────────┘
         ↓
   Health / Monitor / Doctor
   (all read from Metrics Core —
    no independent data collection)
```

---

## 3. Metrics Core — the Foundation

The single most important structural fix: **one shared metrics layer**, instead of every module (Health, Monitor, Events, Doctor) independently collecting the same data.

```text
Metrics Core
     │
     ├── Linux backend    (/proc, /sys, systemctl)
     └── Windows backend  (WMI, Performance Counters)

Everything above (Health, Detection, Events, Doctor) calls a common interface:
   get_cpu(), get_memory(), get_disk(), get_network(), get_services()
```

**Behavior:**
- The latest sampled values are cached briefly (e.g. 1–2 seconds).
- If a value is requested within that window, the cached value is returned — no new system call.
- If the cache has expired, a fresh sample is taken and cached again.
- This is a **cache, not a log**: it holds only the latest snapshot, not history.

**Why this matters:** without it, Health/Monitor/Events/Doctor can each compute CPU independently within the same second — wasting resources and risking inconsistent numbers between modules.

---

## 4. Event Engine — State-Based Alerting (solves alert spam)

Each monitored condition (e.g. `CPU_HIGH`, `RAM_CRITICAL`) tracks a single **active event** with an explicit state:

```text
NORMAL ──(threshold cross)──▶ WARNING ──(escalate)──▶ CRITICAL
                                                          │
                                                   alert sent (once)
                                                          │
NORMAL ◀──────────── RESOLVED ◀──────────────────────────┘
                (alert sent again — resolution notice)
```

**Rule:** a new PDF + email alert is generated **only** on a state transition (entering WARNING/CRITICAL, or reaching RESOLVED) — never on every polling cycle while a condition remains unchanged.

---

## 5. Technology Stack

| Option | Verdict |
|---|---|
| Bash | Linux-only; does not run natively on Windows. Rejected. |
| Python | Cross-platform but heavier runtime overhead, slower startup — not ideal for a background daemon requiring minimal delay. |
| C/C++ | Fastest, but slower to develop and higher risk of memory-safety bugs; harder to maintain across two OS backends. |
| **Go** ✅ | Compiles to a single native binary for both Linux and Windows, low memory footprint, strong built-in concurrency, no runtime dependency, fast startup. |
| Rust | Even more efficient/safe than Go, but steeper learning curve and slower initial development velocity. |

**Decision: Go** — best balance of cross-platform compilation, performance, and development speed. Comparable to tools like Prometheus's `node_exporter` or Netdata, which solve the same class of problem.

---

## 6. Known Flaws / Risks (identified in advance, with mitigations)

| Flaw | Mitigation (built into the framework) |
|---|---|
| **Scope creep** — "malicious activity detection" can balloon into a full SIEM/EDR product | Split into two explicit parts (see §7). Only Part 1 is being built first. |
| **No code reuse from v1** — this is a rewrite, not a port | Explicitly planned and time-boxed as a rewrite in Go against the new Metrics Core interface. |
| **Cross-OS metric mismatch** — Linux (`/proc`) and Windows (WMI) calculate metrics differently | OS abstraction layer inside Metrics Core normalizes both backends behind one common interface. |
| **Alert spam** — naive threshold checks would fire an alert every poll cycle | Event Engine state machine (§4) ensures alerts fire only on state transitions. |
| **Security/privilege risk** — network + activity monitoring needs elevated permissions on both OS, making the tool itself an attack surface | Signed binaries and a safe update mechanism are part of the architecture, not an afterthought. |
| **Real-time detection needs a persistent process** — spawning a process per check doesn't scale | A single persistent Go daemon with concurrent goroutines handles metrics collection and detection instead. |

---

## 7. Scope Split: Part 1 vs Part 2

**Part 1 — Core Monitoring / Detection / Alerting (building now)**
Threshold-based checks for CPU, RAM, Disk, Network, and Services; state-based event tracking; PDF report generation; email delivery on real state changes. Well-defined and achievable.

**Part 2 — Suspicious Activity Indicators (later, on top of Part 1)**
Full malware/threat detection (signatures, threat-intel feeds, behavioral ML) is out of scope — that's the domain of dedicated EDR/AV products built by full security teams. Part 2 is scoped down to **indicator-based detection** using data already available from Metrics Core and the Detection Engine: unusual outbound connections, unexpected high resource use by unrecognized processes, suspicious login attempts, unauthorized service changes. No external threat database required.

---

## 8. Roadmap

```text
Phase 1 — Metrics Core (Go, OS-abstracted, Linux + Windows backends)
Phase 2 — Event Engine (state machine, deduplication)
Phase 3 — Doctor (root-cause correlation using Health + Events + Metrics)
Phase 4 — Monitor (adaptive refresh, trend indicators)
Phase 5 — CLI (unified status overview, consistent output)
Phase 6 — Report Engine (PDF generation) + Notification Engine (Email)
Phase 7 — Detection Engine: Part 1 (threshold-based) → Part 2 (suspicious activity indicators)
```

Explicitly deferred until justified: long-term historical storage/database, persistent daemons beyond what's architecturally necessary, and any "smart"/AI-style explanation features ahead of having real correlation data to explain.

---

## 9. Status

- **Framework name:** pending (currently "Core Insight" — to be renamed)
- **Platform targets:** Arch Linux + Windows
- **Stack:** Go (daemon + CLI), with OS-specific backends for metrics collection
- **Immediate focus:** Part 1 (Metrics Core → Event Engine → Report/Notification)
