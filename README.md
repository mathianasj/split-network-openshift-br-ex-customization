# OpenShift with Split Network Interfaces (OVS Bridge Configuration)

This guide walks you through deploying OpenShift clusters with split network interfaces using an OVS bridge (br-ex) configuration. This setup is useful when you need to separate your management/primary network traffic from your OpenShift cluster network traffic.

**This guide covers two deployment scenarios:**
1. **Single Node OpenShift (SNO)**: All-in-one deployment for edge/development
2. **Multi-Node Cluster**: 3 control plane nodes + 2 worker nodes (standard HA deployment)

## Repository Files

This repository contains example configurations for both deployment types:

**Single Node OpenShift (SNO) Files:**
- `install-config.yaml` - OpenShift installation config for SNO
- `agent-config.yaml` - Agent-based installer config for 1 node
- `sno-node.yaml` - NMState network configuration for the single node
- `mc.yaml` - MachineConfig to apply network settings

**Multi-Node Cluster (3+2) Files:**
- `install-config-multinode.yaml` - OpenShift installation config for 3 masters + 2 workers
- `agent-config-multinode.yaml` - Agent-based installer config for all 5 nodes
- `master-01.yaml` - NMState network config for master-01 (create master-02, master-03 similarly)
- `worker-01.yaml` - NMState network config for worker-01 (create worker-02 similarly)
- `mc-multinode.yaml` - MachineConfig template for all nodes

## Overview

This deployment configures:
- **Primary Interface (ens18/enp1s0)**: Management network with static IP or DHCP, handles default routing
- **Secondary Interface (ens19/enp2s0)**: Bridged via OVS bridge (br-ex) for OpenShift cluster traffic
- **Agent-Based Installer**: Zero-touch installation method
- **OVS Bridge**: Separates cluster traffic for performance, security, or compliance

## Prerequisites

