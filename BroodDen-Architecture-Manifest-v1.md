### ARTIFACT 1: `BroodDen-Architecture-Manifest-v1.md`

# BROODDEN PLATFORM SPECIFICATION v1.0
## Declarative NixOS LXC Incubation Fabric & Orchestration Architecture

---

### 1. EXECUTIVE SUMMARY & THE BROOD LEXICON

`BroodDen` is an opinionated, API-driven **NixOS Container Incubation Fabric** designed for Proxmox VE. It bypasses configuration drift, imperative shell scripts, and heavy state-management tools by treating infrastructure as an immutable biological ecosystem powered natively by **Nix Flakes**.

To maintain operational precision, the platform adopts the following taxonomy:

*   **The Fabric (`BroodDen`):** The overall infrastructure topology (Proxmox host + network mesh + API layers).
*   **The Hive (`proxmox`):** The physical hypervisor node hosting the ecosystem.
*   **The Queen (`brood-queen`):** VMID 100. The persistent control-plane LXC container. She hosts the `BroodAPI` and the native Nix Flake compilation engine.
*   **The Drone (`brood-drone`):** Ephemeral, stateless NixOS LXC micro-containers gestated by the Queen.
*   **The Genome (`genome.nix`):** The immutable, declarative Nix expression defining a Drone's OS, systemd units, network bounds, and application code.
*   **Gestation (`Spawn`):** The process where the Queen wraps a Genome in a Flake, compiles the tarball, and injects it into the hypervisor via API.
*   **The Nursery (`local:vztmpl`):** The hypervisor storage volume holding compiled rootfs tarballs (`.tar.xz`).
*   **Cull (`Purge`):** The immediate, zero-residual destruction of a Drone instance and its associated hypervisor state.

---

### 2. THE BROODDEN DOCTRINE: NATIVE FLAKE ARCHITECTURE

Following the deprecation of third-party tools like `nixos-generators`, BroodDen relies entirely on the modern, robust **Native Nix Flake** ecosystem.

**The Compilation Pipeline:**
1. A user submits a `genome.nix` to the Queen.
2. The Queen creates an isolated temporary directory and drops the `genome.nix` inside, alongside a dynamically generated, standardized `flake.nix` wrapper.
3. The Queen executes the native build target:
   `nix build .#nixosConfigurations.drone.config.system.build.tarball`
4. The Nix daemon natively outputs a hyper-stripped NixOS LXC rootfs tarball in the `./result/` symlink.
5. `BroodAPI` uploads this tarball directly to the Proxmox REST API (`POST /api2/json/nodes/{node}/storage/local/upload`) and awakens the Drone.

---

### 3. TWO-STAGE GENESIS (THE STREAM-SEEDER INCEPTION)

To control a Hive, it must be seeded with a Queen (VMID 100) without polluting the hypervisor OS or exposing PVE to the public WAN.

*   **STAGE 0 (The Stream-Seeder):** Executed from a local workstation/CDE via `seed.py`. The script fetches the pre-compiled `brood-queen.tar.xz` from GitHub Release Assets (or a local build) and streams it directly into the Proxmox REST API (`POST /storage/local/upload`). It then issues `POST /lxc` passing `ostype=unmanaged` and `features="nesting=1"` to instantiate VMID 100.
*   **STAGE 1 (Autonomous Gestation):** The Queen boots, initializes `BroodAPI` on port 8000 via systemd, and listens for the `brood` CLI. She takes over all future Drone compilation (`nix build`) and orchestration autonomously.
---

### 4. ARCHITECTURAL TOPOLOGY

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
                                                              │ (Local Unix Socket / 8006)
                                                              ▼
                                             ┌──────────────────────────────────┐
                                             │ PROXMOX LXC ENGINE & STORAGE     │
                                             │  ├── Nursery: local:vztmpl       │
                                             │  ├── Drone 101: micro-auth       │
                                             │  ├── Drone 102: telemetry-sync   │
                                             │  └── Drone 103: gateway-proxy    │
                                             └──────────────────────────────────┘
```
