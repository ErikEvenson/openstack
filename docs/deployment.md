# Deployment Guide

End-to-end instructions for deploying this OpenStack 2025.2 (Flamingo) lab from a fresh `git clone` to a running cluster.

## Prerequisites

### Hardware

- Apple Silicon Mac (M-series)
- 64 GB RAM minimum (reduce VM sizes in `lima/node*.yaml` if needed)
- ~120 GB free disk space (sparse-allocated, grows on demand)

**Tested on:** M2 Max with 96 GB RAM. The default Lima configs allocate 72 GB to VMs (32+28+12 GB). On a 64 GB machine, reduce node1 to 16 GB and node2 to 16 GB in the Lima YAML files before starting the VMs.

### Software

Install via Homebrew:

```bash
brew install lima socket_vmnet qemu
```

Verify socket_vmnet is installed at `/opt/socket_vmnet/bin/socket_vmnet`:

```bash
ls /opt/socket_vmnet/bin/socket_vmnet
```

If it's not there, follow the [socket_vmnet installation instructions](https://github.com/lima-vm/socket_vmnet#installation) to build and install it with `sudo make install`.

## Step 1: Clone and Configure

### Clone the repo

```bash
git clone <repo-url> ~/Documents/Projects/openstack
cd ~/Documents/Projects/openstack
```

### Generate an SSH key pair

The Lima VM configs inject a specific SSH public key via cloud-init. Generate a new key pair for the project:

```bash
ssh-keygen -t ed25519 -f lima/openstack-lab -N "" -C "openstack-lab-deploy"
```

This creates `lima/openstack-lab` (private, gitignored) and `lima/openstack-lab.pub` (committed).

### Update the SSH public key in Lima configs

The cloud-init scripts in `lima/node{1,2,3}-*.yaml` have a hardcoded public key. Replace it with your newly generated key in all three files:

```bash
# Get your new public key
cat lima/openstack-lab.pub

# Replace the old key in all Lima configs
# Look for the line starting with "ssh-ed25519" inside the cloud-init provision block
# and replace it with your new public key
```

The key appears twice in each file (once for root, once for the default user). Example of what to find and replace:

```yaml
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... openstack-lab-deploy" >> /root/.ssh/authorized_keys
```

### Update the `node_custom_config` path in globals.yml

`kolla/globals.yml` contains an absolute path that must match your local checkout:

```yaml
node_custom_config: "/Users/eevenson/Documents/Projects/openstack/kolla/config"
```

Update this to your actual path:

```bash
# If your repo is cloned elsewhere, update the path:
# e.g., /Users/yourname/Documents/Projects/openstack/kolla/config
```

### Update the SSH key path in the inventory

`kolla/inventory` also references an absolute path to the SSH private key:

```
ansible_ssh_private_key_file=/Users/eevenson/Documents/Projects/openstack/lima/openstack-lab
```

Update this to match your checkout location.

## Step 2: Network Setup

Copy the Lima network definitions to the Lima config directory:

```bash
mkdir -p ~/.lima/_config
cp lima/networks.yaml ~/.lima/_config/networks.yaml
```

This defines four networks:

| Network | Mode | Subnet | Purpose |
|---------|------|--------|---------|
| management | shared (NAT) | 192.168.105.0/24 | APIs, DB, MQ, SSH, internet |
| tunnel | host (isolated) | 10.0.1.0/24 | VXLAN overlay |
| external | host (isolated) | 10.0.2.0/24 | Floating IPs, br-ex |
| storage | host (isolated) | 10.0.3.0/24 | iSCSI, Glance |

## Step 3: Start the VMs

Start all three nodes. Order doesn't matter, but node1 is the largest and takes longest:

```bash
limactl start lima/node1-controller.yaml
limactl start lima/node2-compute.yaml
limactl start lima/node3-storage.yaml
```

Each VM will boot, run cloud-init (static IPs, hostname, packages), and become available via SSH. This takes a few minutes per node.

### Verify SSH access

```bash
ssh -i lima/openstack-lab -o StrictHostKeyChecking=no root@192.168.105.10 hostname
ssh -i lima/openstack-lab -o StrictHostKeyChecking=no root@192.168.105.11 hostname
ssh -i lima/openstack-lab -o StrictHostKeyChecking=no root@192.168.105.12 hostname
```

Expected output: `node1.openstack.local`, `node2.openstack.local`, `node3.openstack.local`.

### Verify networking

Confirm each node can reach the others on the management network:

```bash
ssh -i lima/openstack-lab root@192.168.105.10 'ping -c1 192.168.105.11 && ping -c1 192.168.105.12'
```