Before starting, ensure you have:
- OpenShift installer binary (`openshift-install`) version 4.12 or later
- Valid Red Hat pull secret from [cloud.redhat.com](https://cloud.redhat.com/openshift/install/pull-secret)
- SSH public key for node access
- Network information:
  - Base domain (e.g., `example.com`)
  - Cluster name (e.g., `ocpsno` for SNO or `ocp-cluster` for multi-node)
  - IP address for rendezvous IP (first master node)
  - MAC addresses for all network interfaces on all nodes
  - DNS server IP address
  - Gateway IP address
  - Machine network CIDR
- DNS records configured:
  - `api.<cluster-name>.<base-domain>` pointing to control plane nodes
  - `api-int.<cluster-name>.<base-domain>` pointing to control plane nodes
  - `*.apps.<cluster-name>.<base-domain>` wildcard for ingress (pointing to worker nodes or all nodes for SNO)

**For Single Node OpenShift (SNO):**
- 1 node with:
  - At least 2 network interfaces
  - Storage device for installation (e.g., `/dev/sda`)
  - Minimum 8 CPU cores, 32GB RAM, 120GB disk

**For Multi-Node Cluster (3 control plane + 2 workers):**
- 5 nodes total, each with:
  - At least 2 network interfaces
  - Storage device for installation (e.g., `/dev/sda`)
  - Control plane nodes: Minimum 4 CPU cores, 16GB RAM, 120GB disk each
  - Worker nodes: Minimum 2 CPU cores, 8GB RAM, 120GB disk each (adjust based on workload)

## Choosing Your Deployment Type

### Single Node OpenShift (SNO)
**Use when:**
- Edge computing or remote locations
- Development/testing environments
- Resource-constrained environments
- High availability is not critical

**Files to use:** `install-config.yaml`, `agent-config.yaml`, `sno-node.yaml`, `mc.yaml`

### Multi-Node Cluster (3 Masters + 2 Workers)
**Use when:**
- Production workloads requiring high availability
- Need for rolling updates without downtime
- Separation of control plane and workload
- Scaling workloads across multiple nodes

**Files to use:** `install-config-multinode.yaml`, `agent-config-multinode.yaml`, `master-01.yaml`, `worker-01.yaml`, `mc-multinode.yaml`

## Configuration Files Explained

### 1. install-config.yaml (SNO) / install-config-multinode.yaml (Multi-Node)

This is the main OpenShift installation configuration. The key differences between SNO and multi-node:

**Single Node OpenShift (install-config.yaml):**
```yaml
apiVersion: v1
baseDomain: example.com          # Your DNS base domain
metadata:
  name: ocpsno                    # Cluster name
networking:
  networkType: OVNKubernetes      # SDN type
  clusterNetwork:
  - cidr: 10.128.0.0/14          # Pod network CIDR
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16                # Service network CIDR
  machineNetwork:
  - cidr: 192.168.26.0/24        # Physical network CIDR
compute:
- name: worker
  replicas: 0                     # SNO has no separate workers
controlPlane:
  name: master
  replicas: 1                     # Single node = 1 master
platform:
  none: {}                        # Bare metal/platform-agnostic
bootstrapInPlace:
  installationDisk: /dev/sda      # SNO-specific: bootstrap in place
pullSecret: '<pull-secret-here>'  # Replace with your pull secret
sshKey: '<ssh-key-here>'          # Replace with your SSH public key
```

**Multi-Node Cluster (install-config-multinode.yaml):**
```yaml
apiVersion: v1
baseDomain: example.com          # Your DNS base domain
metadata:
  name: ocp-cluster               # Cluster name
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  machineNetwork:
  - cidr: 192.168.26.0/24
compute:
- name: worker
  replicas: 2                     # 2 worker nodes
controlPlane:
  name: master
  replicas: 3                     # 3 control plane nodes for HA
platform:
  none: {}                        # Bare metal/platform-agnostic
# NO bootstrapInPlace for multi-node
pullSecret: '<pull-secret-here>'
sshKey: '<ssh-key-here>'
```

**Key differences:**
- SNO: `replicas: 1` for control plane, `replicas: 0` for workers, includes `bootstrapInPlace`
- Multi-node: `replicas: 3` for control plane (HA), `replicas: 2+` for workers, NO `bootstrapInPlace`

### 2. agent-config.yaml (SNO) / agent-config-multinode.yaml (Multi-Node)

Agent-based installer configuration for host-specific settings:

- Defines the **rendezvous IP** (initial bootstrap IP - typically the first master node)
- Maps hostname to hardware (MAC addresses)
- Sets initial network configuration (before MachineConfig applies)
- Configures DNS and routing
- Specifies root device for installation
- Defines role for each host (master or worker)

**For SNO:** Contains configuration for 1 host
**For Multi-Node:** Contains configuration for all 5 hosts (3 masters + 2 workers)

**Key points for multi-node `agent-config-multinode.yaml`:**
- First master node uses the `rendezvousIP` (e.g., 192.168.26.101)
- Each host has a unique hostname, MAC addresses, and IP address
- Master nodes typically use IPs 101-103
- Worker nodes typically use IPs 201-202
- Each host entry includes both interfaces (enp1s0 for management, enp2s0 for cluster)
- Initial network config can use static IPs or DHCP (we use static for primary interface)

See `agent-config.yaml` for SNO example and `agent-config-multinode.yaml` for the full multi-node configuration with all 5 nodes.

### 3. Node NMState Configurations

NMState configuration files define the final network state with OVS bridge for each node. You'll need one file per node.

**Files:**
- `sno-node.yaml` - For single node OpenShift
- `master-01.yaml`, `master-02.yaml`, `master-03.yaml` - For control plane nodes
- `worker-01.yaml`, `worker-02.yaml` - For worker nodes

**Network configuration (same pattern for all nodes):**
- **ens18**: Primary interface with static IP, handles default route (management network)
- **ens19**: Secondary interface, member of OVS bridge (no IP directly on interface)
- **br-ex**: OVS bridge interface with DHCP for cluster traffic
- Routes traffic appropriately between interfaces

**Key differences per node:**
- Each node's NMState file has unique MAC addresses matching that node's hardware
- Each node's NMState file has unique static IP on ens18 (e.g., master-01: 192.168.26.101, worker-01: 192.168.26.201)
- The MAC address in `copy-mac-from: ens19` must match that node's ens19 interface

**Example structure (from master-01.yaml):**
```yaml
interfaces:
- name: ens18
  mac-address: 52:54:00:11:01:01  # Unique per node
  ipv4:
    address:
      - ip: 192.168.26.101         # Unique static IP per node
        prefix-length: 24
- name: ens19
  mac-address: 52:54:00:11:01:02  # Unique per node
  ipv4:
    enabled: false                 # No IP on physical interface
- name: br-ex
  type: ovs-bridge
  # Bridge configuration...
- name: br-ex
  type: ovs-interface
  copy-mac-from: ens19             # Use ens19's MAC
  ipv4:
    enabled: true
    dhcp: true                     # DHCP for cluster network
```

### 4. mc.yaml (SNO) / mc-multinode.yaml (Multi-Node)

MachineConfig files that apply the NMState configurations to nodes:

- Embeds each node's NMState YAML as base64 in a MachineConfig resource
- Applied to nodes via label selector (master or worker)
- Placed in `/etc/nmstate/openshift/` on each node with unique filename
- NetworkManager automatically applies the configuration on boot

**For SNO (mc.yaml):**
- Single MachineConfig for the one node
- Label: `machineconfiguration.openshift.io/role: master`
- Path: `/etc/nmstate/openshift/sno-node.yml`

**For Multi-Node (mc-multinode.yaml):**
- 5 separate MachineConfig resources in one file (YAML multi-doc with `---` separators)
- 3 MachineConfigs with label `role: master` (for master-01, master-02, master-03)
- 2 MachineConfigs with label `role: worker` (for worker-01, worker-02)
- Each has unique:
  - `metadata.name` (e.g., `10-br-ex-master-01`)
  - `path` (e.g., `/etc/nmstate/openshift/master-01.yml`)
  - `base64` content (from that node's specific NMState file)

**How nodes match their configuration:**
- The filename in the `path` field matches the node's hostname
- NetworkManager reads all files in `/etc/nmstate/openshift/` and applies configs where interface MACs match
- Each node only applies the config with matching MAC addresses

## Step-by-Step Installation Guide

**Choose your path:** Follow either the SNO or Multi-Node sections below based on your deployment type.

---

## Installation Path 1: Single Node OpenShift (SNO)

### Step 1: Customize Configuration Files (SNO)

1. **Edit install-config.yaml**:
   ```bash
   # Replace these values:
   # - baseDomain: your domain
   # - metadata.name: your cluster name (e.g., ocpsno)
   # - machineNetwork CIDR: your network range
   # - installationDisk: your target disk (check with 'lsblk')
   # - pullSecret: your Red Hat pull secret (single line, in quotes)
   # - sshKey: your SSH public key (single line, in quotes)
   # 
   # Ensure: controlPlane.replicas: 1, compute.replicas: 0, and bootstrapInPlace is present
   ```

2. **Edit agent-config.yaml**:
   ```bash
   # Replace these values:
   # - rendezvousIP: IP address for the single node
   # - hostname: your node's hostname (e.g., sno-node)
   # - macAddress: MAC addresses of your interfaces (find with 'ip link')
   # - ipv4 addresses: static IP or keep DHCP
   # - dns-resolver server: your DNS server IP
   # - routes next-hop-address: your gateway IP
   # - interface names (enp1s0, enp2s0): match your hardware
   ```

3. **Edit sno-node.yaml**:
   ```bash
   # Replace these values:
   # - interface names (ens18, ens19): match your actual interface names
   # - mac-address: match your hardware MAC addresses (both interfaces)
   # - ipv4 address on ens18: your node's static management IP
   # - dns-resolver server: your DNS server IP
   # - routes next-hop-address: your gateway IP
   # 
   # NOTE: Interface names may differ between discovery (enp*) and 
   # final OS (ens*). Verify with 'ip link' on the running system.
   ```

---

## Installation Path 2: Multi-Node Cluster (3 Masters + 2 Workers)

### Step 1: Customize Configuration Files (Multi-Node)

1. **Edit install-config-multinode.yaml**:
   ```bash
   # Replace these values:
   # - baseDomain: your domain
   # - metadata.name: your cluster name (e.g., ocp-cluster)
   # - machineNetwork CIDR: your network range
   # - pullSecret: your Red Hat pull secret (single line, in quotes)
   # - sshKey: your SSH public key (single line, in quotes)
   # 
   # Ensure: controlPlane.replicas: 3, compute.replicas: 2
   # Do NOT include bootstrapInPlace
   ```

2. **Edit agent-config-multinode.yaml**:
   ```bash
   # This file contains ALL 5 nodes. For each node, replace:
   # - hostname: unique per node (master-01, master-02, master-03, worker-01, worker-02)
   # - role: master or worker
   # - macAddress: unique MAC addresses for each node's interfaces
   # - ipv4 address: unique static IP per node:
   #   * master-01: 192.168.26.101
   #   * master-02: 192.168.26.102
   #   * master-03: 192.168.26.103
   #   * worker-01: 192.168.26.201
   #   * worker-02: 192.168.26.202
   # - rendezvousIP: IP of master-01 (192.168.26.101)
   # - dns-resolver server: your DNS server IP
   # - routes next-hop-address: your gateway IP
   # 
   # TIP: Get MAC addresses with 'ip link' on each node during discovery
   ```

3. **Create and edit NMState files for each node**:
   
   You need 5 NMState files total. Start by copying the examples:
   
   ```bash
   # Create files for remaining masters
   cp master-01.yaml master-02.yaml
   cp master-01.yaml master-03.yaml
   
   # Create file for second worker
   cp worker-01.yaml worker-02.yaml
   ```
   
   Then edit each file:
   
   **master-01.yaml:**
   ```bash
   # MAC addresses: match master-01 hardware
   # ens18 IP: 192.168.26.101
   ```
   
   **master-02.yaml:**
   ```bash
   # MAC addresses: match master-02 hardware (e.g., 52:54:00:11:02:01 and 52:54:00:11:02:02)
   # ens18 IP: 192.168.26.102
   ```
   
   **master-03.yaml:**
   ```bash
   # MAC addresses: match master-03 hardware (e.g., 52:54:00:11:03:01 and 52:54:00:11:03:02)
   # ens18 IP: 192.168.26.103
   ```
   
   **worker-01.yaml:**
   ```bash
   # MAC addresses: match worker-01 hardware (already in example)
   # ens18 IP: 192.168.26.201
   ```
   
   **worker-02.yaml:**
   ```bash
   # MAC addresses: match worker-02 hardware (e.g., 52:54:00:22:02:01 and 52:54:00:22:02:02)
   # ens18 IP: 192.168.26.202
   ```

---

## Common Steps (Both SNO and Multi-Node)

### Step 2: Create Cluster Manifests

Run the installer to generate manifests from your configs:

**For SNO:**
```bash
openshift-install agent create cluster-manifests
```

**For Multi-Node:**
```bash
# Rename the multi-node config files to the expected names
cp install-config-multinode.yaml install-config.yaml
cp agent-config-multinode.yaml agent-config.yaml

# Then run the command
openshift-install agent create cluster-manifests
```

This command:
- Validates your `install-config.yaml` and `agent-config.yaml`
- Generates additional manifests in the `cluster-manifests/` directory
- Prepares the configuration for image creation

**Note**: The `install-config.yaml` and `agent-config.yaml` will be consumed (deleted) during this process. Keep backups if needed.

### Step 3: Create the OpenShift Directory

Create a directory for custom manifests:

```bash
mkdir -p openshift
```

This directory will hold MachineConfigs that are included in the installation ISO.

### Step 4: Verify Node-Specific NMState Configurations

At this point, you should have your NMState YAML files ready:

**For SNO:**
- `sno-node.yaml` - Already created and edited in Step 1

**For Multi-Node:**
- `master-01.yaml`, `master-02.yaml`, `master-03.yaml` - Created and edited in Step 1
- `worker-01.yaml`, `worker-02.yaml` - Created and edited in Step 1

**Verification checklist for each file:**
```bash
# For each NMState YAML file, verify:
# 1. Interface names (ens18, ens19) match your hardware naming scheme
# 2. MAC addresses match that specific node's hardware
# 3. Static IP address is unique per node
# 4. DNS server IP is correct
# 5. Gateway IP is correct
# 6. The br-ex bridge references the correct interface (ens19)

# Quick validation (check for common issues):
grep "mac-address:" master-01.yaml  # Should show 2 unique MACs
grep "ip:" master-01.yaml           # Should show the correct static IP
```

**Important**: Verify interface names match your hardware. Interface names can vary:
- During discovery: `enp1s0`, `enp2s0` (PCI-based naming)
- After OS boot: `ens18`, `ens19` (biosdevname or other schemes)

Check your node's interface names and update the YAML files accordingly.

### Step 5: Create MachineConfig with Base64 Encoded NMState

Convert your NMState configurations to base64 and embed them in MachineConfigs:

**For SNO:**

```bash
# Generate base64 encoding
base64 -w0 sno-node.yaml > sno-node.yaml.b64  # Linux
# Or on macOS:
base64 -i sno-node.yaml -o sno-node.yaml.b64

# Display and copy the base64 string
cat sno-node.yaml.b64
```

Edit `openshift/mc.yaml` and replace `<base64-of-sno-node.yaml>` with the base64 string.

**For Multi-Node:**

```bash
# Generate base64 for all 5 nodes
base64 -w0 master-01.yaml > master-01.yaml.b64  # Linux
base64 -w0 master-02.yaml > master-02.yaml.b64
base64 -w0 master-03.yaml > master-03.yaml.b64
base64 -w0 worker-01.yaml > worker-01.yaml.b64
base64 -w0 worker-02.yaml > worker-02.yaml.b64

# On macOS, use -i flag:
# base64 -i master-01.yaml -o master-01.yaml.b64
# (repeat for each file)

# Quick script to generate all at once (Linux):
for node in master-01 master-02 master-03 worker-01 worker-02; do
  base64 -w0 ${node}.yaml > ${node}.yaml.b64
  echo "Generated ${node}.yaml.b64"
done
```

Now edit `openshift/mc-multinode.yaml` (or copy it to `openshift/mc.yaml`):

Replace each placeholder with the corresponding base64 content:
- `<base64-of-master-01.yaml>` → content of `master-01.yaml.b64`
- `<base64-of-master-02.yaml>` → content of `master-02.yaml.b64`
- `<base64-of-master-03.yaml>` → content of `master-03.yaml.b64`
- `<base64-of-worker-01.yaml>` → content of `worker-01.yaml.b64`
- `<base64-of-worker-02.yaml>` → content of `worker-02.yaml.b64`

**Quick replacement script:**
```bash
# Copy the template
cp mc-multinode.yaml openshift/mc.yaml

# Replace placeholders with actual base64 content (Linux/macOS)
sed -i.bak "s|<base64-of-master-01.yaml>|$(cat master-01.yaml.b64)|g" openshift/mc.yaml
sed -i.bak "s|<base64-of-master-02.yaml>|$(cat master-02.yaml.b64)|g" openshift/mc.yaml
sed -i.bak "s|<base64-of-master-03.yaml>|$(cat master-03.yaml.b64)|g" openshift/mc.yaml
sed -i.bak "s|<base64-of-worker-01.yaml>|$(cat worker-01.yaml.b64)|g" openshift/mc.yaml
sed -i.bak "s|<base64-of-worker-02.yaml>|$(cat worker-02.yaml.b64)|g" openshift/mc.yaml

# Verify the file looks correct
grep -c "source: data:text/plain" openshift/mc.yaml
# Should output: 5 (one for each node)
```

Save the completed `mc.yaml` file in the `openshift/` directory.

### Step 6: Create the Installation ISO

Generate the bootable ISO with all configurations:

```bash
openshift-install agent create image
```

This command:
- Incorporates all manifests from `cluster-manifests/` and `openshift/`
- Creates `agent.x86_64.iso` in the current directory
- Embeds your pull secret, SSH key, and network configurations
- Prepares the image for zero-touch deployment

### Step 7: Boot the ISO

Boot your target node(s) from the generated ISO:

1. **Copy the ISO** to your virtualization platform or physical media:
   ```bash
   # The same agent.x86_64.iso is used for ALL nodes
   # Copy to VM datastore, mount in virtualization platform, or burn to USB
   ```

2. **Boot the nodes** from the ISO:

   **For SNO:**
   - Boot the single node from the ISO
   
   **For Multi-Node:**
   - Boot all 5 nodes from the same ISO (can boot simultaneously)
   - Each node will identify itself based on MAC addresses in agent-config
   - The node with the rendezvous IP (master-01) becomes the bootstrap node initially

3. **Monitor the installation**:

   **For SNO:**
   - Initial network configuration from `agent-config.yaml` applies
   - After first boot, MachineConfig applies `sno-node.yaml` configuration
   - Installation takes 30-60 minutes

   **For Multi-Node:**
   - All nodes boot and connect to the rendezvous node (master-01)
   - Master nodes form the control plane first
   - Worker nodes join after control plane is ready
   - MachineConfigs apply to each node based on MAC address matching
   - Installation takes 45-90 minutes

4. **Access the bootstrap console** (optional):

   **For SNO:**
   ```bash
   # SSH to the node
   ssh core@192.168.26.111  # Use your rendezvous IP
   
   # Monitor installation progress
   journalctl -u agent-installer-controller.service -f
   ```

   **For Multi-Node:**
   ```bash
   # SSH to the rendezvous node (master-01)
   ssh core@192.168.26.101
   
   # Monitor installation progress
   journalctl -u agent-installer-controller.service -f
   
   # Check other nodes' status
   journalctl -u assisted-service.service -f
   
   # View agent logs
   tail -f /var/log/agent-installer/current.log
   ```

5. **Installation Progress Indicators**:

   **For SNO:**
   - Agent starts → Network configured → Ignition applied → Bootstrap begins → Control plane operators start → Installation complete

   **For Multi-Node:**
   - All agents start → Network configured on all nodes → Rendezvous node starts bootstrap → Masters join cluster → etcd quorum formed → Bootstrap complete → Workers join → Installation complete

## Post-Installation

### Verify Network Configuration

Once installation completes, verify the network setup on each node:

**For SNO:**
```bash
# SSH to the node
ssh core@192.168.26.111  # Your node's IP

# Check interface status
ip addr show

# Verify OVS bridge
sudo ovs-vsctl show

# Check routes
ip route show

# Verify NetworkManager state
nmcli connection show
```

**For Multi-Node:**
```bash
# Check each node (repeat for all 5 nodes)
# Masters: 192.168.26.101, 192.168.26.102, 192.168.26.103
# Workers: 192.168.26.201, 192.168.26.202

for node in 192.168.26.{101,102,103,201,202}; do
  echo "=== Checking node $node ==="
  ssh core@$node "hostname && ip addr show | grep -E 'ens18|ens19|br-ex'"
done
```

**Expected results on each node:**
- `ens18` with static IP (management network) - matches what you configured
- `ens19` without IP, attached to `br-ex`
- `br-ex` with IP from DHCP (cluster network)
- Default route via `ens18`

**Detailed verification on a single node:**
```bash
ssh core@<node-ip>

# Check OVS bridge configuration
sudo ovs-vsctl show
# Should show: br-ex bridge with ens19 as port

# Check routes
ip route show
# Default route should use ens18

# Check NMState applied correctly
cat /etc/nmstate/openshift/*.yml
nmcli connection show
```

### Access the Cluster

Retrieve the kubeconfig and verify cluster health:

```bash
# The installer creates auth files in your working directory
export KUBECONFIG=$(pwd)/auth/kubeconfig

# Verify cluster access
oc get nodes
# SNO: Should show 1 node with roles: control-plane,master,worker
# Multi-node: Should show 5 nodes (3 masters, 2 workers)

# Check cluster operators
oc get co
# All should show AVAILABLE=True

# Check cluster version
oc get clusterversion
```

**For Multi-Node, verify node roles:**
```bash
oc get nodes -o wide
# Should show:
# master-01   Ready   control-plane,master   ...
# master-02   Ready   control-plane,master   ...
# master-03   Ready   control-plane,master   ...
# worker-01   Ready   worker                 ...
# worker-02   Ready   worker                 ...
```

### Access the Web Console

```bash
# Get console URL
oc whoami --show-console

# Get kubeadmin password
cat auth/kubeadmin-password

# Or use this one-liner to open in browser (macOS):
open "$(oc whoami --show-console)"
```

Default credentials:
- Username: `kubeadmin`
- Password: (from `auth/kubeadmin-password` file)

## Troubleshooting

### Installation Hangs or Fails

1. **Check agent logs**:

   **For SNO:**
   ```bash
   ssh core@<rendezvous-ip>
   journalctl -u agent-installer-controller.service -f
   ```

   **For Multi-Node:**
   ```bash
   # Check rendezvous node (master-01)
   ssh core@192.168.26.101
   journalctl -u agent-installer-controller.service -f
   
   # Check assisted service logs
   journalctl -u assisted-service.service -f
   
   # View detailed agent logs
   tail -f /var/log/agent-installer/current.log
   
   # Check other nodes
   ssh core@192.168.26.102  # master-02
   journalctl -u agent-installer-controller.service -f
   ```

2. **Verify network connectivity**:
   - Can all nodes reach DNS?
   - Can nodes pull container images from registries?
   - Can nodes communicate with each other?
   - Check `/var/log/agent-installer/` for detailed logs

3. **Common Multi-Node Issues**:

   **Nodes not discovering each other:**
   ```bash
   # On rendezvous node, check if other nodes are visible
   ssh core@192.168.26.101
   curl http://localhost:8090/api/assisted-install/v2/clusters
   
   # Verify MAC addresses in agent-config match actual hardware
   # Check that all nodes booted from ISO
   ```

   **etcd quorum issues:**
   ```bash
   # Check etcd status on master nodes
   oc get pods -n openshift-etcd
   oc logs -n openshift-etcd <etcd-pod-name>
   
   # Verify all 3 master nodes are healthy
   oc get nodes -l node-role.kubernetes.io/master
   ```

   **Workers not joining:**
   ```bash
   # Check CSRs (Certificate Signing Requests)
   oc get csr
   
   # Approve pending CSRs if they're from your workers
   oc get csr -o name | xargs oc adm certificate approve
   
   # Check machine-config-daemon on workers
   ssh core@192.168.26.201
   journalctl -u machine-config-daemon.service -f
   ```

### Network Configuration Not Applied

1. **Check MachineConfig status**:
   ```bash
   oc get machineconfig
   oc get machineconfigpool
   
   # For multi-node, check both pools
   oc get mcp master
   oc get mcp worker
   
   # Look for UPDATING or DEGRADED status
   oc describe mcp master
   oc describe mcp worker
   ```

2. **Verify NMState files on each node**:

   **For SNO:**
   ```bash
   ssh core@<node-ip>
   ls -la /etc/nmstate/openshift/
   cat /etc/nmstate/openshift/sno-node.yml
   ```

   **For Multi-Node (check each node):**
   ```bash
   # Master nodes
   for node in 192.168.26.{101,102,103}; do
     echo "=== Node $node ==="
     ssh core@$node "ls -la /etc/nmstate/openshift/ && cat /etc/nmstate/openshift/*.yml | head -20"
   done
   
   # Worker nodes
   for node in 192.168.26.{201,202}; do
     echo "=== Node $node ==="
     ssh core@$node "ls -la /etc/nmstate/openshift/ && cat /etc/nmstate/openshift/*.yml | head -20"
   done
   ```

3. **Check NetworkManager logs**:
   ```bash
   ssh core@<node-ip>
   journalctl -u NetworkManager -f
   
   # Check for NMState errors
   journalctl | grep nmstate
   ```

4. **Verify MAC address matching**:
   ```bash
   # On each node, verify the MAC addresses in the NMState file match actual hardware
   ssh core@<node-ip>
   ip link show
   cat /etc/nmstate/openshift/*.yml | grep mac-address
   
   # They must match exactly for the config to be applied
   ```

### Interface Names Mismatch

If interfaces aren't found:
- Boot the ISO and check actual interface names: `ip link`
- Update `sno-node.yaml` with correct names
- Regenerate base64 encoding
- Update `mc.yaml` and recreate the image

### DNS or Routing Issues

1. **Verify DNS resolution**:
   ```bash
   dig api.<cluster-name>.<base-domain>
   dig *.apps.<cluster-name>.<base-domain>
   ```

2. **Check routes**:
   ```bash
   ip route show
   # Ensure default route points to correct gateway via correct interface
   ```

## Architecture Notes

### Why Split Interfaces?

This configuration is useful when:
- **Network Segmentation**: Separate management from cluster traffic
- **Performance**: Dedicate bandwidth to cluster workloads
- **Security**: Isolate cluster traffic on a separate VLAN or physical network
- **Compliance**: Meet requirements for network separation
- **Bandwidth**: Allow cluster traffic to use full bandwidth of dedicated interface
- **VLAN Isolation**: Put management on one VLAN, cluster traffic on another

### OVS Bridge (br-ex)

OpenShift uses OVS (Open vSwitch) for SDN:
- `br-ex` is the external bridge connecting to physical network
- Pods can reach external networks through this bridge
- The bridge inherits the MAC from the physical interface (ens19)
- DHCP on the bridge gets cluster network addressing
- All pod-to-external traffic flows through br-ex

### Network Flow

**Single Node:**
```
External Network
      |
   [ens18] -----> Management/SSH (Static IP, default route)
      |
   [Host]
      |
   [ens19] -----> [br-ex] -----> Cluster Traffic (DHCP)
                    |
              OVN/SDN to Pods
```

**Multi-Node Cluster:**
```
                    External Network
                           |
    +-----------------+----+----+-----------------+
    |                 |         |                 |
[Master-01]      [Master-02]  [Master-03]   [Worker-01] ...
    |                 |         |                 |
[ens18: .101]   [ens18: .102]  [ens18: .103]  [ens18: .201]
    |                 |         |                 |
  [ens19]→[br-ex]  [ens19]→[br-ex]  [ens19]→[br-ex]  [ens19]→[br-ex]
       |                |          |                |
       +----------------+----------+----------------+
                        |
                   OVN Overlay Network
                        |
                   Pod Network
                   (10.128.0.0/14)
```

### Multi-Node Architecture Details

**Control Plane (3 Masters):**
- etcd cluster runs across all 3 masters (requires quorum of 2)
- API servers run on all 3 masters (load balanced via DNS)
- Scheduler and controller managers run on all 3 masters (leader election)
- Can tolerate loss of 1 master node

**Worker Nodes (2 Workers):**
- Run application workloads
- Can scale horizontally by adding more workers
- MachineConfigPool manages configuration separately from masters

**Network Separation:**
- **Management Network (ens18)**: SSH access, monitoring, node-to-node coordination
- **Cluster Network (br-ex via ens19)**: Pod traffic, service traffic, ingress/egress
- **Overlay Network (OVN)**: Pod-to-pod communication across nodes

### Deployment Comparison

| Aspect | Single Node OpenShift (SNO) | Multi-Node (3+2) |
|--------|----------------------------|------------------|
| **High Availability** | No - single point of failure | Yes - can tolerate 1 master failure |
| **Resource Usage** | Minimal (1 node) | Higher (5 nodes minimum) |
| **Use Case** | Edge, dev/test, demos | Production, HA workloads |
| **Update Strategy** | Full downtime during updates | Rolling updates, no downtime |
| **etcd** | Single instance | 3-member cluster with quorum |
| **Scaling** | Cannot add nodes | Can add worker nodes |
| **Bootstrap** | Bootstrap-in-place | Temporary bootstrap node |
| **Installation Time** | 30-60 minutes | 45-90 minutes |

## Additional Resources

- [OpenShift Agent-Based Installer Documentation](https://docs.openshift.com/container-platform/latest/installing/installing_with_agent_based_installer/preparing-to-install-with-agent-based-installer.html)
- [NMState Configuration Examples](https://nmstate.io/examples.html)
- [Single Node OpenShift Requirements](https://docs.openshift.com/container-platform/latest/installing/installing_sno/install-sno-preparing-to-install-sno.html)

## Quick Reference Commands

### Configuration Phase

```bash
# STEP 1: Edit your config files (SNO or multi-node versions)

# STEP 2: Generate manifests
openshift-install agent create cluster-manifests

# STEP 3: Create openshift directory
mkdir -p openshift

# STEP 4: Create base64 for NMState configs
# Linux:
base64 -w0 sno-node.yaml > sno-node.yaml.b64  # SNO
# Or for multi-node:
for node in master-{01,02,03} worker-{01,02}; do
  base64 -w0 ${node}.yaml > ${node}.yaml.b64
done

# macOS:
base64 -i sno-node.yaml -o sno-node.yaml.b64  # SNO
# Or for multi-node:
for node in master-{01,02,03} worker-{01,02}; do
  base64 -i ${node}.yaml -o ${node}.yaml.b64
done

# STEP 5: Edit mc.yaml with base64 content (or use sed script)

# STEP 6: Create installation ISO
openshift-install agent create image
```

### Installation Phase

```bash
# Boot nodes from agent.x86_64.iso
# SNO: Boot 1 node
# Multi-node: Boot all 5 nodes

# Monitor installation on rendezvous node
ssh core@<rendezvous-ip>
journalctl -u agent-installer-controller.service -f
```

### Post-Installation

```bash
# Set kubeconfig
export KUBECONFIG=./auth/kubeconfig

# Check cluster
oc get nodes
oc get co  # All should be Available

# Get console access
oc whoami --show-console
cat auth/kubeadmin-password

# Verify network on nodes
ssh core@<node-ip> "ip addr show; sudo ovs-vsctl show"

# For multi-node: Check all nodes
for ip in 192.168.26.{101,102,103,201,202}; do
  echo "=== $ip ==="; ssh core@$ip hostname
done
```

### Troubleshooting

```bash
# Check MachineConfigs
oc get mc
oc get mcp

# View agent logs
ssh core@<node-ip>
tail -f /var/log/agent-installer/current.log

# Check network config applied
ssh core@<node-ip>
ls -la /etc/nmstate/openshift/
nmcli connection show

# Multi-node: Approve CSRs if workers pending
oc get csr
oc get csr -o name | xargs oc adm certificate approve
```

---

For questions or issues, please open an issue or reach out for a walkthrough session.
