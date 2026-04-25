# TrueNAS Local AI Health Dashboard — Full Build Spec

## What This Is

A self-hosted web dashboard that runs in Docker/Dockge on TrueNAS SCALE. It pulls
live system metrics from Glances and SMART drive health from Scrutiny, then sends
the combined data to a local Ollama instance for a plain-English health summary.
Nothing leaves the local network.

---

## Full Stack

| Service    | Image                                      | Port  | Purpose                        |
|------------|--------------------------------------------|-------|--------------------------------|
| Glances    | nicolargo/glances:ubuntu-latest-full       | 61208 | CPU, RAM, GPU, IO, alerts      |
| Scrutiny   | ghcr.io/analogj/scrutiny:master-omnibus    | 31054 | SMART drive health + history   |
| Ollama     | external (existing instance)               | any   | Local LLM inference            |
| Dashboard  | nginx:alpine (bind-mounted index.html)     | 8090  | The health summary web UI      |

The Ollama service is **not** included in the compose by default — the dashboard
points at an existing Ollama instance via the "Ollama URL" field in the UI.
A commented-out Ollama service block is left in `docker-compose.yml` if you'd
rather run a fresh one inside this stack.

---

## Repository Layout

```
truenas-health-dashboard/
├── SPEC.md                  ← this file
├── docker-compose.yml       ← Glances + Scrutiny + dashboard
├── html/
│   └── index.html           ← the complete single-file web UI (bind-mounted into nginx)
└── glances/
    └── glances.conf         ← optional glances config
```

For Dockge: drop `docker-compose.yml` and the `html/` directory into the stack
folder (e.g. `/mnt/Pool_1/Configs/Dockge/health/`). No build step —
`nginx:alpine` serves `html/index.html` via a read-only directory bind mount,
so editing the file and restarting the container picks up changes immediately.

---

## 1. Dashboard — index.html

Single self-contained HTML file. No build step. Served by nginx.

**Behaviour:**
- User enters TrueNAS IP (used for Glances + Scrutiny) and a full Ollama URL
  (e.g. `http://192.168.0.10:11434`). Both are persisted to localStorage.
- Clicks "Load models" → hits `<OLLAMA_URL>/api/tags` → populates model dropdown
- Clicks "Fetch + summarise" →
  - Fetches from Glances API (port 61208): cpu, mem, gpu, load, fs, alert, diskio
  - Fetches from Scrutiny API (port 31054): /api/summary
  - Packages all data into a compact JSON payload
  - POSTs to `<OLLAMA_URL>/api/generate` with stream:true
  - Streams Ollama response token-by-token into the summary box

**System prompt sent to Ollama:**
```
You are a TrueNAS/homelab system health expert. Analyse this system data and give
a plain English health summary. Be direct. Flag problems clearly. 3-5 short
paragraphs, no bullet points, no markdown.
```

**Data payload shape sent to Ollama:**
```json
{
  "cpu": { "total_pct": 12.3, "iowait_pct": 8.1 },
  "memory": { "used_pct": 67.2, "used_gb": "21.5", "total_gb": "32" },
  "load": { "min1": 2.1, "min5": 1.8, "min15": 1.4, "cpu_cores": 12 },
  "gpu": [{ "name": "NVIDIA GeForce GTX 1060 6GB", "temp_c": 45, "vram_pct": 12, "load_pct": 3 }],
  "filesystems": [{ "mount": "/", "used_pct": 34, "free_gb": "180.2" }],
  "active_alerts": [{ "type": "CPU_IOWAIT", "state": "CRITICAL", "value": 20.4, "processes": ["storagenode","qbittorrent-nox","txg_sync"] }],
  "disk_io": [{ "disk": "sda", "read_mb_s": "0.0", "write_mb_s": "2.1" }],
  "smart_health": [
    { "device": "/dev/sda", "model": "ST8000VN004", "passed": true, "pending_sectors": 0 },
    { "device": "/dev/sde", "model": "TOSHIBA MQ01ABD100V", "passed": false, "pending_sectors": 280 }
  ]
}
```

See `html/index.html` for the complete implementation.

---

## 2. Full docker-compose.yml

