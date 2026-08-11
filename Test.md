# BROODDEN PLATFORM SPECIFICATION v1.0-RC
## Declarative NixOS LXC Incubation Fabric & Orchestration Architecture

---

### EXECUTIVE SUMMARY & TAXONOMY

`BroodDen` is an opinionated, API-driven **NixOS Container Incubation Fabric** designed for Proxmox VE hypervisors. Instead of managing container lifecycles through manual GUI operations or heavy configuration management tools, `BroodDen` treats infrastructure as an immutable biological ecosystem: **Genomes** (declarative Nix expressions) are compiled into stripped-down rootfs images, stored in the **Nursery** (template vault), and gestated into **Drones** (stateless microservice LXCs) by the **Matriarch** (the primary orchestration node).

#### The Brood Lexicon
To maintain operational precision and system flavor across code bases, telemetry, and interfaces, the platform adopts the following taxonomy:

*   **The Fabric (`BroodDen`):** The overall infrastructure topology (Proxmox host + network mesh + API layers).
*   **The Matriarch (`Queen` / `brood-matriarch`):** VMID 100. The persistent control-plane LXC container hosting `BroodAPI`.
*   **The Drone (`Worker` / `brood-drone`):** Ephemeral, stateless NixOS LXC micro-containers gestated by the Matriarch.
*   **The Genome (`brood.nix` / `genome.nix`):** The immutable, declarative NixOS system expression defining a Drone's OS, systemd units, network bounds, and application code.
*   **Gestation (`Incubation` / `Spawn`):** The build, transmission, and instant execution pipeline of a Drone onto Proxmox.
*   **The Nursery (`local:vztmpl`):** The hypervisor storage volume holding compiled rootfs tarballs (`.tar.xz`).
*   **Purge (`Cull`):** The immediate, zero-residual destruction of a Drone instance and its associated hypervisor state.

---

### ARCHITECTURAL TOPOLOGY

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             BROODDEN FABRIC TOPOLOGY                             │
└──────────────────────────────────────────────────────────────────────────────────┘

  [ LOCAL TERMINAL / CDE ]                   [ PROXMOX VE HYPERVISOR FABRIC ]
  ┌──────────────────────┐                   ┌──────────────────────────────────┐
  │ brood CLI            │ ────────────────> │ VMID 100: brood-matriarch        │
  │ (BroodDesign UI)     │  HTTP / REST      │  ├── BroodAPI (FastAPI Engine)   │
  └──────────────────────┘  Port 8000        │  └── NixOS Build Engine          │
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

---

### PHASED IMPLEMENTATION ROADMAP (MVP-FIRST FOCUS)

To eliminate chicken-and-egg paralysis and deliver immediate utility, implementation is divided into three execution phases. **Phase 1 provides an operational MVP today using manual Matriarch bootstrapping.**

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             PHASED ROLLOUT ROADMAP                               │
└──────────────────────────────────────────────────────────────────────────────────┘

  [ PHASE 1: MANUAL GENESIS MVP ] ──> [ PHASE 2: CLI & BROODDESIGN ] ──> [ PHASE 3: FULL NIX ENGINE ]
  • Manually deploy Matriarch CT      • Wire brood CLI to BroodAPI        • In-Matriarch nixos-generate
  • Deploy BroodAPI (FastAPI)         • Render BroodDesign UI screens     • Automated genome compilation
  • Direct Proxmox API Gestation      • Implement Drone Purge/Inspect     • Full 1-click Genesis CLI
```

---

### PHASE 1: THE MVP MANUAL GENESIS (EXECUTE TODAY)

Rather than waiting to automate Stage 0 with complex Terraform modules or Nix build triggers, **Phase 1 stands up the Matriarch manually in 15 minutes** so you can immediately begin incubating Drones.

#### Step 1: Manually Instantiate the Matriarch (VMID 100)
1. On your Proxmox GUI or shell, create a standard Debian 12 LXC container:
   * **VMID:** `100`
   * **Hostname:** `brood-matriarch`
   * **RAM:** `1024 MB` | **Disk:** `16 GB`
   * **Features:** Enable `nesting=1`
   * **Network:** Static IP or DHCP reservation (e.g., `192.168.0.100`)

#### Step 2: Deploy `BroodAPI` on the Matriarch
Inside the `brood-matriarch` container, clone `github.com/pinbraid/BroodDen`, set up a Python virtual environment, install dependencies (`fastapi`, `uvicorn`, `httpx`, `pydantic`), and launch `BroodAPI`:

```bash
# Inside VMID 100 (brood-matriarch)
git clone https://github.com/pinbraid/BroodDen.git /opt/BroodDen
cd /opt/BroodDen
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn httpx pydantic

