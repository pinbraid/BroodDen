**Operating Principle:** Do not build the Seeding script until the Flake compiles. Do not build the CLI until the API works. Do not build the API until the Stream-Seeder works.

---

### GATE 1: THE BIOLOGICAL PROOF (Local Flake PoC)
**Goal:** Prove that we can generate a Proxmox-compatible LXC rootfs tarball using pure Native Nix Flakes on your local workstation/CDE, without any API or Python involvement.

- [ ] **Task 1.1:** Write a minimal `genome.nix` defining a baseline NixOS system (unprivileged container configuration, systemd init).
- [ ] **Task 1.2:** Write a `flake.nix` wrapper importing `nixpkgs` and defining the target:
    `nixosConfigurations.drone.config.system.build.tarball`
- [ ] **Task 1.3:** Run `nix build .#nixosConfigurations.drone.config.system.build.tarball` locally.
- [ ] **Task 1.4:** Inspect `./result/` and verify the generated `.tar.xz` rootfs payload.
- [ ] **Exit Criteria:** A valid `.tar.xz` rootfs image is successfully compiled locally using pure Nix Flakes.

---

### GATE 2: THE STREAM-SEEDER (`seed.py`)
**Goal:** Prove programmatic, zero-pollution inception of the Queen (VMID 100) onto a stock Proxmox hypervisor over the REST API without installing software on the PVE host.

- [ ] **Task 2.1:** Scaffold `seed/bootstrap.py` using Python `httpx` and `argparse`/`typer`.
- [ ] **Task 2.2:** Implement API token authentication (`PVEAPIToken`) and argument parsing for `--host`, `--node`, `--token-id`, `--token-secret`, `--file`, and `--repo`.
- [ ] **Task 2.3:** Implement local file streaming or GitHub Release asset fetching directly into Proxmox's Nursery endpoint:
    `POST /api2/json/nodes/{node}/storage/{template_storage}/upload` (`content=vztmpl`).
- [ ] **Task 2.4:** Implement Queen instantiation via Proxmox REST API:
    `POST /api2/json/nodes/{node}/lxc` passing mandatory parameters:
    1. `vmid=100`
    2. `hostname=brood-queen`
    3. `ostype=unmanaged`
    4. `features="nesting=1"`
    5. `unprivileged=1`
    6. `net0="name=veth0,bridge=vmbr0,ip=dhcp"`
- [ ] **Task 2.5:** Trigger `POST /api2/json/nodes/{node}/lxc/100/status/start`.
- [ ] **Exit Criteria:** Running `python seed/bootstrap.py` from your local terminal streams the Gate 1 tarball directly into Proxmox, spawns VMID 100, and boots the Queen without touching the Proxmox Web GUI or SSH.

---

### GATE 3: THE QUEEN'S MIND (`BroodAPI` MVP)
**Goal:** Build the FastAPI service that runs inside the Queen LXC (VMID 100) to orchestrate local Proxmox LXC lifecycles.

- [ ] **Task 3.1:** Scaffold `brood_api/main.py` (FastAPI Engine) and `brood_api/schemas.py` (Pydantic Genome contracts).
- [ ] **Task 3.2:** Implement `GET /v1/brood/telemetry` (Reads Proxmox hypervisor and active Drone LXC metrics).
- [ ] **Task 3.3:** Implement `DELETE /v1/brood/cull/{vmid}` with an explicit safety guard blocking requests where `vmid == 100`.
- [ ] **Task 3.4:** Implement template-based `POST /v1/brood/spawn` (Uses pre-existing templates in Nursery storage to verify the API-to-Proxmox spawn pipeline).
- [ ] **Exit Criteria:** You can send `curl` or HTTP requests to `BroodAPI` running inside VMID 100 to query telemetry, spawn basic LXCs, and cull Drones.

---

### GATE 4: AUTONOMOUS GESTATION (Dynamic Flake Engine)
**Goal:** Upgrade `BroodAPI` to dynamically wrap incoming `genome.nix` payloads in Flakes, run `nix build`, and deploy the resulting Drone tarball automatically.

- [ ] **Task 4.1:** Update `POST /v1/brood/spawn` to ingest raw `genome.nix` text strings or file payloads.
- [ ] **Task 4.2:** Write Python subprocess logic in `BroodAPI` to write the `genome.nix` to a temporary directory, generate a standard `flake.nix` wrapper, and execute:
    `nix build .#nixosConfigurations.drone.config.system.build.tarball`
- [ ] **Task 4.3:** Chain the output `./result/` tarball path into `POST /storage/{storage}/upload` and `POST /lxc` to awaken the new Drone automatically.
- [ ] **Exit Criteria:** Sending a raw `genome.nix` payload to `BroodAPI` triggers dynamic Flake compilation, Nursery upload, and immediate startup of a custom Drone on Proxmox in under 2 seconds.

---

### GATE 5: THE USER INTERFACE (`brood` CLI)
**Goal:** Wrap `BroodAPI` and the Stream-Seeder into the sleek, Weyland-Yutani inspired terminal tool using the scale-proof **BroodDesign** specification.

- [ ] **Task 5.1:** Install `typer` and `rich` in `brood_cli/`.
- [ ] **Task 5.2:** Implement `brood_cli/ui.py` containing the Mono-Frame "BroodDesign" render functions (`render_header`, `render_footer`, `render_table`).
- [ ] **Task 5.3:** Implement commands in `brood_cli/main.py`:
    * `brood spawn -f genome.nix`
    * `brood cull <vmid>`
    * `brood list`
    * `brood inspect <vmid>`
    * `brood status`
    * `brood templates`
    * `brood seed` (Integrates the Stream-Seeder logic)
---
**Exit Criteria:** Running `brood status` or `brood spawn` from your workstation renders a scale-proof Mono-Frame terminal readout. The platform is 100% complete, fully documented, and portfolio-ready.