This is the complete stack. Deploy as a single Dockge stack or split into individual stacks.
See `docker-compose.yml` in the repository root for the full file.

---

## 3. Known Issues and Fixes

### Glances GPU not showing
**Symptom:** GPU section empty, `pynvml.NVMLError_LibraryNotFound`
**Cause:** Alpine-based Glances image uses musl libc, incompatible with host nvidia libs
**Fix:** Use `ubuntu-latest-full` image tag (not `latest-full`)

The nvidia-container-toolkit injects `libnvidia-ml.so.1` automatically into Ubuntu-based
containers. Do NOT manually volume-mount the lib — the toolkit tries to create a symlink and
errors with "device or resource busy" if a bind mount is already there.

### CORS errors in browser
**Symptom:** Dashboard shows "Glances unreachable" but Glances is running
**Fix:** Add `--cors-allow-origins=*` to `GLANCES_OPT` environment variable.
Scrutiny allows CORS by default.

**Ollama CORS:** Ollama only allows requests from `localhost`/`127.0.0.1` by
default, so a browser dashboard on another origin gets blocked. Set
`OLLAMA_ORIGINS=*` (or to your dashboard's specific origin) on the Ollama
host. Examples:

- systemd: `sudo systemctl edit ollama` → add `Environment="OLLAMA_ORIGINS=*"` → `sudo systemctl restart ollama`
- macOS app: `launchctl setenv OLLAMA_ORIGINS "*"` then restart Ollama
- docker: add `-e OLLAMA_ORIGINS=*` to the run/compose

### Scrutiny device list
Run on TrueNAS host to get all drives:
```bash
smartctl --scan
```
Add each `/dev/sdX` to the `devices:` section in the compose.

### Seagate IronWolf Raw_Read_Error_Rate false alarm
Seagate drives report a continuously climbing raw error counter that smartd flags.
This is normal Seagate behaviour. The important attributes to watch are:
- `Reallocated_Sector_Ct` (ID 5) — must be 0
- `Current_Pending_Sector` (ID 197) — must be 0
- `Offline_Uncorrectable` (ID 198) — must be 0

A drive with non-zero pending sectors (like the Toshiba MQ01ABD100V on /dev/sde
with 280 pending sectors and 91 ATA errors) requires immediate replacement.

### Pool iowait spikes
Recurring `CPU_IOWAIT` alerts with `txg_sync`, `storagenode`, `qbittorrent-nox`
in the top process list indicate the ZFS transaction group sync is blocking on
spinning disks. Fix: add an NVMe SLOG vdev to the pool.

Quick win before SLOG arrives:
```bash
zfs set sync=disabled <poolname>/qbittorrent-dataset
```

---

## 4. First Run After Deploy

1. Deploy the stack in Dockge
2. Open Scrutiny at `http://truenas-ip:31054` and trigger first collection:
   ```bash
   docker exec scrutiny scrutiny-collector-metrics run
   ```
3. Open Glances at `http://truenas-ip:61208` — verify GPU tab shows your card
4. Make sure your existing Ollama has at least one model pulled and is reachable
   from the dashboard's browser (set `OLLAMA_ORIGINS=*` if not — see above)
5. Open the dashboard at `http://truenas-ip:8090`
6. Enter the TrueNAS IP, enter your Ollama URL (e.g. `http://192.168.0.10:11434`),
   click Load models, select model, click Fetch + summarise. Both the IP and the
   Ollama URL are saved in localStorage so you only enter them once.

---

## 5. Environment Details

- TrueNAS SCALE (Linux/Debian base)
- GPU: NVIDIA GeForce GTX 1060 6GB (driver 550.142, CUDA 12.4)
- Pool: RAIDZ1, 5-wide, 7.28 TiB spinning disks
- Metadata mirror VDEVs present, no SLOG, no L2ARC
- Failing drive: Toshiba MQ01ABD100V on /dev/sde (serial 39PECTBVT)
  - 280 Current_Pending_Sectors, 91 ATA UNC errors — replace immediately
- Seagate IronWolf ST8000VN004 on /dev/sdf — healthy (rising Raw_Read_Error_Rate
  is normal Seagate behaviour, all critical attributes zero)
