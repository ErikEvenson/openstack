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
| Guest OS | Debian 12 (Bookworm) aarch64 |
| OpenStack deployment | Kolla-Ansible 21.0.0 |
| Container images | `quay.io/openstack.kolla/*:2025.2-debian-bookworm-aarch64` |
| Neutron backend | OVN |
| TLS | Everywhere — internal, external, and backend (self-signed CA) |
| Monitoring | Prometheus + Grafana |
| Post-deploy resources | Terraform (OpenStack provider) |
| Deployer | macOS host (Python venv + SSH) |

### Node Layout (3-Node)

| Node | Role(s) | RAM | vCPU | OS Disk | Extra Disk |
|------|---------|-----|------|---------|------------|
| node1 | Controller + Network + Monitoring | 32 GB | 6 | 60 GB | -- |
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
- **External:** `192.168.105.250` (same as internal — no separate external LB network in this lab)

### Security

- TLS everywhere (internal + external + backend) with self-signed CA
- Dedicated SSH key pair for deployment
- All secrets (passwords, keys, openrc files) excluded from git

### Storage Strategy (Phased)

1. **Phase 1:** LVM backend for Cinder (initial deployment)
2. **Phase 2:** Add Ceph via cephadm (MON/MGR on node1, OSDs on node2/node3)

### OpenStack Services (Deployed)

Nova, Neutron (OVN), Cinder (LVM), Glance, Keystone, Placement, Horizon, Prometheus, Grafana

**66 containers** across 3 nodes (44 on controller, 14 on compute, 8 on storage).

**Future enhancements (tracked as issues):**
Heat (#1), Barbican (#2), Octavia (#3), Manila (#4), Swift (#5), Designate (#6), Magnum (#7), Ironic (#8), Telemetry (#9), Ceph (#10), Centralized Logging (#11), Second Compute Node (#12), Lima Terraform Provider (#13), DNS/BIND (#14)

## Host Requirements

- Apple Silicon Mac (M2 Max or similar)
- 96 GB RAM (72 GB allocated to VMs, 16 GB reserved for macOS)
- ~180 GB free disk
- Homebrew

### Dependencies

```
brew install lima socket_vmnet qemu
```

## Getting Started

See the [Deployment Guide](docs/deployment.md) for end-to-end instructions from `git clone` to a running cluster.

## Project Structure

```
openstack/
├── docs/           # Deployment guide, first-instance walkthroughs
├── lima/           # VM definitions, network config, SSH key pair
├── kolla/          # Kolla-Ansible config (globals, inventory, certificates)
│   └── config/     # Service config overrides (merged by Kolla-Ansible)
├── terraform/      # Post-deploy OpenStack resource management (future)
└── venv/           # Python virtualenv with kolla-ansible (gitignored)
```

## Known Limitations

- **VNC console frozen on aarch64** -- Nested QEMU with TCG (software emulation) causes the virtio-gpu framebuffer to freeze after boot. The VNC console in Horizon displays the boot screen but does not accept keyboard input. Workaround: SSH into instances via floating IPs, or use the serial console.
- **Nested virtualization is slow** -- Guest VMs inside OpenStack use QEMU TCG (no KVM), so expect reduced performance. The `cpu_mode=custom` / `cpu_models=max` config in `kolla/config/nova/nova-compute.conf` provides the best available emulated CPU.

## Key Decisions

- **Lima over Multipass** -- Multipass cannot create isolated networks on macOS. Lima + socket_vmnet provides true L2 isolation via vmnet-host.
- **No Terraform for VM layer** -- No viable Terraform provider exists for Lima on macOS. Lima's declarative YAML configs serve as IaC.
- **Debian Bookworm over Ubuntu Noble** -- Ubuntu Noble lacks aarch64 Kolla images on quay.io. Debian Bookworm aarch64 images are available and working.
- **OVN over OVS** -- Modern default, distributed control plane, production standard for new deployments.
- **Deploy from macOS** -- Kolla-Ansible runs from host via Python venv, SSH into VMs. Requires `bcrypt<4.1` for passlib compatibility on Python 3.14.
- **QEMU over VZ** -- Snapshot support, native socket_vmnet compatibility, stable networking. Performance identical for ARM64.
