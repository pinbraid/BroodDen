
### PART I: THE BROODDEN DOCTRINE (WHY THIS IS UNIQUE)

Most homelab tools fall into two uninspired categories:
1. Generic Proxmox wrappers that just call `pct create` or wrap Proxmox GUI features in Bash/Python.
2. Heavy Terraform / Ansible scripts that take 10 minutes to execute, leave orphaned state files, and suffer from configuration drift.

**BroodDen's Uniqueness:**
* **The Declarative Genome:** You do not configure an LXC container *after* it is created. You define the entire operating system, systemd services, API code, and network policies in a single Nix expression (`genome.nix`).
* **Instant Immutable Incubation:** `BroodDen` compiles that Nix expression into a hyper-stripped NixOS LXC rootfs tarball (~30MB–50MB), uploads it via the Proxmox API (`POST /api2/json/nodes/{node}/storage/local/upload`), and awakens a running, fully-configured container in under **2 seconds**.
* **Zero Technical Debt (Ruthless Culling):** If a drone experiences data drift or package corruption, you do not SSH in and troubleshoot. You `cull` it and re-incubate a flawless clone from the genome.

---

### PART II: MULTI-HIVE TOPOLOGY & THE CHICKEN-AND-EGG PROBLEM

To manage multiple hypervisors, **every Proxmox host in your infrastructure becomes a "Hive."** To control a Hive, it must be seeded with a **Queen** (VMID 100). The Queen runs `BroodAPI` and handles the local Nix compilation and LXC orchestration. 

To solve the chicken-and-egg problem of how the Queen is born, we use a **Two-Stage Genesis Architecture**:

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            TWO-STAGE GENESIS ARCHITECTURE                        │
└──────────────────────────────────────────────────────────────────────────────────┘

  [ STAGE 0: SEEDING THE QUEEN ]                   [ STAGE 1: AUTONOMOUS HIVE ]
  Executed locally via MVP Script                  Executed on Proxmox Hypervisor
  ┌──────────────────────────────┐                 ┌──────────────────────────────┐
  │ Local `brood seed` CLI       │                 │ VMID 100: brood-queen        │
  │ Compiles queen.nix via Nix   │ ──────────────> │ Running BroodAPI Engine      │
  │ Hits Proxmox API to launch   │                 │ Listens for `brood` CLI      │
  │ VMID 100 across all Hives    │                 │ Incubates Drone Containers   │
  └──────────────────────────────┘                 └──────────────────────────────┘
```

#### The MVP Pathway (Friction Reduction)
We will not wait for a perfect Terraform module to get started. 
**MVP Execution:** We will write a fast Python/Bash bootstrap script (`brood seed`) that compiles the Queen's genome locally, pushes the tarball to your target Proxmox host, and spins up VMID 100. Once the Queen is alive, we immediately shift focus to building the `BroodAPI` so she can start incubating drones. Terraform polish can come later.

---

### PART III: THE INCUBATION LIFECYCLE (START TO FINISH)

Here is the exact operational telemetry of how a new microservice is born within BroodDen:

```text
1. SEQUENCE ──> You write a custom Python API and define its runtime in a `genome.nix` file.
2. SPAWN    ──> You run `brood spawn -f genome.nix --hive proxmox-slab` from your local terminal.
3. TRANSMIT ──> The payload hits `BroodAPI` inside the target Hive's Queen (VMID 100).
4. GESTATE  ──> The Queen utilizes `nixos-generators` to compile a minimal NixOS LXC rootfs tarball.
5. UPLOAD   ──> `BroodAPI` POSTs the tarball to the local Proxmox API (`/storage/local/upload`).
6. AWAKEN   ──> `BroodAPI` POSTs to `/nodes/{node}/lxc` with `ostype=unmanaged` & `features="nesting=1"`.
7. RUN      ──> The new drone container boots instantly, executing your code as a native systemd service.
```

---

### PART IV: RAPID TESTING (LIFECYCLE CONTROL)

To give you complete freedom to stand up and tear down `BroodDen` Hives for testing without cluttering your infrastructure, the CLI includes two core Genesis commands:

#### `brood seed --hive <ip>`
* **Action:** Compiles `queen.nix` locally, uploads the template to the target Proxmox host, creates VMID 100, configures static IP, and starts the `BroodAPI` service.
* **Result:** A new Hive is brought online and ready to accept spawn commands in ~60 seconds.

#### `brood cull-queen --hive <ip> --force`
* **Action:** Sends a fatal purge command to VMID 100, unlinks the storage allocation, and deletes the `queen` template tarball from Proxmox storage.
* **Result:** Total platform wipe for that specific host with zero residual state.

---

### PART V: EVOLUTION PIPELINES (GITHUB ACTIONS & TERRAFORM)

While the MVP relies on manual seeding, the production state utilizes industry-standard pipelines:

1. **Terraform (The Substrate):** Eventually, the `brood seed` command will wrap a lightweight Terraform module (`bpg/proxmox`) to guarantee that Queen deployments across 3+ Proxmox hosts are strictly idempotent and tracked in state.
2. **GitHub Actions (The Mutation Gate):**
   * Whenever you push code to `github.com/pinbraid/BroodDen`, GitHub Actions runs `pytest` on `BroodAPI` and validates that `queen.nix` evaluates without Nix syntax errors.
   * On release tags, GHA automatically compiles the `queen.nix` rootfs tarball and attaches it as a GitHub Release Asset. This means `brood seed` can simply download the pre-compiled Queen instantly instead of building her locally.

---

### PART VI: REPOSITORY ARCHITECTURE (`pinbraid/BroodDen`)

```text
BroodDen/
├── .github/
│   └── workflows/
│       ├── test.yml            # Pytest & Nix evaluation check
│       └── release.yml         # Pre-compiles Queen LXC tarball release
├── seed/                       # Stage 0 Bootstrap Engine (MVP)
│   ├── queen.nix               # Declarative NixOS genome for the Queen
│   └── bootstrap.py            # MVP script to seed a Queen on a new Proxmox host
├── brood_api/                  # Stage 1 Queen Engine (FastAPI)
│   ├── main.py                 # FastAPI application (The Hive Mind)
│   ├── schemas.py              # Pydantic incubation schemas
│   └── proxmox.py              # Local Proxmox REST API client
├── brood_cli/                  # Local Terminal CLI (BroodDesign UI)
│   ├── main.py                 # Typer CLI entrypoint
│   └── ui.py                   # Mono-Frame BroodDesign renderer
├── genomes/                    # Example Drone NixOS Profiles
│   ├── python-fastapi.nix
│   └── postgres-vector.nix
├── pyproject.toml
└── README.md
```
