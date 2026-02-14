# Launching Your First Instance

This guide walks through uploading an image, creating networks, launching an instance, and setting up external access with floating IPs.

## Prerequisites

Add the controller hostname to your Mac's `/etc/hosts`:

```bash
sudo bash -c 'echo "192.168.105.250 controller.openstack.local" >> /etc/hosts'
```

## OpenStack CLI Access

The OpenStack CLI lives inside `kolla_toolbox` on node1. Use this wrapper:

```bash
# Add to ~/.zshrc or ~/.bashrc
osc() {
  ssh -i ~/Documents/Projects/openstack/lima/openstack-lab \
    -o StrictHostKeyChecking=no root@192.168.105.10 \
    "docker exec kolla_toolbox bash -c '
      export OS_PROJECT_DOMAIN_NAME=Default
      export OS_USER_DOMAIN_NAME=Default
      export OS_PROJECT_NAME=admin
      export OS_USERNAME=admin
      export OS_PASSWORD=\$(cat /etc/kolla/passwords.yml 2>/dev/null | grep ^keystone_admin_password | cut -d\" \" -f2 || true)
      export OS_AUTH_URL=https://controller.openstack.local:5000
      export OS_INTERFACE=internal
      export OS_IDENTITY_API_VERSION=3
      export OS_REGION_NAME=RegionOne
      export OS_CACERT=/etc/ssl/certs/ca-certificates.crt
      openstack $*'" -- "$@"
}
```

Wait -- `kolla_toolbox` doesn't have `passwords.yml` mounted. Use the password directly instead:

```bash
osc() {
  ssh -i ~/Documents/Projects/openstack/lima/openstack-lab \
    -o StrictHostKeyChecking=no root@192.168.105.10 \
    "docker exec kolla_toolbox openstack \
      --os-project-domain-name Default \
      --os-user-domain-name Default \
      --os-project-name admin \
      --os-username admin \
      --os-password '$(grep OS_PASSWORD ~/Documents/Projects/openstack/kolla/admin-openrc.sh | cut -d"'" -f2)' \
      --os-auth-url https://controller.openstack.local:5000 \
      --os-interface internal \
      --os-identity-api-version 3 \
      --os-region-name RegionOne \
      --os-cacert /etc/ssl/certs/ca-certificates.crt \
      $@"
}
```

Test it:

```bash
osc service list
```

## Step 1: Upload a Test Image

Download and upload the CirrOS aarch64 image (tiny Linux VM for testing):

```bash
# Download CirrOS aarch64 to your Mac
curl -L -o /tmp/cirros-0.6.3-aarch64-disk.img \
  https://download.cirros-cloud.net/0.6.3/cirros-0.6.3-aarch64-disk.img

# Copy into kolla_toolbox and upload to Glance
ssh -i ~/Documents/Projects/openstack/lima/openstack-lab \
  -o StrictHostKeyChecking=no root@192.168.105.10 \
  'docker cp /dev/stdin kolla_toolbox:/tmp/cirros.img' \
  < /tmp/cirros-0.6.3-aarch64-disk.img

osc image create cirros \
  --file /tmp/cirros.img \
  --disk-format qcow2 \
  --container-format bare \
  --public \
  --property hw_architecture=aarch64
```

Verify:

```bash
osc image list
```

Create a flavor for the tiny CirrOS image:

```bash
osc flavor create m1.tiny --ram 512 --disk 1 --vcpus 1 --public
osc flavor create m1.small --ram 2048 --disk 20 --vcpus 1 --public
osc flavor create m1.medium --ram 4096 --disk 40 --vcpus 2 --public
```

## Step 2: Create a Tenant Network and Router

Create a private network for instances:

```bash
# Create the private network
osc network create demo-net

# Create a subnet (instances get IPs from this range)
osc subnet create demo-subnet \
  --network demo-net \
  --subnet-range 172.16.0.0/24 \
  --gateway 172.16.0.1 \
  --dns-nameserver 8.8.8.8

# Create a router
osc router create demo-router
```

Connect the router to the private network:

```bash
osc router add subnet demo-router demo-subnet
```

