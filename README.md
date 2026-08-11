<div align="center">

<pre align="center" style="background-color: #0d1117; color: #58a6ff; font-family: monospace; overflow-x: auto; padding: 16px; border-radius: 6px;">
┌──────────────────────────────────────────────────────────────┐
│  B R O O D D E N  //  CONTAINER INCUBATION ENGINE            │
└──────────────────────────────────────────────────────────────┘
</pre>

<p align="center">
  <b>Declarative NixOS LXC Incubation Fabric & Orchestration Engine for Proxmox VE</b>
</p>

</div>

---

## Overview

**BroodDen** is an opinionated, API-driven Container Incubation Fabric. It bridges the gap between the absolute reproducibility of **NixOS** and the lightweight virtualization of **Proxmox LXC containers**.

It bypasses configuration drift, imperative shell scripts, and heavy state-management tools by treating infrastructure as an immutable biological ecosystem powered natively by **Nix Flakes**. 

Instead of configuring a container *after* it boots, BroodDen dynamically compiles a hyper-stripped NixOS rootfs tarball and injects it directly into the Proxmox REST API. The result is a fully configured, production-ready microservice.

##  Core Architecture

BroodDen operates on a **Two-Stage Genesis** architecture to maintain a zero-pollution policy on the Proxmox hypervisor. No software is installed on the Proxmox host OS.

1. **Stage 0 (Stream-Seeder):** A local script streams the `brood-queen` rootfs directly into the Proxmox REST API to instantiate **The Queen** (VMID 100).
2. **Stage 1 (Autonomous Gestation):** The Queen runs `BroodAPI`, listening for incubation requests. She dynamically wraps declarative Nix manifests (`genome.nix`) into Flakes, natively compiles them via `nix build`, and injects the resulting Drone containers into the hypervisor.

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             BROODDEN FABRIC TOPOLOGY                             │
└──────────────────────────────────────────────────────────────────────────────────┘

  [ LOCAL TERMINAL / CDE ]                   [ PROXMOX VE HYPERVISOR FABRIC ]
  ┌──────────────────────┐                   ┌──────────────────────────────────┐
  │ brood CLI            │ ────────────────> │ VMID 100: brood-queen            │
  │ (BroodDesign UI)     │  HTTP / REST      │  ├── BroodAPI (FastAPI Engine)   │
  └──────────────────────┘  Port 8000        │  └── Native Nix Flake Engine     │
                                             └────────────────┬─────────────────┘
                                                              │ Proxmox REST API
                                                              │ (Local Unix Socket / HTTPS)
                                                              ▼
                                             ┌──────────────────────────────────┐
                                             │ PROXMOX LXC ENGINE & STORAGE     │
                                             │  ├── Nursery: local:vztmpl       │
                                             │  ├── Drone 101: micro-auth       │
                                             │  ├── Drone 102: telemetry-sync   │
                                             │  └── Drone 103: gateway-proxy    │
                                             └──────────────────────────────────┘
```

## The Interface (`brood` CLI)

BroodDen features a high-density, utilitarian CLI built with Python `typer` and `rich`, adhering to the strict **BroodDesign** scale-proof terminal specification.

```text
$ brood spawn --name micro-auth --template genome.nix --ram 256

┌──────────────────────────────────────────────────────────────┐
│  B R O O D D E N  //  INCUBATION SEQUENCE                    │
└──────────────────────────────────────────────────────────────┘

INCUBATION PIPELINE
  ├── Validating genome manifest schema... [OK +8ms]
  ├── Compiling native Nix Flake rootfs... [OK +1.2s]
  ├── Streaming payload to Nursery (local:vztmpl)... [OK +310ms]
  ├── Allocating hypervisor VMID: 101... [OK +14ms]
  └── Booting instance kernel... [OK +410ms]

PROVISIONED TARGET SPECIFICATIONS
  ID:          101
  HOSTNAME:    micro-auth
  STATUS:      ● ONLINE
  ADDRESS:     192.168.0.33 (DHCP)
  RESOURCES:   256 MB RAM | 1 Core | 8 GB Storage

SYSTEM TELEMETRY
  NODE: proxmox | ELAPSED: 1.94s | API: BroodAPI v1.0.0
```

## Development Status & Execution Gates

This project is currently in **active development**, progressing through strict operational gates to ensure a stable Minimum Viable Product (MVP). 

- [ ] **GATE 1:** The Biological Proof (Local Flake Compilation PoC)
- [ ] **GATE 2:** The Stream-Seeder (Zero-pollution Queen bootstrapping)
- [ ] **GATE 3:** The Queen's Mind (BroodAPI FastAPI scaffolding)
- [ ] **GATE 4:** Autonomous Gestation (Dynamic Nix Flake generation inside BroodAPI)
- [ ] **GATE 5:** The User Interface (`brood` CLI implementation)

*For granular tracking of the development pipeline, see [`BroodDen-Execution-Tracker.md`](docs/BroodDen-Execution-Tracker.md).*

## The Brood Lexicon

To maintain operational precision across the codebase, the platform utilizes the following terminology:

*   **The Queen (`brood-queen`):** The persistent control-plane LXC (VMID 100) running `BroodAPI`.
*   **The Drone (`brood-drone`):** Ephemeral, stateless NixOS LXC micro-containers gestated by the Queen.
*   **The Genome (`genome.nix`):** The immutable, declarative Nix expression defining a Drone's OS and services.
*   **Gestation (`Spawn`):** The end-to-end pipeline of building a Nix Flake and injecting it via the Proxmox API.
*   **Cull (`Purge`):** The immediate, zero-residual destruction of a Drone instance.

---
*Architected for Proxmox VE. Built with Python (FastAPI), Native Nix Flakes, and Typer.*
