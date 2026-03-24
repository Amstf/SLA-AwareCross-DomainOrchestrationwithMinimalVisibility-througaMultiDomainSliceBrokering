# 5G Testbed Setup Guide

This guide covers the full deployment of the 5G testbed across four components:

| Component | Submodule | Container | Status |
|-----------|-----------|-----------|--------|
| Core Network (OAI CN5G — network slicing) | `Core-Network` | `CORE-NETWORK` | ✅ Documented |
| gNB (OAI NR with E2 agent) | `OAI-GNB-Network` | `RAN-CONTAINER` | 🚧 Coming soon |
| nearRT-RIC (OSC RIC platform) | `OSC-RIC-Network` | `RIC-CONTAINER` | 🚧 Coming soon |
| MVNO Broker xApp | `MVNO-BROKER` | — (runs on RIC) | 🚧 Coming soon |

---

## Prerequisites

**Clone the repository with all submodules:**
```bash
git clone --recurse-submodules <repo-url>
# or, if already cloned:
git submodule update --init --recursive
```

**LXC must be configured on the host.** Refer to `<add-your-lxc-setup-doc-here>`.

**GitHub SSH key** must be generated and registered before running any setup script. Refer to the Pre-step in each component's `src/README.md`.

---

## 1. Core Network Setup

The core network runs inside an LXC container (`CORE-NETWORK`) and uses Docker Compose to deploy the OAI 5G core with two independent network slices.

For full details on each script and execution step, refer to [`Core-Network/src/README.md`](Core-Network/src/README.md).

---

### 1.1 Provision the container

```bash
cd Core-Network/src/setup_container/
./setup_core_container.sh <image>.tar.gz <image> CORE-NETWORK <username> ~/.ssh/github-keys
```

| Argument | Description |
|----------|-------------|
| `<image>.tar.gz` | LXC image archive to import |
| `<image>` | Alias assigned to the imported image |
| `CORE-NETWORK` | LXC container name |
| `<username>` | Colosseum username |
| `~/.ssh/github-keys` | GitHub SSH private key path |

The script runs 5 sequential steps: image download, container import and launch, network configuration, static IP assignment, SSH key injection, repository clone, and Docker image pull.

Expected output (abbreviated):
```
✅ Container 'CORE-NETWORK' is ready.
✅ eth0 is now static at 10.87.204.x
✅ SSH authentication to GitHub succeeded.
Pulling oai-nrf        ... done
Pulling oai-amf        ... done
Pulling oai-smf-slice1 ... done
Pulling oai-upf-slice1 ... done
...
✅ All steps completed successfully for container 'CORE-NETWORK'
```

Verify the container is running on the host:
```bash
lxc list
```

Expected entry:
```
| CORE-NETWORK | RUNNING | 192.168.70.129 (br-...)  |  | CONTAINER | 0 |
|              |         | 172.18.0.1 (docker0)     |  |           |   |
|              |         | 10.87.204.x (eth0)       |  |           |   |
```

---

### 1.2 Start the core network

Enter the container and launch the core network stack:

```bash
lxc exec CORE-NETWORK -- bash
cd /root/Core-Network/oai-cn5g
./start_cn.sh -m rfsim    # or: -m usrp for Colosseum
```

The script recreates the Docker bridge network, tears down any previous deployment, starts all NF containers, and injects UPF tunnel gateway addresses.

Expected output:
```
[slicing] Mode: rfsim — using interface: eth0
[slicing] Recreating demo-oai-public-net network
[slicing] Bringing down previous deployment (if any)
[slicing] Starting slicing core services
Creating mysql          ... done
Creating oai-nrf        ... done
Creating oai-nssf       ... done
Creating oai-udr        ... done
Creating oai-udm        ... done
Creating oai-ausf       ... done
Creating oai-amf        ... done
Creating oai-smf-slice1 ... done
Creating oai-smf-slice2 ... done
Creating oai-upf-slice1 ... done
Creating oai-upf-slice2 ... done
Creating oai-ext-dn     ... done
[slicing] Configuring UPF tunnel gateways on eth0
```

---

### 1.3 Verify

```bash
docker ps
```

All 12 containers must be in `Up (healthy)` state:

```
NAMES             IMAGE                                      STATUS
oai-ext-dn        oaisoftwarealliance/trf-gen-cn5g:latest    Up (healthy)
oai-smf-slice1    oaisoftwarealliance/oai-smf:develop        Up (healthy)
oai-smf-slice2    oaisoftwarealliance/oai-smf:develop        Up (healthy)
oai-amf           oaisoftwarealliance/oai-amf:develop        Up (healthy)
oai-ausf          oaisoftwarealliance/oai-ausf:develop       Up (healthy)
oai-udm           oaisoftwarealliance/oai-udm:develop        Up (healthy)
oai-udr           oaisoftwarealliance/oai-udr:develop        Up (healthy)
oai-nssf          oaisoftwarealliance/oai-nssf:develop       Up (healthy)
oai-nrf           oaisoftwarealliance/oai-nrf:develop        Up (healthy)
oai-upf-slice1    oaisoftwarealliance/oai-upf:develop        Up (healthy)
oai-upf-slice2    oaisoftwarealliance/oai-upf:develop        Up (healthy)
mysql             mysql:8.0                                  Up (healthy)
```