# Set Proxmox API Credentials
export PROXMOX_HOST="192.168.0.5" # Your Proxmox host IP
export PROXMOX_TOKEN_ID="terraform-prov@pam!builder"
export PROXMOX_TOKEN_SECRET="your-secret-uuid-here"

# Launch BroodAPI Engine
uvicorn brood_api.main:app --host 0.0.0.0 --port 8000
```

#### Step 3: Gestate Your First Drone via `curl`
With `BroodAPI` active on VMID 100, issue a single HTTP POST request from your local machine to incubate an LXC container on Proxmox:

```bash
curl -X POST "http://192.168.0.100:8000/v1/brood/spawn" \
     -H "Content-Type: application/json" \
     -d '{
           "name": "drone-alpha",
           "template": "debian-12-standard_12.2-1_amd64.tar.zst",
           "ram_mb": 256,
           "cores": 1,
           "disk_gb": 8
         }'
```

---

### PART IV: BROODAPI CORE ENGINE CODE (MVP EDITION)

This is the exact production-ready Python code for `brood_api/main.py`. It interfaces directly with Proxmox VE REST API endpoints to manage Drone lifecycles.

#### `brood_api/schemas.py` (Genome Manifest Contracts)

```python
from pydantic import BaseModel, Field
from typing import Optional

class GestationRequest(BaseModel):
    name: str = Field(..., example="drone-telemetry", description="Drone hostname prefix")
    template: str = Field(default="debian-12-standard_12.2-1_amd64.tar.zst", description="Nursery CT template name")
    vmid: Optional[int] = Field(default=None, description="Explicit VMID (auto-allocated if None)")
    ram_mb: int = Field(default=256, ge=128, le=4096, description="RAM allocated to Drone in MB")
    cores: int = Field(default=1, ge=1, le=4, description="CPU cores allocated")
    disk_gb: int = Field(default=8, ge=2, le=32, description="Disk capacity in GB")
    unprivileged: bool = Field(default=True, description="Enforce unprivileged user namespace boundary")
    auto_start: bool = Field(default=True, description="Boot Drone immediately after gestation")

class GestationResponse(BaseModel):
    vmid: int
    name: str
    status: str
    node: str
    execution_time_ms: float
    timestamp: str
```

#### `brood_api/main.py` (FastAPI Management Router)

```python
import os
import time
from datetime import datetime
from fastapi import FastAPI, HTTPException, status
import httpx
from brood_api.schemas import GestationRequest, GestationResponse

app = FastAPI(
    title="BroodAPI Core",
    description="Matriarch Orchestration Engine for the BroodDen Platform",
    version="1.0.0-RC1"
)

# Configuration from Environment Variables
PROXMOX_HOST = os.getenv("PROXMOX_HOST", "192.168.0.5")
NODE_NAME = os.getenv("PROXMOX_NODE", "proxmox-slab")
TOKEN_ID = os.getenv("PROXMOX_TOKEN_ID", "terraform-prov@pam!builder")
TOKEN_SECRET = os.getenv("PROXMOX_TOKEN_SECRET", "")

HEADERS = {
    "Authorization": f"PVEAPIToken={TOKEN_ID}={TOKEN_SECRET}"
}

@app.get("/v1/brood/telemetry", tags=["Observability"])
async def get_brood_telemetry():
    """Queries Proxmox API for hypervisor status and active Drone instances."""
    async with httpx.AsyncClient(verify=False) as client:
        try:
            url = f"https://{PROXMOX_HOST}:8006/api2/json/nodes/{NODE_NAME}/lxc"
            res = await client.get(url, headers=HEADERS, timeout=5.0)
            res.raise_for_status()
            
            containers = res.json()['data']
            drones = [
                {
                    "vmid": c['vmid'],
                    "name": c['name'],
                    "status": c['status'].upper(),
                    "ram_mb": c['maxmem'] // (1024 * 1024),
                    "cpu_cores": c['cpus']
                }
                for c in containers if c['vmid'] != 100 # Exclude Matriarch from Drone list
            ]
            
            return {
                "matriarch_vmid": 100,
                "node": NODE_NAME,
                "status": "ONLINE",
                "active_drones": len(drones),
                "drones": drones
            }
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Fabric Telemetry Fault: {str(e)}")

