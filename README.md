# 5G Testbed Setup Guide

This guide covers the full deployment of the 5G testbed across three components:

| Component | Container | Status |
|-----------|-----------|--------|
| Core Network (OAI CN5G — network slicing) | `CORE-NETWORK` | ✅ Documented |
| gNB (OAI NR with E2 agent) | `RAN-CONTAINER` | 🚧 Coming soon |
| nearRT-RIC (OSC RIC platform + xApp) | `RIC-CONTAINER` | 🚧 Coming soon |

---

## Prerequisites

**Clone the repository with all submodules:**
```bash
git clone --recurse-submodules <repo-url>
# or, if already cloned:
git submodule update --init --recursive
```

**LXC must be configured on the host.** Refer to `<add-lxc-setup-doc-here>`.

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
lxc exec CORE-NETWORK \bin\bash
cd /root/Core-Network/oai-cn5g
./start_cn.sh -m rfsim    
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

The AMF is reachable at `192.168.70.132`. The gNB must point to this address for N2 connectivity.

---

## 2. gNB Setup

> 🚧 Coming soon. Will cover LXC container provisioning, OAI gNB compilation, and launch in both `rfsim` and `usrp` modes.
>
> Refer to [`OAI-RAN-Network/src/README.md`](OAI-RAN-Network/src/README.md) and [`OAI-RAN-Network/oai_ran/README.md`](OAI-RAN-Network/oai_ran/README.md) in the meantime.

---

## 3. nearRT-RIC Setup

> 🚧 Coming soon. Will cover LXC container provisioning, OSC RIC platform deployment, and xApp onboarding.
>
> Refer to [`OSC-RIC-Network/src/README.md`](OSC-RIC-Network/src/README.md) in the meantime.

---
