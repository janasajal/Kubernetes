# OpenShift Virtualization Networking Guide
## Complete Web Console (GUI) Reference for On-Premises Production Environments

**Author:** Sajal Jana  
**Last Updated:** February 2026  
**Target Audience:** Beginners to Advanced Users  
**Environment:** On-Premises OpenShift with OpenShift Virtualization Operator

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Day-0 Networking Verification](#2-day-0-networking-verification)
   - 2.1 [Cluster Network Status](#21-cluster-network-status)
   - 2.2 [Node Network Configuration](#22-node-network-configuration)
   - 2.3 [CNI Plugin Verification](#23-cni-plugin-verification)
3. [Default VM Pod Networking](#3-default-vm-pod-networking)
   - 3.1 [Understanding Pod Network](#31-understanding-pod-network)
   - 3.2 [Creating VM with Default Networking](#32-creating-vm-with-default-networking)
   - 3.3 [Testing Pod Network Connectivity](#33-testing-pod-network-connectivity)
4. [Multus Secondary Networks](#4-multus-secondary-networks)
   - 4.1 [Multus CNI Overview](#41-multus-cni-overview)
   - 4.2 [Creating NetworkAttachmentDefinition](#42-creating-networkattachmentdefinition)
   - 4.3 [Verifying Multus Configuration](#43-verifying-multus-configuration)
5. [Bridge and VLAN-Backed Networks](#5-bridge-and-vlan-backed-networks)
   - 5.1 [Linux Bridge Configuration](#51-linux-bridge-configuration)
   - 5.2 [VLAN Tagging Setup](#52-vlan-tagging-setup)
   - 5.3 [Production Bridge Networks](#53-production-bridge-networks)
6. [Attaching Networks to VMs](#6-attaching-networks-to-vms)
   - 6.1 [Single Network Attachment](#61-single-network-attachment)
   - 6.2 [Multiple Network Interfaces](#62-multiple-network-interfaces)
   - 6.3 [Hot-Plug Network Interfaces](#63-hot-plug-network-interfaces)
7. [SR-IOV Networking](#7-sr-iov-networking)
   - 7.1 [SR-IOV Operator Installation](#71-sr-iov-operator-installation)
   - 7.2 [SR-IOV Network Node Policy](#72-sr-iov-network-node-policy)
   - 7.3 [SR-IOV Network Configuration](#73-sr-iov-network-configuration)
   - 7.4 [Attaching SR-IOV to VMs](#74-attaching-sr-iov-to-vms)
8. [Live Migration Networking](#8-live-migration-networking)
   - 8.1 [Dedicated Migration Network](#81-dedicated-migration-network)
   - 8.2 [Migration Network Policy](#82-migration-network-policy)
   - 8.3 [Testing Live Migration](#83-testing-live-migration)
9. [VM DNS and DHCP](#9-vm-dns-and-dhcp)
   - 9.1 [DNS Configuration Options](#91-dns-configuration-options)
   - 9.2 [DHCP Server Setup](#92-dhcp-server-setup)
   - 9.3 [Static IP Configuration](#93-static-ip-configuration)
10. [Network Security and Policies](#10-network-security-and-policies)
    - 10.1 [NetworkPolicy Basics](#101-networkpolicy-basics)
    - 10.2 [Creating Network Policies](#102-creating-network-policies)
    - 10.3 [VM Network Isolation](#103-vm-network-isolation)
11. [Day-2 Operations and Troubleshooting](#11-day-2-operations-and-troubleshooting)
    - 11.1 [Monitoring Network Performance](#111-monitoring-network-performance)
    - 11.2 [Common Network Issues](#112-common-network-issues)
    - 11.3 [Network Diagnostics Tools](#113-network-diagnostics-tools)
    - 11.4 [Backup and Recovery](#114-backup-and-recovery)
12. [Best Practices Summary](#12-best-practices-summary)
13. [References and Resources](#13-references-and-resources)

---

## 1. Introduction

This guide provides a complete, production-ready approach to configuring OpenShift Virtualization networking using only the OpenShift Web Console. All configurations are GUI-based with exact navigation paths and click-by-click instructions.

**Prerequisites:**
- OpenShift cluster (4.12+) installed and running
- OpenShift Virtualization operator installed
- Administrator access to Web Console
- Basic understanding of networking concepts (IP, VLAN, routing)

**What You'll Learn:**
- Configure VM networking from scratch
- Implement production-grade network architectures
- Troubleshoot common networking issues
- Apply security best practices

---

## 2. Day-0 Networking Verification

### 2.1 Cluster Network Status

**What it is:** Initial verification that OpenShift cluster networking is healthy and ready for virtualization workloads.

**Why it matters:**
- Ensures baseline cluster network connectivity before adding VM networking layers
- Identifies existing network configurations that may impact VM networking

**GUI-Based Configuration Steps:**

1. Log into OpenShift Web Console at `https://console-openshift-console.apps.<cluster-domain>`
2. Click **Networking** → **Networks** in left navigation
3. Verify the **Cluster Network** appears with status "Ready"
4. Note the following values:
   - Service Network CIDR (e.g., `172.30.0.0/16`)
   - Cluster Network CIDR (e.g., `10.128.0.0/14`)
5. Click **Networking** → **Services**
6. Verify `kubernetes` service shows status "Active"
7. Click **Compute** → **Nodes**
8. Verify all nodes show "Ready" status

**Validation in GUI:**
- All nodes display green checkmarks under Status
- No warning banners appear at top of console
- Cluster Network section shows "Ready" state

**Common Mistakes:**
- Skipping this verification leads to confusion between cluster issues and VM networking issues
- Not documenting existing CIDR ranges causes IP conflicts later

---

### 2.2 Node Network Configuration

**What it is:** Verification of physical network interfaces on nodes that will host VMs.

**Why it matters:**
- Identifies available NICs for VM networking
- Confirms node network readiness for bridge and SR-IOV configurations

**GUI-Based Configuration Steps:**

1. Navigate to **Compute** → **Nodes**
2. Click on any worker node name
3. Select the **Details** tab
4. Scroll to **Addresses** section
5. Note the Hostname and Internal IP
6. Click the **YAML** tab
7. Search for `status.addresses` section
8. Document IP addresses assigned
9. Click **Terminal** tab (if available in your OpenShift version)
10. Run `ip addr show` to see all interfaces (optional validation)
11. Repeat for 2-3 worker nodes to understand network topology

**Validation in GUI:**
- Each node shows at least one Internal IP address
- Nodes have consistent network configuration patterns
- No "NetworkNotReady" conditions in node status

**Common Mistakes:**
- Not checking all nodes assumes homogeneous network setup
- Ignoring additional NICs that could be used for dedicated VM networks

---

### 2.3 CNI Plugin Verification

**What it is:** Confirming the Container Network Interface plugin (typically OVN-Kubernetes) is operational.

**Why it matters:**
- CNI provides pod networking that VMs will use by default
- Multus requires a working primary CNI plugin

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **Networks**
2. Click on **default** network
3. Verify **Type** shows "OVNKubernetes" or "OpenShiftSDN"
4. Check **Cluster Network** CIDR is populated
5. Click **Workloads** → **Pods**
6. Filter namespace to **openshift-network-operator**
7. Verify all pods show "Running" status:
   - `network-operator-*`
8. Click on `network-operator-*` pod
9. Select **Logs** tab
10. Confirm no recent error messages (last 5 minutes)
11. Navigate to **Workloads** → **Pods**
12. Filter namespace to **openshift-ovn-kubernetes** or **openshift-sdn**
13. Verify all `ovnkube-*` or `sdn-*` pods are "Running"

**Validation in GUI:**
- All network operator pods show green "Running" status
- No CrashLoopBackOff or Error states
- Logs show successful network plugin initialization

**Common Mistakes:**
- Proceeding with degraded CNI plugin causes all VM networking to fail
- Not checking operator logs misses configuration warnings

---

## 3. Default VM Pod Networking

### 3.1 Understanding Pod Network

**What it is:** VMs run as pods and inherit standard Kubernetes pod networking by default.

**Why it matters:**
- Simplest networking model requiring zero configuration
- Good for testing and development environments
- Suitable for VMs that only need cluster-internal communication

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NetworkAttachmentDefinitions**
2. Select project/namespace **default**
3. Observe no additional network attachments yet exist
4. Navigate to **Networking** → **Services**
5. Note any existing services you want VMs to access
6. Navigate to **Virtualization** → **Overview**
7. Check that "VirtualMachine" shows available resources
8. Review default storage and compute capacity

**Validation in GUI:**
- NetworkAttachmentDefinitions list may be empty (this is normal)
- Virtualization dashboard shows healthy status
- No blocking errors or warnings

**Common Mistakes:**
- Expecting external network access with default pod networking (requires additional configuration)
- Not understanding pod networking limitations for production VMs

---

### 3.2 Creating VM with Default Networking

**What it is:** Launching a VM that uses only the cluster pod network without additional interfaces.

**Why it matters:**
- Validates basic OpenShift Virtualization functionality
- Establishes baseline before adding complex networking

**GUI-Based Configuration Steps:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create VirtualMachine** button
3. Select **From template** option
4. Choose a template (e.g., "Red Hat Enterprise Linux 9 VM")
5. Click **Customize VirtualMachine**
6. Fill in basic details:
   - Name: `test-vm-podnet`
   - Namespace: `default`
7. Under **Network Interfaces** section:
   - Observe default interface `default/pod-network`
   - Type shows "Pod networking (masquerade)"
8. Leave network settings as default
9. Adjust **Disk** and **CPU/Memory** if needed
10. Click **Create VirtualMachine**
11. Click **Start** when prompted
12. Wait for status to change to "Running"
13. Click on VM name `test-vm-podnet`
14. Select **Console** tab to access VM

**Validation in GUI:**
- VM status shows green "Running" indicator
- Console tab displays VM boot sequence
- No error events in the **Events** tab

**Common Mistakes:**
- Expecting VM to have external IP address (pod networking uses cluster internal IPs)
- Not waiting for VM to fully boot before testing connectivity

---

### 3.3 Testing Pod Network Connectivity

**What it is:** Verifying the VM can communicate within the cluster network.

**Why it matters:**
- Confirms CNI plugin integration with virtualization works
- Establishes connectivity baseline for troubleshooting

**GUI-Based Configuration Steps:**

1. In VM details page, click **Console** tab
2. Log into VM using template credentials
3. Run command: `ip addr show`
4. Note the `eth0` interface with IP in cluster network range (e.g., `10.128.x.x`)
5. Run command: `ping 172.30.0.1` (Kubernetes service IP)
6. Verify ping responses received
7. Navigate to **Networking** → **Services**
8. Find the `kubernetes` service ClusterIP
9. Return to VM console
10. Run: `curl -k https://<kubernetes-service-ip>:443` (should see certificate info)
11. From OpenShift console, go to **Workloads** → **Pods**
12. Find the pod for your VM (named `virt-launcher-test-vm-podnet-*`)
13. Click the pod name
14. Note the **Pod IP** address
15. Click **Terminal** tab
16. From another pod, verify you can reach this IP

**Validation in GUI:**
- VM receives IP in cluster network CIDR range
- Ping to Kubernetes API service succeeds
- VM pod appears in pod list with assigned IP

**Common Mistakes:**
- Testing external connectivity (not available with default masquerade mode)
- Forgetting that pod IPs change when VMs restart

---

## 4. Multus Secondary Networks

### 4.1 Multus CNI Overview

**What it is:** Multus is a meta-CNI plugin that enables attaching multiple network interfaces to pods (and VMs).

**Why it matters:**
- Required for production VM networking beyond basic pod network
- Enables direct connection to VLANs, physical networks, and SR-IOV

**GUI-Based Configuration Steps:**

1. Navigate to **Operators** → **Installed Operators**
2. Filter namespace: **All Projects**
3. Verify **OpenShift Virtualization** operator shows "Succeeded" status
4. Click on the operator
5. Scroll to **Provided APIs** section
6. Verify **NetworkAddonsConfig** is available
7. Click **NetworkAddonsConfig** link
8. Click on **cluster** instance
9. Select **YAML** tab
10. Verify `spec.multus` section shows `{}`
11. Check `status.conditions` shows multus as "Available: True"
12. Navigate to **Workloads** → **DaemonSets**
13. Filter namespace to **openshift-multus**
14. Verify `multus-admission-controller` shows desired number matching available
15. Verify `network-metrics-daemon` is running

**Validation in GUI:**
- NetworkAddonsConfig shows "Available" condition
- Multus DaemonSets are running on all nodes
- No error conditions in status

**Common Mistakes:**
- Assuming Multus is not installed (it comes with OpenShift Virtualization automatically)
- Not verifying admission controller is running causes network attachment failures

---

### 4.2 Creating NetworkAttachmentDefinition

**What it is:** A custom resource that defines how to attach a secondary network to pods/VMs.

**Why it matters:**
- Acts as template for network interface configuration
- Enables reusable network definitions across multiple VMs

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NetworkAttachmentDefinitions**
2. Select target namespace (e.g., `default`)
3. Click **Create NetworkAttachmentDefinition**
4. Choose **Form view** (or **YAML view** if comfortable)
5. Fill in the form:
   - **Name:** `linux-bridge-net1`
   - **Description:** "Bridge network for VM external connectivity"
6. Under **Network Type**, select **CNV Linux bridge**
7. Configure bridge settings:
   - **Bridge name:** `br1`
   - **VLAN:** leave empty (for now)
   - **Preserve default route:** No (unchecked)
8. Leave **MAC spoof check** enabled
9. Click **Create**
10. Wait for NAD to appear in list
11. Click on created NAD name
12. Verify **Status** shows as configured

**Alternative YAML View Steps:**

1. Click **Create NetworkAttachmentDefinition**
2. Select **YAML view**
3. Replace content with:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: linux-bridge-net1
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "linux-bridge-net1",
      "type": "cnv-bridge",
      "bridge": "br1",
      "macspoofchk": true
    }
```
4. Click **Create**

**Validation in GUI:**
- NAD appears in list with creation timestamp
- YAML view shows proper configuration structure
- No error messages displayed

**Common Mistakes:**
- Creating NAD without verifying bridge exists on nodes (causes attachment failures later)
- Using wrong CNI type (must use `cnv-bridge` or `cnv-tuning` for VM networks)

---

### 4.3 Verifying Multus Configuration

**What it is:** Confirming the NetworkAttachmentDefinition is ready to use with VMs.

**Why it matters:**
- Prevents VM creation failures due to misconfigured networks
- Identifies issues before impacting production workloads

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NetworkAttachmentDefinitions**
2. Verify your NAD (`linux-bridge-net1`) appears in list
3. Click on the NAD name
4. Review the **Details** tab:
   - Namespace is correct
   - Name is correct
   - Creation timestamp shows recent time
5. Click **YAML** tab
6. Verify `config` section contains valid JSON
7. Check for typos in bridge name
8. Navigate to **Compute** → **Nodes**
9. Click on a worker node
10. Select **Terminal** tab (if available)
11. Run: `ip link show br1` (should show "does not exist" - bridge will be created when VM starts)
12. Navigate to **Virtualization** → **VirtualMachines**
13. Click **Create** → **With YAML**
14. Use a test VM manifest with your NAD attached (see section 6.1)
15. Monitor VM creation for network attachment errors

**Validation in GUI:**
- NAD is visible in the dropdown when adding networks to VMs
- YAML configuration has no syntax errors
- Status section shows no error conditions

**Common Mistakes:**
- Not testing NAD attachment before production use leads to runtime failures
- Assuming bridge auto-creation on all nodes (may require NMState configuration)

---

## 5. Bridge and VLAN-Backed Networks

### 5.1 Linux Bridge Configuration

**What it is:** Creating a Linux bridge on OpenShift nodes that connects VM virtual interfaces to physical network interfaces.

**Why it matters:**
- Enables VMs to communicate on the same Layer 2 network as physical hosts
- Required for VMs to receive DHCP from external servers and be reachable from outside cluster

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
2. Click **Create NodeNetworkConfigurationPolicy**
3. Select **YAML view**
4. Enter the following configuration:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-eth1-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br1
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth1
```
5. Update `eth1` to match your node's physical interface name
6. Click **Create**
7. Wait for status to change to "Available"
8. Navigate to **Compute** → **Nodes**
9. Click on a worker node
10. Select **YAML** tab
11. Search for `status.addresses` to confirm node is still healthy
12. Navigate back to **NodeNetworkConfigurationPolicies**
13. Click on `br1-eth1-policy`
14. Check **Status** section shows "SuccessfullyConfigured: True" for all nodes

**Validation in GUI:**
- Policy status shows "Available" with no degraded conditions
- All matching nodes show "SuccessfullyConfigured"
- Nodes remain in "Ready" state after bridge creation
- NodeNetworkState resources show bridge interface

**Common Mistakes:**
- Using wrong physical interface name causes policy to fail
- Enabling STP on bridges can cause network loops in some environments
- Not using node selector creates bridges on master nodes (unnecessary and potentially problematic)

---

### 5.2 VLAN Tagging Setup

**What it is:** Configuring VLAN-tagged subinterfaces on bridges to segregate VM traffic by VLAN.

**Why it matters:**
- Enables network segmentation for security and organization
- Allows VMs to participate in existing enterprise VLAN architecture

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
2. Click **Create NodeNetworkConfigurationPolicy**
3. Select **YAML view**
4. Enter the following configuration:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-vlan100-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: eth1.100
        type: vlan
        state: up
        vlan:
          base-iface: eth1
          id: 100
      - name: br1-vlan100
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth1.100
```
5. Update `eth1` and VLAN ID `100` to match your environment
6. Click **Create**
7. Monitor the policy status until "Available"
8. Navigate to **Networking** → **NetworkAttachmentDefinitions**
9. Click **Create NetworkAttachmentDefinition**
10. Select **YAML view**
11. Enter:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: vlan100-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "vlan100-network",
      "type": "cnv-bridge",
      "bridge": "br1-vlan100",
      "vlan": 100,
      "macspoofchk": true
    }
```
12. Click **Create**
13. Verify NAD appears in list

**Validation in GUI:**
- NodeNetworkConfigurationPolicy shows "SuccessfullyConfigured" for all nodes
- NetworkAttachmentDefinition created without errors
- NodeNetworkState shows both VLAN interface and VLAN bridge

**Common Mistakes:**
- Mismatching VLAN IDs between bridge and NAD configuration
- Using same bridge name for multiple VLANs (each VLAN needs dedicated bridge)

---

### 5.3 Production Bridge Networks

**What it is:** Comprehensive bridge setup for production environments with multiple networks.

**Why it matters:**
- Provides template for enterprise-grade VM networking
- Demonstrates best practices for network organization

**GUI-Based Configuration Steps:**

1. Plan your network architecture:
   - Management network (VLAN 10)
   - Application network (VLAN 20)
   - Database network (VLAN 30)
2. Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
3. Click **Create NodeNetworkConfigurationPolicy**
4. Create management network policy:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: prod-mgmt-network
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: eth1.10
        type: vlan
        state: up
        vlan:
          base-iface: eth1
          id: 10
      - name: br-mgmt
        type: linux-bridge
        state: up
        ipv4:
          enabled: false
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth1.10
```
5. Click **Create**
6. Wait for "Available" status
7. Repeat steps 3-6 for application network (VLAN 20, br-app)
8. Repeat steps 3-6 for database network (VLAN 30, br-db)
9. Navigate to **Networking** → **NetworkAttachmentDefinitions**
10. Create NAD for each network (3 total)
11. Example for management:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: management-network
  namespace: default
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "management-network",
      "type": "cnv-bridge",
      "bridge": "br-mgmt",
      "vlan": 10,
      "macspoofchk": true
    }
```
12. Verify all three NADs created successfully
13. Document network naming convention for team

**Validation in GUI:**
- Three NodeNetworkConfigurationPolicies all show "Available"
- Three NetworkAttachmentDefinitions exist in target namespace
- NodeNetworkState shows all bridges and VLANs configured

**Common Mistakes:**
- Not documenting VLAN to purpose mapping causes confusion
- Using inconsistent naming conventions across bridges and NADs

---

## 6. Attaching Networks to VMs

### 6.1 Single Network Attachment

**What it is:** Adding one secondary network interface to a VM in addition to default pod network.

**Why it matters:**
- Most common production scenario for VMs needing external connectivity
- First step toward multi-homed VM configuration

**GUI-Based Configuration Steps:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create VirtualMachine**
3. Select **From template**
4. Choose template (e.g., "Red Hat Enterprise Linux 9 VM")
5. Click **Customize VirtualMachine**
6. Fill basic details:
   - Name: `vm-with-bridge`
   - Namespace: `default`
7. Scroll to **Network Interfaces** section
8. Click **Add Network Interface**
9. Configure new interface:
   - **Name:** `nic-bridge`
   - **Model:** `virtio`
   - **Network:** `default/linux-bridge-net1` (select from dropdown)
   - **Type:** `bridge`
   - **MAC Address:** leave empty (auto-generate)
10. Leave default pod network interface (usually first in list)
11. Review configuration:
    - Interface 0: Pod network (masquerade)
    - Interface 1: Bridge network
12. Click **Create VirtualMachine**
13. Click **Start** when VM is created
14. Wait for "Running" status
15. Click VM name to view details
16. Select **Network Interfaces** tab
17. Verify both interfaces show "Bound" status

**Validation in GUI:**
- VM shows two network interfaces in details view
- Both interfaces display "Bound" status with green checkmark
- No error events related to network attachment
- Console access confirms two network interfaces inside VM

**Common Mistakes:**
- Forgetting to keep default pod network causes loss of cluster connectivity
- Selecting wrong network type (must be "bridge" for secondary networks)

---

### 6.2 Multiple Network Interfaces

**What it is:** Configuring a VM with three or more network interfaces for different purposes.

**Why it matters:**
- Common for database servers, firewalls, and network appliances
- Enables traffic segregation at VM level

**GUI-Based Configuration Steps:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create VirtualMachine**
3. Select **From template** or **With YAML**
4. If using template, click **Customize VirtualMachine**
5. Name the VM: `vm-multi-nic`
6. Scroll to **Network Interfaces** section
7. Add first secondary interface:
   - Click **Add Network Interface**
   - Name: `mgmt-nic`
   - Model: `virtio`
   - Network: `default/management-network`
   - Type: `bridge`
8. Add second secondary interface:
   - Click **Add Network Interface**
   - Name: `app-nic`
   - Model: `virtio`
   - Network: `default/application-network`
   - Type: `bridge`
9. Add third secondary interface:
   - Click **Add Network Interface**
   - Name: `db-nic`
   - Model: `virtio`
   - Network: `default/database-network`
   - Type: `bridge`
10. Review total interfaces (should have 4: pod network + 3 secondary)
11. Click **Create VirtualMachine**
12. Start the VM
13. Navigate to VM details → **Network Interfaces** tab
14. Verify all 4 interfaces show proper configuration
15. Access VM console
16. Run `ip link show` inside VM
17. Confirm 4 network interfaces present (eth0, eth1, eth2, eth3)

**Validation in GUI:**
- Network Interfaces tab lists all interfaces with "Bound" status
- Each interface shows correct NetworkAttachmentDefinition
- VM console displays all interfaces with proper MAC addresses
- No network-related error events

**Common Mistakes:**
- Exceeding node resource limits with too many interfaces
- Not planning interface ordering causes confusion in guest OS configuration

---

### 6.3 Hot-Plug Network Interfaces

**What it is:** Adding or removing network interfaces while the VM is running without restart.

**Why it matters:**
- Enables network changes without downtime
- Critical for production environments requiring high availability

**GUI-Based Configuration Steps:**

**Adding Interface to Running VM:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on a running VM name
3. Verify VM status shows "Running"
4. Select **Network Interfaces** tab
5. Click **Add Network Interface** button
6. Configure new interface:
   - Name: `hotplug-nic`
   - Model: `virtio`
   - Network: `default/management-network`
   - Type: `bridge`
7. Click **Add**
8. Observe interface appears in list immediately
9. Check **Status** column shows "Bound"
10. Access VM console
11. Run `ip link show` to see new interface
12. Configure interface inside VM:
    - `sudo dhclient eth2` (adjust interface name)
13. Verify connectivity on new interface

**Removing Interface from Running VM:**

1. In VM details, select **Network Interfaces** tab
2. Find the interface to remove (e.g., `hotplug-nic`)
3. Click three-dot menu (⋮) on interface row
4. Select **Delete**
5. Confirm deletion in popup dialog
6. Verify interface disappears from list
7. Access VM console
8. Run `ip link show` to confirm interface removed
9. Check VM remains stable and running

**Validation in GUI:**
- New interface appears in list within seconds
- Status transitions to "Bound" without VM restart
- Removed interface disappears without affecting VM uptime
- Events tab shows successful interface operations

**Common Mistakes:**
- Assuming all network types support hot-plug (SR-IOV may have limitations)
- Not configuring interface inside guest OS after hot-plug add

---

## 7. SR-IOV Networking

### 7.1 SR-IOV Operator Installation

**What it is:** Installing the SR-IOV Network Operator to enable high-performance direct device passthrough networking.

**Why it matters:**
- Provides near-native network performance for VMs
- Required for low-latency, high-throughput workloads

**GUI-Based Configuration Steps:**

1. Navigate to **Operators** → **OperatorHub**
2. In search box, type: `SR-IOV Network Operator`
3. Click on **SR-IOV Network Operator** tile
4. Click **Install** button
5. On Install Operator page, configure:
   - **Update channel:** `stable`
   - **Installation mode:** `All namespaces on the cluster`
   - **Installed Namespace:** `openshift-sriov-network-operator`
   - **Update approval:** `Automatic`
6. Click **Install** button
7. Wait for installation (2-5 minutes)
8. Click **View Operator** when installation completes
9. Verify operator status shows "Succeeded"
10. Navigate to **Workloads** → **Pods**
11. Filter namespace: `openshift-sriov-network-operator`
12. Verify following pods are "Running":
    - `sriov-network-operator-*`
    - `sriov-network-config-daemon-*` (DaemonSet on all nodes)
13. Click on `sriov-network-config-daemon-*` pod
14. Check **Logs** tab for successful startup messages
15. Navigate to **Operators** → **Installed Operators**
16. Verify **SR-IOV Network Operator** shows "Succeeded" status

**Validation in GUI:**
- Operator appears in Installed Operators with green checkmark
- All operator pods running without crashes
- SriovNetworkNodeState custom resources appear for each node
- No error conditions in operator status

**Common Mistakes:**
- Installing in wrong namespace causes operator malfunction
- Not verifying config daemon runs on all worker nodes

---

### 7.2 SR-IOV Network Node Policy

**What it is:** Configuration that identifies SR-IOV capable NICs and creates virtual functions (VFs).

**Why it matters:**
- Tells operator which physical NICs to enable SR-IOV on
- Creates the VF pool that VMs will use

**GUI-Based Configuration Steps:**

**Prerequisites Check:**
1. Navigate to **Compute** → **Nodes**
2. Select a worker node with SR-IOV NIC
3. Click **Terminal** tab
4. Run: `lspci | grep Ethernet`
5. Note PCI address of SR-IOV capable NIC (e.g., `0000:03:00.0`)
6. Run: `cat /sys/class/net/<interface>/device/sriov_totalvfs`
7. Note maximum VFs supported (e.g., `63`)

**Creating Policy:**
1. Navigate to **Networking** → **SR-IOV Network Node Policies**
2. Click **Create SriovNetworkNodePolicy**
3. Select **Form view**
4. Configure policy:
   - **Name:** `sriov-network-policy`
   - **Resource Name:** `intelnics`
5. Under **Node Selector**, add:
   - Key: `feature.node.kubernetes.io/network-sriov.capable`
   - Value: `true`
6. Under **NIC Selector**:
   - **Vendor:** `8086` (for Intel, check your NIC vendor ID)
   - **Device ID:** leave empty to match all SR-IOV devices
   - **Root Devices:** enter PCI addresses (e.g., `0000:03:00.0`)
   - **PF Names:** leave empty (or specify interface name like `ens3f0`)
7. Configure settings:
   - **Number of VFs:** `10` (start conservative, max varies by NIC)
   - **Device Type:** `vfio-pci` (for VM passthrough)
   - **Link Type:** leave empty
8. Click **Create**
9. Navigate to **Networking** → **SR-IOV Network Node States**
10. Click on node name
11. Monitor **Status** section for `syncStatus: Succeeded`
12. **NOTE:** Node will reboot to apply SR-IOV configuration (wait 5-10 minutes)
13. After reboot, verify node returns to "Ready" status
14. Check SR-IOV Network Node State again
15. Verify `status.syncStatus` shows "Succeeded"

**Validation in GUI:**
- SriovNetworkNodePolicy created without errors
- SriovNetworkNodeState shows "Succeeded" sync status
- Node successfully reboots and rejoins cluster
- VFs visible in node state status

**Common Mistakes:**
- Not expecting node reboot causes panic when node becomes NotReady
- Setting VF count too high exhausts hardware resources
- Wrong vendor/device ID causes policy to match nothing

---

### 7.3 SR-IOV Network Configuration

**What it is:** Creating a NetworkAttachmentDefinition specifically for SR-IOV networks.

**Why it matters:**
- Makes SR-IOV VFs available as attachable networks in VMs
- Provides configuration template for SR-IOV interfaces

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **SR-IOV Networks**
2. Click **Create SriovNetwork**
3. Select **Form view**
4. Configure network:
   - **Name:** `sriov-network-1`
   - **Namespace:** `default`
   - **Description:** "SR-IOV network for high-performance VMs"
5. Under **Network Type**, select **SR-IOV**
6. Configure SR-IOV settings:
   - **Resource Name:** `intelnics` (match policy resource name)
   - **Network Namespace:** `default`
   - **VLAN:** `100` (if using VLAN tagging)
   - **Capabilities:** leave default
   - **IPAM:** `{}` (empty, VM will use DHCP or static config)
7. Click **Create**
8. Navigate to **Networking** → **NetworkAttachmentDefinitions**
9. Verify `sriov-network-1` appears in list
10. Click on `sriov-network-1`
11. Select **YAML** tab
12. Verify configuration includes:
    - `type: sriov`
    - `resourceName: intelnics`
13. Check annotations for automatic creation by SR-IOV operator

**Validation in GUI:**
- SriovNetwork appears in SR-IOV Networks list
- Corresponding NetworkAttachmentDefinition auto-created
- NAD configuration shows correct resource name
- No error events or conditions

**Common Mistakes:**
- Mismatching resourceName between policy and network
- Creating SR-IOV network before policy is fully synced
- Not specifying VLAN when physical network requires it

---

### 7.4 Attaching SR-IOV to VMs

**What it is:** Configuring a VM to use SR-IOV virtual functions for direct device passthrough.

**Why it matters:**
- Provides highest performance networking option
- Enables VMs to achieve near-native network throughput and latency

**GUI-Based Configuration Steps:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create VirtualMachine**
3. Select **From template**
4. Choose appropriate template
5. Click **Customize VirtualMachine**
6. Name VM: `vm-sriov-test`
7. Scroll to **Network Interfaces** section
8. Click **Add Network Interface**
9. Configure SR-IOV interface:
   - **Name:** `sriov-nic`
   - **Model:** `virtio` (SR-IOV uses its own driver, but keep virtio)
   - **Network:** `default/sriov-network-1`
   - **Type:** `SR-IOV`
   - **MAC Address:** leave empty
10. **Important:** Remove or keep default pod network based on needs
    - Keep pod network for cluster management
    - Remove for pure SR-IOV networking
11. Review interface configuration
12. Click **Create VirtualMachine**
13. Start the VM
14. Monitor VM startup carefully
15. Navigate to VM details → **Network Interfaces** tab
16. Verify SR-IOV interface shows "Bound" status
17. Click on **Events** tab
18. Check for successful VF allocation events
19. Access VM console
20. Run `lspci` inside VM to see SR-IOV device passed through

**Validation in GUI:**
- VM starts successfully with SR-IOV interface
- Network Interfaces tab shows SR-IOV type
- Events show successful VF allocation
- Inside VM, lspci shows actual network device

**Common Mistakes:**
- Not verifying VF availability before attaching (check node state)
- Attempting SR-IOV on nodes without capable hardware
- Forgetting that SR-IOV may limit live migration capabilities

---

## 8. Live Migration Networking

### 8.1 Dedicated Migration Network

**What it is:** Configuring a separate network specifically for VM live migration traffic.

**Why it matters:**
- Prevents migration traffic from saturating production networks
- Improves migration speed and reliability
- Required for optimal production cluster performance

**GUI-Based Configuration Steps:**

**Create Migration Network Infrastructure:**

1. Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
2. Click **Create NodeNetworkConfigurationPolicy**
3. Create dedicated migration bridge:
```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: migration-network-policy
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  desiredState:
    interfaces:
      - name: br-migration
        type: linux-bridge
        state: up
        ipv4:
          enabled: true
          dhcp: false
          address:
            - ip: 192.168.100.{{ node_id }}
              prefix-length: 24
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth2
```
4. Note: Replace `{{ node_id }}` with actual node IPs or use DHCP
5. Click **Create**
6. Wait for policy to apply to all nodes
7. Navigate to **Networking** → **NetworkAttachmentDefinitions**
8. Click **Create NetworkAttachmentDefinition**
9. Create migration network NAD:
```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: migration-network
  namespace: openshift-cnv
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "migration-network",
      "type": "cnv-bridge",
      "bridge": "br-migration",
      "macspoofchk": true
    }
```
10. Click **Create**

**Configure HyperConverged CR for Migration:**

11. Navigate to **Operators** → **Installed Operators**
12. Click **OpenShift Virtualization**
13. Click **HyperConverged** tab
14. Click on **kubevirt-hyperconverged** instance
15. Select **YAML** tab
16. Find `spec:` section
17. Add or modify `liveMigrationConfig`:
```yaml
spec:
  liveMigrationConfig:
    network: openshift-cnv/migration-network
    completionTimeoutPerGiB: 800
    progressTimeout: 150
```
18. Click **Save**
19. Wait for configuration to reconcile
20. Navigate to **Workloads** → **Pods**
21. Filter namespace: `openshift-cnv`
22. Verify `virt-handler-*` pods restart (rolling restart)

**Validation in GUI:**
- NodeNetworkConfigurationPolicy shows "Available" status
- Migration network NAD exists in openshift-cnv namespace
- HyperConverged CR shows updated liveMigrationConfig
- All virt-handler pods successfully restart

**Common Mistakes:**
- Using production network for migration causes performance degradation
- Not having sufficient bandwidth on migration network
- Forgetting to create NAD in openshift-cnv namespace (where virt-launcher runs)

---

### 8.2 Migration Network Policy

**What it is:** Configuring network policies to isolate and secure migration traffic.

**Why it matters:**
- Prevents unauthorized access to migration network
- Ensures migration traffic is not exposed to untrusted networks

**GUI-Based Configuration Steps:**

1. Navigate to **Networking** → **NetworkPolicies**
2. Select namespace: `openshift-cnv`
3. Click **Create NetworkPolicy**
4. Select **YAML view**
5. Create restrictive policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-migration-traffic
  namespace: openshift-cnv
spec:
  podSelector:
    matchLabels:
      kubevirt.io: virt-launcher
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: openshift-cnv
        podSelector:
          matchLabels:
            kubevirt.io: virt-launcher
      ports:
        - protocol: TCP
          port: 49152
          endPort: 49216
  egress:
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: openshift-cnv
        podSelector:
          matchLabels:
            kubevirt.io: virt-launcher
      ports:
        - protocol: TCP
          port: 49152
          endPort: 49216
```
6. Click **Create**
7. Verify policy appears in list
8. Navigate to policy details
9. Check that affected pods show in **Pod Selector** section
10. Navigate to **Workloads** → **Pods**
11. Filter to `openshift-cnv` namespace
12. Click on a `virt-launcher-*` pod
13. Select **Network** tab (if available in your OpenShift version)
14. Verify policy is applied

**Validation in GUI:**
- NetworkPolicy created successfully
- Policy shows matching pod count
- Migration still functions after policy application
- No network-related error events

**Common Mistakes:**
- Creating overly restrictive policy that blocks legitimate migration
- Not testing migration after applying policy
- Forgetting that NetworkPolicies only work with compatible CNI plugins

---

### 8.3 Testing Live Migration

**What it is:** Performing and validating a VM live migration using the dedicated migration network.

**Why it matters:**
- Confirms migration network is properly configured
- Establishes baseline for migration performance monitoring

**GUI-Based Configuration Steps:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Select a running VM (not using SR-IOV)
3. Click on VM name to view details
4. Note current node in **Details** tab under "Node"
5. Click **Actions** dropdown (top right)
6. Select **Migrate**
7. Review migration dialog:
   - Confirm source node
   - Destination node will be auto-selected
8. Click **Migrate** button
9. Watch status change from "Running" to "Migrating"
10. Monitor **Events** tab for migration progress:
    - "Migration started"
    - "Migration in progress"
    - "Migration completed"
11. Refresh page to see updated node location
12. Verify VM is now on different node
13. Click **Console** tab
14. Verify VM remained accessible during migration
15. Check application inside VM for any interruption
16. Navigate to **Observe** → **Dashboards**
17. Select "Kubernetes / Compute Resources / Namespace (Pods)"
18. Change namespace to VM's namespace
19. Look for migration-related network traffic spike
20. Navigate to **Networking** → **NetworkAttachmentDefinitions**
21. Click on `migration-network`
22. Review if any errors logged during migration

**Performance Validation:**

1. Navigate to **Virtualization** → **Overview**
2. Click on **Metrics** tab
3. Review "Top VMs by Migration Duration"
4. Compare with expected migration times based on VM memory size
5. Migration time should be approximately:
   - (VM Memory Size in GB × 8) / (Network Bandwidth in Gbps) seconds
   - Example: 8GB VM on 10Gbps network ≈ 6-8 seconds

**Validation in GUI:**
- VM successfully migrates to different node
- VM remains "Running" throughout migration
- Console access maintained
- Events show successful migration completion
- No application downtime inside VM

**Common Mistakes:**
- Testing migration during high cluster load gives inaccurate results
- Not testing migration under various VM memory sizes
- Assuming first successful migration means all future ones will succeed

---

## 9. VM DNS and DHCP

### 9.1 DNS Configuration Options

**What it is:** Methods for providing DNS resolution to VMs on secondary networks.

**Why it matters:**
- VMs need DNS for name resolution
- Different approaches for different network types
- Critical for application functionality

**GUI-Based Configuration Steps:**

**Option 1: Using Cluster DNS (for pod network):**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on VM with pod network
3. Access **Console** tab
4. Log into VM
5. Check DNS configuration: `cat /etc/resolv.conf`
6. Should show cluster DNS IP (typically `172.30.0.10` or similar)
7. Test: `nslookup kubernetes.default.svc.cluster.local`
8. Should resolve to Kubernetes API service IP

**Option 2: External DHCP Server (for bridge networks):**

1. Ensure external DHCP server is configured on physical network
2. DHCP server should provide:
   - IP address
   - Subnet mask
   - Default gateway
   - DNS servers
3. Navigate to **Virtualization** → **VirtualMachines**
4. Create/edit VM with bridge network
5. Ensure VM configuration allows DHCP:
   - Interface type: `bridge`
   - No static IP configuration
6. Start VM
7. Access console
8. Verify IP obtained: `ip addr show`
9. Check DNS from DHCP: `cat /etc/resolv.conf`
10. Test external resolution: `nslookup google.com`

**Option 3: Static Configuration in VM:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on VM name
3. Access **Console** tab
4. Edit network configuration inside VM:
   - For RHEL/CentOS: Edit `/etc/sysconfig/network-scripts/ifcfg-eth1`
   - For Ubuntu: Edit `/etc/netplan/*.yaml`
5. Add DNS servers:
   - RHEL: `DNS1=8.8.8.8` and `DNS2=8.8.4.4`
   - Ubuntu: Under `nameservers:` add `addresses: [8.8.8.8, 8.8.4.4]`
6. Apply configuration:
   - RHEL: `sudo systemctl restart NetworkManager`
   - Ubuntu: `sudo netplan apply`
7. Verify: `cat /etc/resolv.conf`
8. Test: `nslookup example.com`

**Option 4: Using ConfigMap for Cloud-Init:**

1. Navigate to **Workloads** → **ConfigMaps**
2. Select VM's namespace
3. Click **Create ConfigMap**
4. Name: `vm-network-config`
5. Add data key: `networkdata`
6. Add cloud-init network configuration:
```yaml
version: 2
ethernets:
  eth1:
    dhcp4: false
    addresses:
      - 192.168.1.100/24
    gateway4: 192.168.1.1
    nameservers:
      addresses:
        - 8.8.8.8
        - 8.8.4.4
```
7. Click **Create**
8. Navigate to **Virtualization** → **VirtualMachines**
9. Edit VM YAML to reference ConfigMap in cloud-init section
10. Restart VM to apply configuration

**Validation in GUI:**
- VM console shows proper DNS configuration in `/etc/resolv.conf`
- DNS queries resolve correctly (test with nslookup/dig)
- Application functionality dependent on DNS works
- No DNS timeout errors in VM logs

**Common Mistakes:**
- Expecting pod network DNS to work on bridge interfaces (requires external DNS)
- Not configuring DNS when using static IPs
- Using cluster DNS IPs from bridge networks (unreachable)

---

### 9.2 DHCP Server Setup

**What it is:** Deploying a DHCP server within OpenShift to serve bridge network VMs.

**Why it matters:**
- Provides automated IP addressing for bridge-connected VMs
- Eliminates need for external DHCP infrastructure
- Enables self-contained VM networking solution

**GUI-Based Configuration Steps:**

**Option 1: External DHCP (Recommended for Production):**

This section assumes external DHCP exists. Ensure:
1. DHCP server is on the same L2 network as bridge
2. DHCP scope configured for VM subnet
3. DHCP server has sufficient leases available
4. Verify by creating test VM and checking IP assignment

**Option 2: Containerized DHCP Server (Lab/Dev):**

1. Navigate to **Workloads** → **Deployments**
2. Select target namespace (e.g., `default`)
3. Click **Create Deployment**
4. Select **YAML view**
5. Create DHCP server deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dhcp-server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dhcp-server
  template:
    metadata:
      labels:
        app: dhcp-server
    spec:
      containers:
      - name: dhcp-server
        image: networkboot/dhcpd
        command:
          - /usr/sbin/dhcpd
          - -f
          - -d
          - --no-pid
        volumeMounts:
        - name: dhcp-config
          mountPath: /data
      volumes:
      - name: dhcp-config
        configMap:
          name: dhcp-server-config
```
6. Click **Create**
7. Navigate to **Workloads** → **ConfigMaps**
8. Click **Create ConfigMap**
9. Name: `dhcp-server-config`
10. Add data key: `dhcpd.conf`
11. Add DHCP configuration:
```
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  default-lease-time 600;
  max-lease-time 7200;
}
```
12. Click **Create**
13. Navigate to **Networking** → **NetworkAttachmentDefinitions**
14. Edit bridge network NAD to attach DHCP pod
15. Ensure DHCP server pod uses same bridge as VMs
16. Navigate to **Workloads** → **Pods**
17. Verify `dhcp-server-*` pod is "Running"
18. Click on pod and check logs for DHCP server startup

**Note:** This containerized approach is NOT recommended for production. Use enterprise DHCP infrastructure instead.

**Validation in GUI:**
- DHCP server pod runs successfully
- Pod logs show DHCP server listening
- Test VM receives IP from configured range
- VM can communicate using DHCP-assigned IP

**Common Mistakes:**
- Running containerized DHCP in production (use external infrastructure)
- DHCP server not on correct network segment
- IP range conflicts with existing allocations
- Not configuring gateway and DNS options in DHCP

---

### 9.3 Static IP Configuration

**What it is:** Manually configuring fixed IP addresses for VMs using cloud-init or manual configuration.

**Why it matters:**
- Required for VMs needing predictable IP addresses
- Common for database servers, load balancers, and infrastructure VMs
- Bypasses DHCP dependency

**GUI-Based Configuration Steps:**

**Method 1: Cloud-Init with Static IP:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click **Create VirtualMachine**
3. Select **From template**
4. Choose template that supports cloud-init
5. Click **Customize VirtualMachine**
6. Fill in basic details
7. Scroll to **Scripts** section
8. Click **Edit** under "Cloud-init"
9. Select **Custom script** option
10. Enter network configuration:
```yaml
#cloud-config
network:
  version: 2
  ethernets:
    eth1:
      dhcp4: false
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```
11. Click **Apply**
12. Complete VM creation
13. Start VM
14. Access console
15. Verify static IP: `ip addr show eth1`
16. Test connectivity: `ping 192.168.1.1`

**Method 2: Manual Configuration in Running VM:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on running VM
3. Access **Console** tab
4. Log into VM
5. For RHEL/CentOS 8/9:
   ```bash
   sudo nmcli con add type ethernet ifname eth1 con-name static-eth1
   sudo nmcli con mod static-eth1 ipv4.addresses 192.168.1.50/24
   sudo nmcli con mod static-eth1 ipv4.gateway 192.168.1.1
   sudo nmcli con mod static-eth1 ipv4.dns "8.8.8.8 8.8.4.4"
   sudo nmcli con mod static-eth1 ipv4.method manual
   sudo nmcli con up static-eth1
   ```
6. For Ubuntu:
   ```bash
   sudo vi /etc/netplan/01-netcfg.yaml
   ```
   Add:
   ```yaml
   network:
     version: 2
     ethernets:
       eth1:
         addresses:
           - 192.168.1.50/24
         routes:
           - to: default
             via: 192.168.1.1
         nameservers:
           addresses: [8.8.8.8, 8.8.4.4]
   ```
   Apply: `sudo netplan apply`
7. Verify configuration: `ip addr show`
8. Test routing: `ip route show`
9. Test DNS: `nslookup google.com`

**Method 3: Using Dedicated ConfigMap:**

1. Navigate to **Workloads** → **ConfigMaps**
2. Select VM's namespace
3. Click **Create ConfigMap**
4. Name: `vm-static-network`
5. Add key: `network-config`
6. Add value with network configuration
7. Click **Create**
8. Navigate to VM YAML
9. Add ConfigMap reference to cloudInitConfigMap
10. Restart VM to apply

**Validation in GUI:**
- VM console shows configured static IP on interface
- IP persists across VM reboots
- Routing table shows correct default gateway
- DNS resolution works with configured nameservers
- No DHCP requests in network logs

**Common Mistakes:**
- Configuring static IP that conflicts with DHCP range
- Forgetting to configure gateway causes routing failures
- Not adding DNS servers results in name resolution failures
- Using pod network interface name (eth0) instead of bridge interface (eth1)

---

## 10. Network Security and Policies

### 10.1 NetworkPolicy Basics

**What it is:** Kubernetes NetworkPolicy resources that control traffic flow between pods (including VMs).

**Why it matters:**
- Implements micro-segmentation for VM workloads
- Enforces network-level security controls
- Required for compliance and zero-trust architectures

**GUI-Based Configuration Steps:**

**Understanding Default Behavior:**

1. Navigate to **Networking** → **NetworkPolicies**
2. Select a namespace with VMs
3. Observe if any policies exist
4. **Note:** Without policies, all traffic is allowed (default allow)
5. Navigate to **Virtualization** → **VirtualMachines**
6. Click on any VM
7. Select **YAML** tab
8. Note labels under `metadata.labels` (e.g., `app: database`)
9. These labels will be used in policy selectors

**Checking Policy Effect:**

1. Navigate to **Networking** → **NetworkPolicies**
2. If policies exist, click on policy name
3. Review **Pod Selector** section
4. Check which VMs are affected by labels
5. Review **Ingress Rules** and **Egress Rules**
6. Navigate to affected VM
7. Click **Networking** tab (if available)
8. View which policies apply

**Policy Types Overview:**

- **Ingress policies:** Control incoming traffic to VMs
- **Egress policies:** Control outgoing traffic from VMs
- **Combined policies:** Control both directions

**Validation in GUI:**
- NetworkPolicies list shows existing policies
- VM labels visible in YAML
- Policy pod selector matches VM labels
- No error conditions on policies

**Common Mistakes:**
- Creating policies before understanding default allow behavior
- Not labeling VMs consistently makes policies ineffective
- Forgetting that empty policy blocks all traffic

---

### 10.2 Creating Network Policies

**What it is:** Step-by-step creation of NetworkPolicies to control VM traffic.

**Why it matters:**
- Practical implementation of security controls
- Demonstrates common policy patterns
- Prevents common misconfigurations

**GUI-Based Configuration Steps:**

**Scenario 1: Restrict Database VM to Application VM Only:**

1. Navigate to **Networking** → **NetworkPolicies**
2. Select namespace containing VMs
3. Click **Create NetworkPolicy**
4. Select **YAML view**
5. Create restrictive ingress policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-from-app-only
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: application
      ports:
        - protocol: TCP
          port: 5432
```
6. Click **Create**
7. Verify policy appears in list
8. Click policy name
9. Review **Pod Selector** shows matching database VMs
10. Check **Ingress Rules** shows only application VMs allowed

**Scenario 2: Egress Control for Web VMs:**

1. Click **Create NetworkPolicy**
2. Create egress policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-egress-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Egress
  egress:
    - to:
      - podSelector:
          matchLabels:
            app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
      - namespaceSelector:
          matchLabels:
            name: kube-system
      ports:
        - protocol: UDP
          port: 53
```
3. Click **Create**
4. This allows web VMs to reach database and DNS only

**Scenario 3: Deny All, Allow Specific:**

1. Click **Create NetworkPolicy**
2. Create deny-all baseline:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```
3. Click **Create**
4. This blocks all traffic to/from all VMs
5. Create specific allow policies afterward
6. Example allow SSH from bastion:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ssh-from-bastion
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: bastion
      ports:
        - protocol: TCP
          port: 22
```
7. Click **Create**

**Validation in GUI:**
- All policies show "Active" status
- Pod selector match count updates automatically
- Test connectivity from allowed sources succeeds
- Test connectivity from denied sources fails
- No unexpected traffic appears in VM logs

**Common Mistakes:**
- Creating deny-all without subsequent allow policies locks out all access
- Not testing policies immediately after creation
- Assuming policies take effect instantly (may have slight delay)
- Overlapping policies can cause confusion (last created may not override)

---

### 10.3 VM Network Isolation

**What it is:** Complete network segregation of VMs from other workloads and VMs.

**Why it matters:**
- Implements zero-trust networking principles
- Required for multi-tenant environments
- Prevents lateral movement in security breaches

**GUI-Based Configuration Steps:**

**Complete Namespace Isolation:**

1. Navigate to **Administration** → **Namespaces**
2. Click **Create Namespace**
3. Name: `isolated-vms`
4. Add label: `isolation=strict`
5. Click **Create**
6. Navigate to **Networking** → **NetworkPolicies**
7. Select namespace: `isolated-vms`
8. Click **Create NetworkPolicy**
9. Create complete isolation policy:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: isolated-vms
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []
  egress:
    - to:
      - namespaceSelector:
          matchLabels:
            name: kube-system
      ports:
        - protocol: UDP
          port: 53
    - to:
      - podSelector: {}
```
10. Click **Create**
11. This allows:
    - DNS queries to kube-system
    - Communication within namespace only
    - Blocks all external communication

**VM-to-VM Isolation Within Namespace:**

1. In same namespace, create per-VM isolation
2. Label each VM uniquely: `vm-id: vm-001`, `vm-id: vm-002`
3. Click **Create NetworkPolicy**
4. Create policy for VM-001:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-vm-001
  namespace: isolated-vms
spec:
  podSelector:
    matchLabels:
      vm-id: vm-001
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            access-vm-001: "true"
  egress:
    - to:
      - podSelector:
          matchLabels:
            vm-id: vm-001-allowed
```
5. Repeat for each VM requiring isolation
6. Only label other VMs with `access-vm-001: "true"` if they need access

**Temporary Access Patterns:**

1. For break-glass access, create temporary policy
2. Click **Create NetworkPolicy**
3. Name: `temp-admin-access-<date>`
4. Add time-bound annotation: `delete-after: 2026-02-20`
5. Create allow-all from specific admin VM
6. Manually delete policy after maintenance

**Validation in GUI:**
- VMs in isolated namespace cannot reach other namespaces
- VMs with isolation policies cannot reach each other
- Only specifically allowed traffic succeeds
- Attempting connections shows "connection refused" or timeout
- Checking pod logs shows no unexpected communication

**Common Mistakes:**
- Forgetting to allow DNS causes all name resolution to fail
- Complete isolation prevents legitimate monitoring and management
- Not documenting isolation requirements leads to troubleshooting confusion
- Overly complex policies become unmaintainable

---

## 11. Day-2 Operations and Troubleshooting

### 11.1 Monitoring Network Performance

**What it is:** Using OpenShift console to monitor VM network health and performance metrics.

**Why it matters:**
- Identifies performance bottlenecks proactively
- Provides baseline for capacity planning
- Essential for SLA compliance

**GUI-Based Configuration Steps:**

**Accessing Virtualization Metrics:**

1. Navigate to **Virtualization** → **Overview**
2. Click **Metrics** tab
3. Review available metrics:
   - VM Network Traffic (bytes in/out)
   - Network Errors
   - Migration Performance
4. Adjust time range using dropdown (Last 1h, 6h, 24h, etc.)
5. Click on specific metric to expand
6. Export to CSV if needed for reporting

**Per-VM Network Monitoring:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on specific VM name
3. Select **Metrics** tab
4. Review VM-specific network metrics:
   - Network Receive (Bytes)
   - Network Transmit (Bytes)
   - Network Errors
5. Compare against baseline or SLA thresholds
6. Adjust time window to identify patterns
7. Click **Diagnostics** tab
8. Review recent events for network issues

**Using Prometheus Queries:**

1. Navigate to **Observe** → **Metrics**
2. Click **Metrics** in left navigation
3. Enter PromQL queries for detailed analysis:
   - `kubevirt_vmi_network_receive_bytes_total`
   - `kubevirt_vmi_network_transmit_bytes_total`
   - `kubevirt_vmi_network_receive_errors_total`
4. Select specific VM using label selector
5. Click **Add Query** for comparisons
6. Switch to **Graph** view for visualization
7. Save useful queries for recurring checks

**Setting Up Alerts:**

1. Navigate to **Observe** → **Alerting**
2. Click **Alerting Rules** tab
3. Review existing rules for VM networking
4. Create custom alert rule:
5. Click **Create** → **Alerting Rule**
6. Define alert for high network errors:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vm-network-alerts
  namespace: openshift-cnv
spec:
  groups:
    - name: virtualization.network.rules
      interval: 30s
      rules:
        - alert: VMHighNetworkErrors
          expr: rate(kubevirt_vmi_network_receive_errors_total[5m]) > 10
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High network errors on VM {{ $labels.name }}"
```
7. Click **Create**
8. Navigate to **Observe** → **Alerting**
9. Verify alert appears in rules list
10. Test by generating network errors (optional)

**Validation in GUI:**
- Metrics display data for all running VMs
- Graphs show expected network patterns
- Alerts configured and visible in alerting dashboard
- No missing data or broken queries

**Common Mistakes:**
- Not establishing baseline metrics makes anomaly detection impossible
- Ignoring gradual performance degradation
- Over-alerting on normal traffic spikes
- Not correlating network metrics with application performance

---

### 11.2 Common Network Issues

**What it is:** Identifying and resolving frequently encountered VM networking problems using GUI tools.

**Why it matters:**
- Reduces mean time to resolution (MTTR)
- Prevents recurring issues through proper diagnosis
- Builds operational knowledge base

**GUI-Based Configuration Steps:**

**Issue 1: VM Cannot Get IP Address:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. Click on affected VM
3. Check **Events** tab for DHCP-related errors
4. Select **Network Interfaces** tab
5. Verify interface shows "Bound" status
6. If not bound, check NetworkAttachmentDefinition:
   - Navigate to **Networking** → **NetworkAttachmentDefinitions**
   - Click on relevant NAD
   - Review YAML for configuration errors
7. Check node network status:
   - Navigate to **Compute** → **Nodes**
   - Click on node hosting VM
   - Check **Conditions** for network problems
8. Verify bridge exists:
   - Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
   - Check policy status shows "Available"
   - Review NodeNetworkState for bridge creation
9. Access VM console
10. Manually request DHCP: `sudo dhclient -v eth1`
11. Check for DHCP discover/offer in output
12. If no response, verify DHCP server availability on network

**Issue 2: VM Network Interface Not Attaching:**

1. Navigate to affected VM details
2. Select **Events** tab
3. Look for "FailedAttachNetwork" events
4. Click on event to expand details
5. Common causes and checks:
   - NetworkAttachmentDefinition not found:
     - Verify NAD exists in same namespace as VM
     - Check NAD name spelling in VM spec
   - Resource not available:
     - For SR-IOV, check VF availability in node state
     - Navigate to **Networking** → **SR-IOV Network Node States**
     - Verify VFs allocated and not all in use
   - Bridge doesn't exist:
     - Check NodeNetworkConfigurationPolicy
     - Verify bridge created on VM's current node
6. Try removing and re-adding interface:
   - In VM details, go to **Network Interfaces** tab
   - Delete problematic interface
   - Re-add using correct configuration
7. Check pod events:
   - Navigate to **Workloads** → **Pods**
   - Find `virt-launcher-<vm-name>-*` pod
   - Review pod **Events** tab for network errors

**Issue 3: No Network Connectivity Between VMs:**

1. Navigate to affected VMs
2. Verify both VMs are running
3. Check VMs are on same network:
   - Review **Network Interfaces** tab
   - Confirm both use same NetworkAttachmentDefinition
4. Test from VM console:
   - Access first VM console
   - Get IP: `ip addr show`
   - Ping second VM IP
5. If ping fails, check NetworkPolicies:
   - Navigate to **Networking** → **NetworkPolicies**
   - Identify policies affecting VMs
   - Review ingress/egress rules
   - Temporarily delete policy to test (re-create after)
6. Check for CNI issues:
   - Navigate to **Workloads** → **Pods**
   - Filter namespace: `openshift-multus`
   - Verify all multus pods running
   - Check logs for errors
7. Verify bridge connectivity on nodes:
   - Navigate to **Compute** → **Nodes**
   - Access node terminal
   - Check bridge: `ip link show br1`
   - Verify bridge has ports: `brctl show br1`

**Issue 4: Poor Network Performance:**

1. Navigate to affected VM
2. Check **Metrics** tab
3. Look for:
   - Sudden drops in throughput
   - High packet errors
   - Retransmissions
4. Compare with other VMs on same node
5. Check node resource usage:
   - Navigate to **Compute** → **Nodes**
   - Click on node hosting VM
   - Review CPU and memory usage
6. Verify interface type:
   - In VM spec, check network interface model
   - Ensure using `virtio` for best performance
   - Change if using older model (requires VM restart)
7. Check for CPU pinning issues:
   - Review VM **YAML**
   - Look for CPU allocation
   - Ensure sufficient CPU assigned
8. Test with iperf if available:
   - Deploy iperf server VM
   - Run iperf client from affected VM
   - Compare results with baseline

**Validation in GUI:**
- Resolved issues show no errors in events
- VMs obtain IPs successfully
- Network connectivity tests pass
- Performance metrics return to normal
- No recurring error patterns

**Common Mistakes:**
- Jumping to complex solutions before checking basic configuration
- Not capturing error messages before attempting fixes
- Changing multiple things at once makes root cause unclear
- Not documenting resolution steps for future reference

---

### 11.3 Network Diagnostics Tools

**What it is:** Using built-in OpenShift console features and VM-based tools to diagnose network problems.

**Why it matters:**
- Enables systematic troubleshooting approach
- Reduces dependence on external tools
- Provides evidence for support cases

**GUI-Based Configuration Steps:**

**Console-Based Diagnostics:**

1. Navigate to **Observe** → **Dashboards**
2. Select "OpenShift Virtualization / VMs" dashboard
3. Review network-related panels:
   - Network Traffic
   - Network Errors
   - Interface Status
4. Filter by namespace or VM name
5. Identify anomalies in graphs
6. Adjust time range to find issue onset
7. Export data using "Export" button for analysis

**Using Debug Pods:**

1. Navigate to **Workloads** → **Pods**
2. Click **Create Pod**
3. Select **YAML view**
4. Create debug pod on same network as VM:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: linux-bridge-net1
spec:
  containers:
  - name: debug
    image: nicolaka/netshoot
    command: ["sleep", "3600"]
  nodeSelector:
    kubernetes.io/hostname: <node-name>
```
5. Click **Create**
6. Wait for pod to reach "Running" status
7. Click pod name
8. Select **Terminal** tab
9. Run diagnostic commands:
   - `ping <vm-ip>` - Basic connectivity
   - `traceroute <vm-ip>` - Path analysis
   - `nmap -p 1-1000 <vm-ip>` - Port scan
   - `tcpdump -i any -n` - Packet capture
   - `iperf3 -c <vm-ip>` - Performance test
10. Document findings

**Node Network Debugging:**

1. Navigate to **Compute** → **Nodes**
2. Click on node hosting problematic VM
3. Select **Terminal** tab
4. Check bridge status:
   ```bash
   ip link show
   brctl show
   ip addr show br1
   ```
5. Verify interface connectivity:
   ```bash
   ethtool eth1
   ethtool -S eth1  # Statistics
   ```
6. Check for errors:
   ```bash
   dmesg | grep -i network
   journalctl -u NetworkManager --since "1 hour ago"
   ```
7. Test connectivity from node:
   ```bash
   ping <vm-ip>
   curl -I http://<vm-ip>:port
   ```

**VM Console Diagnostics:**

1. Navigate to affected VM
2. Click **Console** tab
3. Log into VM
4. Inside VM, run:
   ```bash
   # Interface status
   ip addr show
   ip link show
   ip route show
   
   # DNS testing
   cat /etc/resolv.conf
   nslookup google.com
   dig google.com
   
   # Connectivity testing
   ping -c 4 <gateway-ip>
   ping -c 4 8.8.8.8
   traceroute google.com
   
   # Socket statistics
   ss -tulpn
   netstat -i  # Interface stats
   
   # Firewall (if applicable)
   sudo firewall-cmd --list-all
   sudo iptables -L -n -v
   ```
5. Document all outputs

**Packet Capture from Node:**

1. Navigate to node terminal (as above)
2. Identify VM interface on node:
   ```bash
   virsh list --all
   virsh domiflist <vm-domain>
   ```
3. Capture traffic:
   ```bash
   tcpdump -i <vm-interface> -w /tmp/capture.pcap
   ```
4. Let run for 30-60 seconds while reproducing issue
5. Stop capture (Ctrl+C)
6. Copy capture file:
   ```bash
   base64 /tmp/capture.pcap > /tmp/capture.b64
   ```
7. Copy output for analysis with Wireshark locally

**Validation in GUI:**
- Diagnostic commands complete without errors
- Captured data provides actionable insights
- Issue symptoms visible in diagnostic output
- Root cause identified through systematic testing

**Common Mistakes:**
- Not documenting diagnostic steps and results
- Running diagnostics from wrong network context
- Capturing too much data makes analysis difficult
- Not testing from multiple points in network path

---

### 11.4 Backup and Recovery

**What it is:** Backing up network configurations and recovering from network-related failures.

**Why it matters:**
- Protects against configuration loss
- Enables quick recovery from mistakes
- Supports disaster recovery planning

**GUI-Based Configuration Steps:**

**Backing Up Network Configurations:**

1. Navigate to **Networking** → **NetworkAttachmentDefinitions**
2. Select each NAD one by one
3. Click on NAD name
4. Select **YAML** tab
5. Click **Download** button to save YAML
6. Save with descriptive name: `nad-<name>-<date>.yaml`
7. Repeat for all NADs
8. Navigate to **Networking** → **NodeNetworkConfigurationPolicies**
9. Download each policy YAML
10. Navigate to **Networking** → **SR-IOV Networks** (if applicable)
11. Download all SR-IOV configurations
12. Navigate to **Networking** → **SR-IOV Network Node Policies**
13. Download all node policies
14. Store all YAMLs in version control system

**Backing Up VM Network Configurations:**

1. Navigate to **Virtualization** → **VirtualMachines**
2. For each critical VM:
3. Click on VM name
4. Select **YAML** tab
5. Click **Download**
6. Save as `vm-<name>-<date>.yaml`
7. Alternative: Backup entire namespace:
8. Navigate to **Projects**
9. Click on namespace
10. Click **YAML** in Resources tab
11. Document all resources for namespace

**Recovering Network Configuration:**

1. Navigate to **Networking** → **NetworkAttachmentDefinitions**
2. Click **Create NetworkAttachmentDefinition**
3. Select **YAML view**
4. Paste backed-up NAD configuration
5. Click **Create**
6. Verify NAD appears and is functional
7. Test with existing VM or create test VM
8. Repeat for other network resources

**Recovering from Failed Configuration:**

1. If configuration change breaks networking:
2. Navigate to resource that was changed
3. Click **YAML** tab
4. Click **Reload** to discard unsaved changes (if just edited)
5. Or use **Actions** → **Edit** to manually fix
6. For NodeNetworkConfigurationPolicy:
   - Navigate to policy
   - Click **Actions** → **Delete**
   - Recreate from backup
   - Wait for nodes to stabilize
7. For catastrophic failure:
   - Delete problematic resource entirely
   - Recreate from known-good backup YAML
   - Wait for reconciliation

**Emergency VM Network Recovery:**

1. If VM loses network connectivity:
2. Navigate to affected VM
3. Click **Actions** → **Stop**
4. Wait for full stop
5. Click **YAML** tab
6. Locate `spec.template.spec.networks` section
7. Remove problematic network entry
8. Click **Save**
9. Click **Actions** → **Start**
10. VM should start with pod network only
11. Diagnose and fix network issue
12. Re-add fixed network configuration

**Testing Backup Restoration:**

1. Schedule regular backup testing:
2. Create test namespace: `network-backup-test`
3. Restore all NADs to test namespace
4. Create test VM using restored configurations
5. Verify VM can attach and use networks
6. Document any restoration issues
7. Update backup procedures accordingly
8. Delete test namespace after validation

**Validation in GUI:**
- All backup YAMLs download successfully
- Restored configurations create resources without errors
- Test VMs using restored configs function properly
- Recovery procedures documented and tested
- Backup schedule established and followed

**Common Mistakes:**
- Not testing backup restorations regularly
- Backing up only VMs without network dependencies
- Not versioning backup files (can't track changes)
- Storing backups only within cluster (vulnerable to cluster failure)

---

## 12. Best Practices Summary

This section consolidates key best practices for OpenShift Virtualization networking in production environments.

### Network Architecture

**Planning:**
- Document all VLAN assignments before starting configuration
- Create naming convention for networks and VMs early
- Separate management, application, and storage networks
- Plan IP addressing scheme to avoid conflicts

**Implementation:**
- Use dedicated physical NICs for different traffic types
- Implement redundant network paths where possible
- Reserve network capacity for migration traffic
- Document network topology in wiki/repository

### Configuration Management

**Standards:**
- Use consistent naming across all network resources
- Label all VMs with clear application and tier labels
- Store all network configuration YAMLs in version control
- Create templates for common network patterns

**Change Control:**
- Test network changes in non-production first
- Back up configurations before making changes
- Apply changes during maintenance windows
- Validate after each change before proceeding

### Security

**Network Isolation:**
- Implement NetworkPolicies for all production namespaces
- Use principle of least privilege for network access
- Separate tenant workloads into different namespaces
- Enable MAC spoofing protection on all bridges

**Monitoring:**
- Set up alerts for network anomalies
- Monitor network policy violations
- Track network performance metrics
- Review network access logs regularly

### Performance

**Optimization:**
- Use virtio network interfaces for VMs
- Enable SR-IOV for latency-sensitive workloads
- Dedicate network bandwidth for live migration
- Size node network interfaces appropriately

**Capacity Planning:**
- Monitor network utilization trends
- Plan for growth in VM count
- Reserve capacity for failover scenarios
- Document network capacity limits

### Operations

**Day-2 Activities:**
- Perform regular backup of network configurations
- Test backup restoration quarterly
- Review and update network documentation monthly
- Conduct network troubleshooting training

**Troubleshooting:**
- Maintain runbook for common network issues
- Use systematic diagnostic approach
- Document all issues and resolutions
- Share knowledge across team

### High Availability

**Resilience:**
- Configure bonded interfaces where supported
- Implement network redundancy at all layers
- Test failover scenarios regularly
- Monitor backup path availability

**Migration:**
- Verify migration network performance regularly
- Test live migration across all node pairs
- Document migration best practices
- Monitor migration success rates

---

## 13. References and Resources

### Official Documentation

**Red Hat OpenShift Virtualization:**
- OpenShift Virtualization Documentation: https://docs.openshift.com/container-platform/latest/virt/about-virt.html
- Networking Guide: https://docs.openshift.com/container-platform/latest/virt/vm_networking/virt-about-networking.html
- SR-IOV Documentation: https://docs.openshift.com/container-platform/latest/networking/hardware_networks/about-sriov.html

**Kubernetes Networking:**
- NetworkPolicy Documentation: https://kubernetes.io/docs/concepts/services-networking/network-policies/
- Multus CNI: https://github.com/k8snetworkplumbingwg/multus-cni
- CNI Specification: https://github.com/containernetworking/cni

### Community Resources

**Forums and Support:**
- Red Hat Customer Portal: https://access.redhat.com
- OpenShift Commons: https://commons.openshift.org
- Kubernetes Slack #sig-network: https://kubernetes.slack.com

**Tutorials and Examples:**
- OpenShift Virtualization Labs: https://github.com/RHsyseng/telco-operations
- KubeVirt Examples: https://kubevirt.io/user-guide/

### Tools and Utilities

**Network Diagnostics:**
- netshoot container: nicolaka/netshoot
- OpenShift must-gather: https://docs.openshift.com/container-platform/latest/support/gathering-cluster-data.html

**Performance Testing:**
- iperf3 for bandwidth testing
- qperf for latency and throughput

### Learning Paths

**Red Hat Training:**
- Red Hat OpenShift Virtualization (DO316)
- Red Hat OpenShift Administration (DO280)

**Self-Paced Learning:**
- Red Hat Developer Sandbox
- OpenShift Interactive Learning Portal
- KubeVirt Community Workshops

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Feb 2026 | Sajal Jana | Initial comprehensive guide |

---

## Feedback and Contributions

This guide is maintained as a living document. For updates, corrections, or additions, please:

1. Test all procedures in your environment
2. Document your specific use case
3. Share improvements with the community

---

**End of Guide**

This guide provides complete coverage of OpenShift Virtualization networking using only the Web Console, from initial setup through production operations and troubleshooting.