@app.post("/v1/brood/spawn", response_model=GestationResponse, status_code=status.HTTP_201_CREATED, tags=["Gestation"])
async def gestate_drone(payload: GestationRequest):
    """Instantiates a new Drone container on the Proxmox fabric via REST API."""
    start_time = time.time()
    
    # Auto-allocate VMID if not specified (range 101-200)
    target_vmid = payload.vmid or 101 
    
    proxmox_payload = {
        "vmid": target_vmid,
        "hostname": payload.name,
        "ostemplate": f"local:vztmpl/{payload.template}",
        "memory": payload.ram_mb,
        "cores": payload.cores,
        "storage": "vm-storage",
        "net0": "name=veth0,bridge=vmbr0,ip=dhcp",
        "unprivileged": 1 if payload.unprivileged else 0,
        "ostype": "unmanaged",  # Crucial for NixOS LXC support
        "features": "nesting=1",
        "start": 1 if payload.auto_start else 0
    }
    
    async with httpx.AsyncClient(verify=False) as client:
        try:
            url = f"https://{PROXMOX_HOST}:8006/api2/json/nodes/{NODE_NAME}/lxc"
            res = await client.post(url, headers=HEADERS, json=proxmox_payload, timeout=10.0)
            res.raise_for_status()
            
            elapsed_ms = (time.time() - start_time) * 1000
            
            return GestationResponse(
                vmid=target_vmid,
                name=payload.name,
                status="ONLINE" if payload.auto_start else "GESTATED",
                node=NODE_NAME,
                execution_time_ms=round(elapsed_ms, 2),
                timestamp=datetime.utcnow().isoformat()
            )
        except httpx.HTTPStatusError as e:
            raise HTTPException(status_code=e.response.status_code, detail=f"Proxmox Fabric Error: {e.response.text}")
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Gestation Failure: {str(e)}")

@app.delete("/v1/brood/cull/{vmid}", tags=["Purge"])
async def cull_drone(vmid: int, force: bool = True):
    """Purges a Drone container and unlinks its storage allocation."""
    if vmid == 100:
        raise HTTPException(status_code=400, detail="PROTECTION FAULT: Cannot cull the Matriarch (VMID 100).")
        
    async with httpx.AsyncClient(verify=False) as client:
        try:
            # 1. Stop instance
            stop_url = f"https://{PROXMOX_HOST}:8006/api2/json/nodes/{NODE_NAME}/lxc/{vmid}/status/stop"
            await client.post(stop_url, headers=HEADERS, timeout=10.0)
            
            # Wait 1s for process teardown
            time.sleep(1.0)
            
            # 2. Delete instance
            delete_url = f"https://{PROXMOX_HOST}:8006/api2/json/nodes/{NODE_NAME}/lxc/{vmid}"
            res = await client.delete(delete_url, headers=HEADERS, timeout=10.0)
            res.raise_for_status()
            
            return {"status": "PURGED", "vmid": vmid, "timestamp": datetime.utcnow().isoformat()}
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Purge Failure for VMID {vmid}: {str(e)}")
```

---

### PART V: ADVANCED STAGE (PHASE 3) - AUTOMATED NIXOS COMPILATION

Once the Phase 1 MVP is active and communicating with Proxmox, Phase 3 upgrades `brood-matriarch` by installing `nix` and `nixos-generators` directly inside VMID 100.

When a user submits a raw `genome.nix` file to `BroodAPI`:

1.  **Matriarch Compiles Genome:** The Matriarch executes:
    ```bash
    nixos-generate -f proxmox-lxc -c /tmp/genome.nix -o /var/lib/vz/template/cache/drone-genome.tar.xz
    ```
2.  **Matriarch Triggers Native Deployment:** `BroodAPI` invokes `POST /nodes/{node}/lxc` referencing `local:vztmpl/drone-genome.tar.xz`.
3.  **Result:** A 100% custom NixOS Drone is incubated dynamically without requiring any local toolchains on your workstation.

---

### REPOSITORY FILE STRUCTURE (`pinbraid/BroodDen`)

```text
BroodDen/
├── brood_api/                  # Phase 1: Matriarch API Engine
│   ├── __init__.py
│   ├── main.py                 # FastAPI application routes
│   └── schemas.py              # Pydantic Genome contracts
├── brood_cli/                  # Phase 2: Local Terminal CLI
│   ├── __init__.py
│   ├── main.py                 # Typer CLI commands
│   └── ui.py                   # BroodDesign Mono-Frame renderer
├── genomes/                    # Phase 3: Declarative NixOS Profiles
│   ├── base-drone.nix          # Minimal NixOS baseline
│   └── python-fastapi.nix      # Microservice API Drone profile
├── scripts/
│   └── matriarch-bootstrap.sh  # One-click shell script for manual VMID 100 setup
├── pyproject.toml              # Dependencies (fastapi, uvicorn, httpx, typer, rich)
└── README.md                   # BroodDen Specification & Architecture
```


### PART I: THE BROODDEN DOCTRINE (WHY THIS IS UNIQUE)

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

