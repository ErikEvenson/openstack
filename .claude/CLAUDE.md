# OpenStack Local Deployment Project

## Communication Style

- Ask questions **one at a time** — do not batch multiple questions into a single message.
- Err on the side of asking too many questions rather than too few.

## Project Purpose

- Refresh OpenStack infrastructure and operations knowledge.
- Production-like deployment experience (not just API interaction).

## Architecture Decisions

- **OpenStack Release:** 2025.2 (Flamingo) — latest stable
- **Deployment Tool:** Kolla-Ansible (containerized services via Docker)
- **VM Hypervisor:** Lima + socket_vmnet (CNCF project, declarative YAML, multiple isolated L2 networks via vmnet-host)
- **Lima Backend:** QEMU (via Apple HVF — native ARM64 speed, snapshot support, native socket_vmnet compatibility)
- **VM Provisioning:** Lima declarative YAML configs + `limactl` CLI (not Terraform — no viable provider exists for macOS)
- **Image Building:** Packer (if needed) or Lima cloud images + cloud-init
- **Configuration Management:** Ansible (via Kolla-Ansible)
- **Post-Deploy Resource Management:** Terraform (OpenStack provider)
- **Linux Distribution:** Ubuntu
- **Topology:** Multi-node, starting with 3 nodes (scalable to 4+ later)
- **Architecture:** ARM64 native with hybrid fallback (build from source if specific aarch64 Kolla images fail)
- **Persistence:** Required (survives reboots)
- **Everything in version control** — all IaC lives in this git repo.

## Node Layout (3-Node)

| Node | Role(s) | RAM | vCPU | OS Disk | LVM Disk | Ceph OSD Disk |
|------|---------|-----|------|---------|----------|---------------|
| node1 | Controller + Network (+ Ceph MON/MGR later) | 32 GB | 6 | 60 GB | — | — |
| node2 | Compute (+ Ceph OSD later) | 28 GB | 4 | 40 GB | — | 20 GB (pre-provisioned) |
| node3 | Storage / Cinder (+ Ceph OSD later) | 12 GB | 2 | 30 GB | 10 GB | 20 GB (pre-provisioned) |

- **Totals:** 72 GB RAM, 12 vCPU, 130 GB OS disk, 10 GB LVM disk, 40 GB Ceph OSD disk
- **Nominal total disk:** 180 GB (sparse allocation — actual usage much lower initially)
- **Host reserve:** ~16 GB RAM, ~30 GB disk for macOS
- **Scaling:** Add a second compute node later by reducing controller to 24 GB and compute nodes to 20 GB each.

## Storage Strategy (Phased)

### Phase 1: LVM (Initial Deployment)
- Cinder uses LVM backend on node3 with a dedicated virtual disk
- `enable_cinder_backend_lvm: "yes"` in Kolla-Ansible globals.yml
- Gets OpenStack fully operational first

### Phase 2: Ceph (Post-Deployment Addition)
- Deploy Ceph separately via `cephadm` (Kolla-Ansible no longer deploys Ceph — removed since Ussuri)
- Co-locate MON + MGR on node1, OSDs on node2 and node3 (using pre-provisioned 20 GB disks)
- `osd_pool_default_size=2` for real replication across two hosts
- `osd_memory_target=1 GB` per OSD (tuned for lab)
- Configure Kolla-Ansible to use external Ceph for Cinder, Glance, and Nova
- Can run LVM and Ceph as simultaneous Cinder backends, migrate volumes between them
- Ceph OSD disks are provisioned from the start on node2/node3 so they're ready

## Known Risks

- **ARM64 Kolla images are "Untested"** in the official support matrix. Images are buildable and community reports are positive, but CI doesn't validate aarch64. Fallback: build individual images from source via `kolla-build`.
- **Nested virtualization:** Compute nodes run inside Lima VMs, so guest VMs inside OpenStack will use software emulation (QEMU TCG), not KVM. Expect slower guest VM performance.
- **Disk is the tightest constraint.** Aggressive Glance image management needed. Consider external drive if more space is required.

## Design Principles