## Step 3: Launch a Test Instance

Create a security group that allows SSH and ICMP:

```bash
osc security group create demo-sg --description "Allow SSH and ICMP"
osc security group rule create demo-sg --protocol icmp --ingress
osc security group rule create demo-sg --protocol tcp --dst-port 22 --ingress
```

Generate a keypair for SSH access:

```bash
# Generate a key on your Mac
ssh-keygen -t ed25519 -f ~/.ssh/openstack-demo -N "" -C "openstack-demo"

# Upload the public key
osc keypair create demo-key --public-key "$(cat ~/.ssh/openstack-demo.pub)"
```

Launch the instance:

```bash
osc server create test-vm \
  --image cirros \
  --flavor m1.tiny \
  --network demo-net \
  --security-group demo-sg \
  --key-name demo-key
```

Wait for it to become ACTIVE:

```bash
# Watch the status (repeat until ACTIVE)
osc server list

# Check console log for boot output
osc console log show test-vm
```

You can also access the console via **Horizon** at `https://controller.openstack.local` (admin login).

## Step 4: Set Up External Network and Floating IPs

### Create the External Provider Network

This maps to the physical `lima2` interface on node1 via OVN's `physnet1`:

```bash
# Create the external network (flat, provider type)
osc network create external-net \
  --external \
  --provider-network-type flat \
  --provider-physical-network physnet1

# Create the external subnet
# Gateway 10.0.2.1 = Mac's bridge102 interface
# Allocation pool avoids conflict with the gateway
osc subnet create external-subnet \
  --network external-net \
  --subnet-range 10.0.2.0/24 \
  --gateway 10.0.2.1 \
  --allocation-pool start=10.0.2.100,end=10.0.2.200 \
  --no-dhcp
```

### Connect the Router to the External Network

```bash
osc router set demo-router --external-gateway external-net
```

This gives the router an external IP (visible with `osc router show demo-router`). Instances on `demo-net` can now reach the external network through NAT.

### Assign a Floating IP

```bash
# Allocate a floating IP from the external pool
osc floating ip create external-net

# Note the floating IP address from the output, then assign it
osc server add floating ip test-vm <FLOATING_IP>
```

### Test Connectivity from Your Mac

Your Mac already has `bridge102` on `10.0.2.0/24`, so floating IPs are directly reachable:

```bash
# Ping the instance
ping <FLOATING_IP>

# SSH into the instance (CirrOS default user: cirros)
ssh -i ~/.ssh/openstack-demo cirros@<FLOATING_IP>
```

> **Note:** CirrOS also accepts password login: `cirros` / `gocubsgo`

## Horizon Dashboard

Access at `https://controller.openstack.local` in your browser (accept the self-signed certificate warning).

- **Domain:** Default
- **User:** admin
- **Password:** see `kolla/admin-openrc.sh` (`grep OS_PASSWORD`)

From Horizon you can view instances, access the noVNC console, manage networks, and monitor resource usage.

## Grafana Monitoring

Access at `https://controller.openstack.local:3000`.

- **User:** admin
- **Password:** `grep grafana_admin_password kolla/passwords.yml | awk '{print $2}'`

## Troubleshooting

**Instance stuck in BUILD:**
```bash
osc server show test-vm -c fault
osc console log show test-vm
```

**Instance can't reach external network:**
```bash
# Check router has external gateway
osc router show demo-router -c external_gateway_info

# Check OVN flows
ssh -i lima/openstack-lab root@192.168.105.10 \
  'docker exec ovn_controller ovs-ofctl dump-flows br-int | head -20'
```

**Can't ping floating IP from Mac:**
```bash
# Verify Mac has the external bridge
ifconfig bridge102

# Check floating IP is assigned
osc floating ip list
osc server show test-vm -c addresses
```

**CirrOS won't boot (no console output):**
This is likely a QEMU/aarch64 issue. Check:
```bash
# Verify the image architecture
osc image show cirros -c properties

# Check nova-compute logs
ssh -i lima/openstack-lab root@192.168.105.11 \
  'docker logs nova_compute --tail 50'
```
