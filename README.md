# OpenStack Local Lab

A production-like OpenStack 2025.2 (Flamingo) deployment running locally on macOS Apple Silicon via Lima VMs and Kolla-Ansible.

## Purpose

Refresh OpenStack infrastructure and operations knowledge through a hands-on, multi-node deployment that mirrors production architecture.

## Architecture

### Tool Stack

| Layer | Tool |
|-------|------|
| VM provisioning | Lima + socket_vmnet (QEMU backend via HVF) |
| VM networking | socket_vmnet (isolated L2 segments) |
| Container engine | Docker CE |
| OpenStack deployment | Kolla-Ansible |
| Neutron backend | OVN |
| TLS | Everywhere (self-signed CA) |
| DNS | BIND on node1 (`openstack.local`) |
| Monitoring | Prometheus + Grafana |
| Post-deploy resources | Terraform (OpenStack provider) |
| Deployer | macOS host (Python venv + SSH) |

### Node Layout (3-Node)

| Node | Role(s) | RAM | vCPU | OS Disk | Extra Disk |
|------|---------|-----|------|---------|------------|
| node1 | Controller + Network + DNS (BIND) | 32 GB | 6 | 60 GB | -- |
| node2 | Compute | 28 GB | 4 | 40 GB | 20 GB (Ceph OSD) |
| node3 | Storage (Cinder LVM) | 12 GB | 2 | 30 GB | 10 GB (LVM) + 20 GB (Ceph OSD) |

### Network Topology (4 Networks)

| Network | Mode | Subnet | Purpose |
|---------|------|--------|---------|
| Management | shared (NAT) | 192.168.105.0/24 | APIs, DB, MQ, SSH |
| Tunnel | host (isolated) | 10.0.1.0/24 | VXLAN overlay |
| External | host (isolated) | 10.0.2.0/24 | Floating IPs, br-ex |
| Storage | host (isolated) | 10.0.3.0/24 | iSCSI, Glance |

### VIP Addresses

- **Internal:** `192.168.105.250` (management network)
- **External:** `10.0.2.250` (external network, TLS termination)

### Security

- TLS everywhere (internal + external) with self-signed CA
- Separate internal and external VIPs
- Dedicated SSH key pair for deployment

### Storage Strategy (Phased)

1. **Phase 1:** LVM backend for Cinder (initial deployment)
2. **Phase 2:** Add Ceph via cephadm (MON/MGR on node1, OSDs on node2/node3)

### OpenStack Services

**Core (initial deployment):** Nova, Neutron (OVN), Cinder, Glance, Keystone, Horizon

**Future enhancements (tracked as issues):**
Heat (#1), Barbican (#2), Octavia (#3), Manila (#4), Swift (#5), Designate (#6), Magnum (#7), Ironic (#8), Telemetry (#9), Ceph (#10), Centralized Logging (#11), Second Compute Node (#12), Lima Terraform Provider (#13)

## Host Requirements

- Apple Silicon Mac (M2 Max or similar)
- 96 GB RAM (72 GB allocated to VMs, 16 GB reserved for macOS)
- ~180 GB free disk
- Homebrew

### Dependencies

```
brew install lima socket_vmnet qemu
```

## Project Structure

```
openstack/
├── lima/           # VM definitions and network config
├── kolla/          # Kolla-Ansible configuration
├── terraform/      # Post-deploy OpenStack resource management
└── scripts/        # Helper scripts
```

## Key Decisions

- **Lima over Multipass** -- Multipass cannot create isolated networks on macOS. Lima + socket_vmnet provides true L2 isolation via vmnet-host.
- **No Terraform for VM layer** -- No viable Terraform provider exists for Lima on macOS. Lima's declarative YAML configs serve as IaC.
- **ARM64 native** -- Kolla aarch64 images are officially "Untested" but community reports are positive. Fallback: build from source via `kolla-build`.
- **OVN over OVS** -- Modern default, distributed control plane, production standard for new deployments.
- **BIND for DNS** -- Production-like, positions for Designate integration later.
- **Deploy from macOS** -- Kolla-Ansible runs from host via Python venv, SSH into VMs.
- **QEMU over VZ** -- Snapshot support, native socket_vmnet compatibility, stable networking. Performance identical for ARM64.