- **IaC/Immutable Infrastructure** — no manual VM creation, no in-place edits. VMs are replaced, not patched.
- **Full Stack** — Nova, Neutron, Cinder, Glance, Keystone, Horizon (all core services).
- **Production-like** — always choose the production-like option. Do not simplify unless there is a hard technical constraint.
- **Neutron Backend:** OVN (modern default, distributed L3/DHCP/metadata).
- **TLS Everywhere** — all API traffic encrypted (internal and external). Self-signed CA managed by Kolla-Ansible.
- **Monitoring:** Prometheus + Grafana enabled from the start. Centralized logging (OpenSearch) deferred — add later via `kolla-ansible reconfigure`.
- **Container Engine:** Docker CE (standard for Kolla-Ansible production deployments).
- **Ubuntu Version:** 24.04 LTS (Noble).
- **Deployer:** Kolla-Ansible runs from macOS host via Python venv, SSH into Lima VMs.
- **SSH:** Dedicated key pair for the project (public key injected via cloud-init, private key in `.gitignore`).
- **DNS:** BIND on node1. Domain: `openstack.local`. Production-like setup, positions for Designate integration later (Issue #6).
- **Additional Services:** Core stack only (Nova, Neutron, Cinder, Glance, Keystone, Horizon). Others tracked as enhancement issues (#1-#9).

## Network Topology (4 Networks)

| Network | socket_vmnet Mode | Subnet | Purpose | Controller+Network | Compute | Storage |
|---------|-------------------|--------|---------|:---:|:---:|:---:|
| **Management** | shared (NAT) | 192.168.105.0/24 | APIs, DB, MQ, SSH, internet access | Yes | Yes | Yes |
| **Tunnel** | host (isolated) | 10.0.1.0/24 | VXLAN overlay (VM-to-VM) | Yes | Yes | No |
| **External** | host (isolated) | 10.0.2.0/24 | Floating IPs, br-ex (raw port, no IP) | Yes | No | No |
| **Storage** | host (isolated) | 10.0.3.0/24 | iSCSI (Cinder), Glance image transfers | Yes | Yes | Yes |

### VIP Addresses

- **`kolla_internal_vip_address`:** `192.168.105.250` (management network, keepalived)
- **`kolla_external_vip_address`:** `10.0.2.250` (external network, public API/Horizon, TLS termination)

### Kolla-Ansible Interface Mapping

| Variable | Interface Purpose | Has IP? |
|----------|-------------------|---------|
| `network_interface` | Management/API | Yes |
| `tunnel_interface` | VXLAN overlay | Yes |
| `neutron_external_interface` | External/provider (raw port into br-ex) | **No** |
| `storage_interface` | iSCSI/Glance | Yes |
| `kolla_external_vip_interface` | External VIP (HAProxy public endpoints) | Yes (VIP) |

### Notes

- `neutron_external_interface` is handed as a raw port to OVS. Any IP on it becomes unusable after deployment.
- Management network uses `shared` mode for NAT internet access (package downloads, container image pulls).
- Per-node interface overrides go in inventory `host_vars/` or `group_vars/`, NOT in `globals.yml` (which has highest precedence and overrides everything).

## Host Environment

- **Machine:** Apple M2 Max (12-core), 96GB RAM
- **OS:** macOS (Darwin)
- **Available Disk:** ~180GB (external drive possible)

## Tool Stack Summary

| Layer | Tool |
|-------|------|
| VM provisioning | Lima + socket_vmnet (declarative YAML, `limactl` CLI) |
| VM networking | socket_vmnet (isolated L2 segments via vmnet-host with UUIDs) |
| Base image building | Packer or Lima cloud images + cloud-init |
| Container engine | Docker CE |
| OpenStack deployment | Kolla-Ansible (Ansible + Docker) |
| Neutron backend | OVN |
| TLS | Everywhere (self-signed CA) |
| DNS | BIND on node1 (`openstack.local`) |
| Monitoring | Prometheus + Grafana |
| Post-deploy resources | Terraform (OpenStack provider) |
| Deployer | macOS host (Python venv + SSH) |

## Key Decision: Why Not Terraform for VM Layer

No Terraform provider exists for Lima on macOS. Multipass has a provider but cannot create isolated networks on macOS. Building a custom Lima provider (~5-7 weeks) was deemed too much of a detour from the OpenStack learning goal. Lima's declarative YAML configs are still IaC — version-controlled, reproducible, and parameterized via cloud-init. Terraform is used for post-deploy OpenStack resource management.

## Project Directory Structure

```
openstack/
├── .claude/
│   └── CLAUDE.md
├── lima/
│   ├── networks.yaml          # socket_vmnet network definitions
│   ├── node1-controller.yaml  # Lima VM config
│   ├── node2-compute.yaml
│   └── node3-storage.yaml
├── kolla/
│   ├── globals.yml            # Kolla-Ansible global config
│   ├── passwords.yml          # Service passwords (generated)
│   ├── multinode              # Ansible inventory
│   ├── host_vars/             # Per-node interface overrides
│   └── config/                # Service-specific overrides
├── terraform/
│   └── openstack/             # Post-deploy resource management
├── scripts/
│   └── ...                    # Helper scripts (bootstrap, teardown, etc.)
├── README.md
└── .gitignore
```

## Constraints

- **NEVER store credentials, passwords, keys, or secrets in the git repo.** All sensitive material goes in `.gitignore`.
- **No `null_resource` usage** in Terraform configs.
- **No Terraform at the VM layer** — Lima handles this directly.