## Step 4: Python Virtual Environment

Create a Python venv and install Kolla-Ansible:

```bash
python3 -m venv venv
source venv/bin/activate

# Pin bcrypt FIRST — passlib 1.7.4 is broken with bcrypt >= 4.1
pip install 'bcrypt<4.1'

# Install Kolla-Ansible (matches 2025.2 Flamingo)
pip install 'kolla-ansible==21.0.0'
```

> **Why pin bcrypt?** passlib 1.7.4 (a Kolla-Ansible dependency) calls a removed `bcrypt.__about__` attribute in bcrypt 4.1+, causing `kolla-genpwd` and password hashing to fail silently or error out.

## Step 5: Generate Passwords

Kolla-Ansible requires a `passwords.yml` file containing auto-generated service passwords. This file is gitignored and must be generated locally:

```bash
source venv/bin/activate
kolla-genpwd --passwords kolla/passwords.yml
```

This populates `kolla/passwords.yml` with random passwords for all OpenStack services (MariaDB, RabbitMQ, Keystone, etc.).

## Step 6: Generate TLS Certificates

The repo commits only the OpenSSL config files (`openssl-kolla.cnf`, `openssl-kolla-backend.cnf`) and CSRs. The actual certificates and keys are gitignored and must be regenerated:

```bash
source venv/bin/activate
kolla-ansible certificates \
  --configdir kolla
```

This generates:

- Self-signed root CA (`kolla/certificates/ca/root.crt`)
- HAProxy frontend certificates (`haproxy.pem`, `haproxy-internal.pem`)
- Backend TLS certificates (`backend-cert.pem`, `backend-key.pem`)
- MariaDB/ProxySQL certificates

All certificates use the SANs defined in `kolla/certificates/openssl-kolla.cnf`:

- DNS: `controller.openstack.local`
- IP: `192.168.105.250`

## Step 7: Deploy OpenStack

The deployment has three phases. All commands must include `--limit` to exclude the macOS deploy host (which fails Linux OS prechecks) and `--configdir` to point at the kolla config directory.

```bash
source venv/bin/activate
```

### Bootstrap the servers

Installs Docker, configures the container engine, and prepares the nodes:

```bash
kolla-ansible bootstrap-servers \
  --configdir kolla \
  -i kolla/inventory \
  --limit 'node1,node2,node3'
```

### Run prechecks

Validates that all prerequisites are met before deployment:

```bash
kolla-ansible prechecks \
  --configdir kolla \
  -i kolla/inventory \
  --limit 'node1,node2,node3'
```

### Deploy

Pulls container images and starts all OpenStack services:

```bash
kolla-ansible deploy \
  --configdir kolla \
  -i kolla/inventory \
  --limit 'node1,node2,node3'
```

This takes 20-40 minutes depending on network speed (image pulls) and disk performance. It deploys ~66 containers across the three nodes.

> **If a node fails:** One fatal error on a node kills all subsequent plays for that node. Fix the root cause and re-run `deploy` — don't chase the cascading failures.

## Step 8: Post-Deploy

### Generate the admin openrc file

```bash
kolla-ansible post-deploy \
  --configdir kolla \
  -i kolla/inventory \
  --limit 'node1,node2,node3'
```

This generates `kolla/admin-openrc.sh` and `kolla/clouds.yaml` (both gitignored).

### Add `/etc/hosts` entry on your Mac

```bash
sudo bash -c 'echo "192.168.105.250 controller.openstack.local" >> /etc/hosts'
```

### Verify services

SSH into node1 and check that all services are running:

```bash
ssh -i lima/openstack-lab root@192.168.105.10 'docker ps --format "table {{.Names}}\t{{.Status}}" | head -20'
```

Use the OpenStack CLI via `kolla_toolbox`:

```bash
ssh -i lima/openstack-lab root@192.168.105.10 \
  'docker exec kolla_toolbox openstack \
    --os-project-domain-name Default \
    --os-user-domain-name Default \
    --os-project-name admin \
    --os-username admin \
    --os-password '"'"'<password>'"'"' \
    --os-auth-url https://controller.openstack.local:5000 \
    --os-interface internal \
    --os-identity-api-version 3 \
    --os-region-name RegionOne \
    --os-cacert /etc/ssl/certs/ca-certificates.crt \
    service list'
```

Replace `<password>` with the Keystone admin password from:

```bash
grep OS_PASSWORD kolla/admin-openrc.sh
```

### Access Horizon (Web Dashboard)

