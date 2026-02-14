# Launching Your First Instance (Horizon GUI)

This guide walks through the same steps as the CLI guide, but entirely through the Horizon web dashboard.

## Prerequisites

Add the controller hostname to your Mac's `/etc/hosts`:

```bash
sudo bash -c 'echo "192.168.105.250 controller.openstack.local" >> /etc/hosts'
```

## Log In to Horizon

1. Open `https://controller.openstack.local` in your browser
2. Accept the self-signed certificate warning
3. Log in:
   - **Domain:** Default
   - **User Name:** admin
   - **Password:** see `grep OS_PASSWORD kolla/admin-openrc.sh`

## Step 1: Upload a Test Image

**Admin > Compute > Images > Create Image**

| Field | Value |
|-------|-------|
| Image Name | `cirros` |
| Image Description | CirrOS aarch64 test image |
| Image Source | Image Location |
| Image Location | `https://download.cirros-cloud.net/0.6.3/cirros-0.6.3-aarch64-disk.img` |
| Format | QCOW2 - QEMU Emulator |
| Image Sharing > Visibility | Public |

Under **Metadata**, add:
- `hw_architecture` = `aarch64`

Click **Create Image**. Wait for the status to change from "Saving" to "Active".

### Create Flavors

**Admin > Compute > Flavors > Create Flavor**

Create these three flavors:

| Name | VCPUs | RAM (MB) | Root Disk (GB) |
|------|-------|----------|----------------|
| m1.tiny | 1 | 512 | 1 |
| m1.small | 1 | 2048 | 20 |
| m1.medium | 2 | 4096 | 40 |

For each: fill in the fields on the **Flavor Info** tab, leave **Flavor Access** empty (public), and click **Create Flavor**.

## Step 2: Create a Tenant Network and Router

### Create the Network

**Project > Network > Networks > Create Network**

**Network** tab:

| Field | Value |
|-------|-------|
| Network Name | `demo-net` |
| Enable Admin State | checked |
| Create Subnet | checked |

**Subnet** tab:

| Field | Value |
|-------|-------|
| Subnet Name | `demo-subnet` |
| Network Address | `172.16.0.0/24` |
| IP Version | IPv4 |
| Gateway IP | `172.16.0.1` |

**Subnet Details** tab:

| Field | Value |
|-------|-------|
| Enable DHCP | checked |
| DNS Name Servers | `8.8.8.8` |

Click **Create**.

### Create the Router

**Project > Network > Routers > Create Router**

| Field | Value |
|-------|-------|
| Router Name | `demo-router` |
| Enable Admin State | checked |
| External Network | (leave empty for now -- we'll set this in Step 4) |

Click **Create Router**.

### Attach the Router to the Network

1. Click on **demo-router** to open its detail page
2. Go to the **Interfaces** tab
3. Click **Add Interface**
4. Select `demo-subnet` from the **Subnet** dropdown
5. Click **Submit**

## Step 3: Launch a Test Instance

### Create a Security Group

**Project > Network > Security Groups > Create Security Group**

| Field | Value |
|-------|-------|
| Name | `demo-sg` |
| Description | Allow SSH and ICMP |

Click **Create Security Group**. You'll be taken to the rules page.

**Add ICMP rule:**
1. Click **Add Rule**
2. Rule: `All ICMP`
3. Direction: `Ingress`
4. Click **Add**

**Add SSH rule:**
1. Click **Add Rule**
2. Rule: `SSH`
3. Click **Add**

### Import a Key Pair

Generate a key on your Mac first (if you haven't already):

```bash
ssh-keygen -t ed25519 -f ~/.ssh/openstack-demo -N "" -C "openstack-demo"
cat ~/.ssh/openstack-demo.pub
```

Copy the public key output.

**Project > Compute > Key Pairs > Import Public Key**

| Field | Value |
|-------|-------|
| Key Pair Name | `demo-key` |
| Key Type | SSH Key |
| Public Key | (paste the contents of `~/.ssh/openstack-demo.pub`) |

Click **Import Public Key**.

### Launch the Instance

**Project > Compute > Instances > Launch Instance**

**Details** tab:

| Field | Value |
|-------|-------|
| Instance Name | `test-vm` |
| Count | 1 |

**Source** tab:

| Field | Value |
|-------|-------|
| Select Boot Source | Image |
| Create New Volume | No |

Click the **up arrow** next to `cirros` to move it to "Allocated".

**Flavor** tab:

Click the **up arrow** next to `m1.tiny` to select it.

**Networks** tab:

Click the **up arrow** next to `demo-net` to select it. Remove any other networks if present.

**Security Groups** tab:

Move `demo-sg` to "Allocated". Remove `default` if you prefer.

**Key Pair** tab:

Verify `demo-key` is selected (allocated).

Click **Launch Instance**.

### Verify the Instance

The instance will appear in the **Instances** list. Wait for:
- **Status:** Active
- **Power State:** Running

Click the instance name to see details. Use the **Console** tab to access the noVNC console directly in your browser.

> CirrOS login: `cirros` / `gocubsgo`

## Step 4: Set Up External Network and Floating IPs

### Create the External Provider Network

This step requires the **Admin** panel.

**Admin > Network > Networks > Create Network**

**Network** tab:

| Field | Value |
|-------|-------|
| Name | `external-net` |
| Project | admin |
| Provider Network Type | Flat |
| Physical Network | `physnet1` |
| Enable Admin State | checked |
| External Network | **checked** |

**Subnet** tab:

| Field | Value |
|-------|-------|
| Subnet Name | `external-subnet` |
| Network Address | `10.0.2.0/24` |
| IP Version | IPv4 |
| Gateway IP | `10.0.2.1` |

**Subnet Details** tab:

| Field | Value |
|-------|-------|
| Enable DHCP | **unchecked** |
| Allocation Pools | `10.0.2.100,10.0.2.200` |

Click **Create**.

### Set the Router's External Gateway

1. Go to **Project > Network > Routers**
2. In the `demo-router` row, click the dropdown arrow on the right and select **Set Gateway**
3. **External Network:** select `external-net`
4. Click **Set Gateway**

### Assign a Floating IP to the Instance

1. Go to **Project > Compute > Instances**
2. In the `test-vm` row, click the dropdown arrow and select **Associate Floating IP**
3. Click the **+** button next to "IP Address" to allocate a new floating IP
4. **Pool:** `external-net`
5. Click **Allocate IP**
6. The new IP (e.g., `10.0.2.100`) will be selected
7. **Port to be associated:** select the `test-vm` port
8. Click **Associate**

### Test Connectivity from Your Mac

Your Mac already has `bridge102` on `10.0.2.0/24`, so the floating IP is directly reachable:

```bash
# Ping the instance
ping 10.0.2.100

# SSH in (CirrOS)
ssh -i ~/.ssh/openstack-demo cirros@10.0.2.100
```

## Grafana Monitoring

Open `https://controller.openstack.local:3000` in your browser.

- **User:** admin
- **Password:** `grep grafana_admin_password kolla/passwords.yml | awk '{print $2}'`

## Quick Reference: Horizon Navigation

| Task | Path |
|------|------|
| View instances | Project > Compute > Instances |
| Instance console | Instance detail > Console tab |
| View networks | Project > Network > Networks |
| Network topology diagram | Project > Network > Network Topology |
| View volumes | Project > Volumes > Volumes |
| View images | Project > Compute > Images |
| Manage flavors (admin) | Admin > Compute > Flavors |
| View hypervisors (admin) | Admin > Compute > Hypervisors |
| View all services (admin) | Admin > System Information |
| Provider networks (admin) | Admin > Network > Networks |