Verify the UPF tunnel addresses were injected correctly:
```bash
docker exec oai-upf-slice1 ip addr show eth0
docker exec oai-upf-slice2 ip addr show eth0
```

Expected addresses:
- `oai-upf-slice1` → `12.1.1.1/24`
- `oai-upf-slice2` → `12.1.2.1/24`

> If a container is not healthy or the UPF addresses are missing, inspect logs with `docker logs <container-name>` before proceeding.

The AMF is reachable at `192.168.70.132`. The gNB must point to this address for N2 connectivity.

---

## 2. gNB Setup

> 🚧 Coming soon. Will cover LXC container provisioning, OAI gNB compilation, and launch in both `rfsim` and `usrp` modes.
>
> Refer to [`OAI-GNB-Network/src/README.md`](OAI-GNB-Network/src/README.md) and [`OAI-GNB-Network/oai_ran/README.md`](OAI-GNB-Network/oai_ran/README.md) in the meantime.

---

## 3. nearRT-RIC Setup

> 🚧 Coming soon. Will cover LXC container provisioning, OSC RIC platform deployment, and xApp onboarding via `dms_cli`.
>
> Refer to [`OSC-RIC-Network/src/README.md`](OSC-RIC-Network/src/README.md) in the meantime.

---

## 4. MVNO Broker xApp

The MVNO Broker is an O-RAN nearRT-RIC xApp implementing a tick-based multi-operator resource brokering control loop. It runs on top of the deployed RIC (Section 3) and connects to the gNB E2 agent (Section 2) to receive per-UE RAN indications and actuate rate and slice control decisions.

> 🚧 Deployment integration into this guide is coming soon. This section will cover xApp onboarding via `dms_cli`, MongoDB setup, launch configuration, and end-to-end verification with a running gNB and RIC.

### Overview

At each tick, the control loop performs the following operations:

1. Receives per-UE RAN indications from the gNB E2 agent over UDP / Protocol Buffers
2. Aggregates per-UE KPIs (throughput, BLER, MCS, PRBs, RSRP) into a system snapshot
3. Invokes the broker decision engine in one of three operating modes:

| Mode | Flag | Behavior |
|------|------|----------|
| Dynamic (SLA + cost) | *(default)* | SLA steering restores compliance when violated; cost rebalancing shifts load to cheaper domains during safe intervals |
| SLA-only | `--sla` | SLA steering only; cost rebalancing disabled |
| Static | `--static` | Fixed UE placement; no steering or actuation |

4. Actuates rate caps and slice assignments back to the gNB
5. Persists per-tick metrics and decisions to MongoDB (`Paper1` database)

### Prerequisites

- Python 3.8
- MongoDB instance reachable from the xApp host:
  ```bash
  export MONGODB_URI="mongodb://<host>:<port>"
  ```
- Python dependencies:
  ```bash
  pip install -r requirements.txt
  ```

### Configuration

Three files in `MVNO-BROKER/conf/` control broker behavior:

| File | Purpose |
|------|---------|
| `config_loop.json` | gNB targets, slice identity (SST/SD), broker tuning parameters, SLA thresholds |
| `background_traffic_gnb.json` | Capacity-generator parameters, phase progression, pricing coefficients per domain |
| `ue_placement.conf` | UE-to-gNB mapping: pod name, port, role (active/ghost), initial rate demand |

Refer to [`MVNO-BROKER/conf/configuration_reference.md`](MVNO-BROKER/conf/configuration_reference.md) for the full parameter reference.

### Launch

```bash
cd MVNO-BROKER
python3 xapp_main.py \
  --config conf/config_loop.json \
  --traffic conf/background_traffic_gnb.json \
  --ue-placement conf/ue_placement.conf
```

Refer to [`MVNO-BROKER/README.md`](MVNO-BROKER/README.md) for the full CLI reference, MongoDB persistence schema, and broker timing details.

---

## Notes

- The container base image runs **Ubuntu 18.04 (Bionic)** — this is expected.
- The `apt` lock warning (`E: Could not get lock`) on first container run is benign; the script retries automatically.
- All Docker images are pulled inside the container during provisioning. Internet access from inside the container is required.
- The `-m rfsim` mode binds UPF gateways to `eth0`. On Colosseum with USRP hardware, use `-m usrp` to bind to `tun0` instead.