Open `https://controller.openstack.local` in your browser (accept the self-signed certificate warning).

- **Domain:** Default
- **User:** admin
- **Password:** see `grep OS_PASSWORD kolla/admin-openrc.sh`

### Access Grafana (Monitoring)

Open `https://controller.openstack.local:3000`.

- **User:** admin
- **Password:** `grep grafana_admin_password kolla/passwords.yml | awk '{print $2}'`

## Next Steps

Once the cluster is running, follow these guides:

- [First Instance (CLI)](first-instance.md) -- upload an image, create networks, launch a VM, assign floating IPs
- [First Instance (GUI)](first-instance-gui.md) -- same workflow via the Horizon dashboard

## Using Claude Code as an Assistant

This project includes a `.claude/CLAUDE.md` file that gives Claude Code full context about the architecture, network topology, node layout, and design decisions. When working with Claude Code in this repo:

- **Troubleshooting:** Describe the error and Claude Code can reference the architecture to suggest fixes. It knows the network layout, which services run where, and common pitfalls.
- **Reconfiguration:** Ask Claude Code to modify `globals.yml`, inventory, or service overrides. It understands the Kolla-Ansible config structure and inter-service dependencies.
- **Day 2 operations:** Claude Code can help with `kolla-ansible reconfigure`, adding new services (tracked as GitHub issues #1-#13), or scaling to additional nodes.
- **OpenStack CLI:** Claude Code knows the `kolla_toolbox` wrapper pattern for running `openstack` commands on node1.

## Teardown and Pause

### Pause (preserve state)

Stop the VMs without destroying them. All data and containers are preserved:

```bash
limactl stop node3-storage
limactl stop node2-compute
limactl stop node1-controller
```

Resume later:

```bash
limactl start node1-controller
limactl start node2-compute
limactl start node3-storage
```

OpenStack services auto-start inside their containers on boot.

### Full teardown (destroy everything)

```bash
limactl delete node3-storage --force
limactl delete node2-compute --force
limactl delete node1-controller --force
```

This deletes all VM disks. To redeploy, start again from Step 3.

## Troubleshooting

### `kolla-genpwd` fails or produces empty passwords

**Cause:** `bcrypt >= 4.1` breaks `passlib 1.7.4`.

**Fix:** Pin bcrypt in your venv:

```bash
pip install 'bcrypt<4.1'
```

### No Ubuntu Noble aarch64 images on quay.io

Kolla does not publish Ubuntu Noble (24.04) images for aarch64. The project uses Debian Bookworm instead:

```yaml
# globals.yml
kolla_base_distro: "debian"
openstack_tag_suffix: "-aarch64"
```

The Lima VMs still run Ubuntu Noble as the host OS — only the Kolla container images use Debian.

### Deploy host fails OS prechecks

The macOS deploy host cannot pass Linux-specific prechecks (kernel modules, sysctl, etc.). Always use `--limit`:

```bash
kolla-ansible <command> --limit 'node1,node2,node3' ...
```

### VNC console frozen in Horizon

Nested QEMU with TCG (software emulation) causes the virtio-gpu framebuffer to freeze after boot. The noVNC console displays the boot screen but doesn't accept input.

**Workaround:** SSH into instances via floating IPs, or use the serial console:

```bash
osc console url show --type serial <instance-name>
```

### Slow guest VM performance

Guest VMs run under QEMU TCG (no KVM available inside Lima VMs). This is expected. The `cpu_mode=custom` / `cpu_models=max` config in `kolla/config/nova/nova-compute.conf` provides the best available emulated CPU for aarch64.

### `kolla_copy_ca_into_containers` errors

If you see errors about missing backend certificates, ensure `kolla_enable_tls_backend` is set to `"yes"` in `globals.yml`. The `kolla_copy_ca_into_containers` option triggers certificate copy for ALL services, which requires backend certs to exist.

### Certificates not trusted by OpenStack CLI

Python's `requests` library uses `certifi` for CA validation, not the system trust store. The self-signed CA is injected into containers via `kolla_copy_ca_into_containers`, and `openstack_cacert` in `globals.yml` points to the system CA bundle inside the containers:

```yaml
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
```

If running the `openstack` CLI outside of containers (e.g., from your Mac), you need to export:

```bash
export OS_CACERT=/path/to/kolla/certificates/ca/root.crt
```

### One node failed, everything after it shows errors

Kolla-Ansible aborts all remaining plays for a node after a fatal error. The cascading failures are symptoms, not separate problems. Fix the root cause (usually the first error in the output) and re-run.
