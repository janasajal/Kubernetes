# Red Hat OpenShift Virtualization Guide for Beginners

**Author:** Sajal Jana

---

## Table of Contents

1. [Introduction](#introduction)
2. [KubeVirt Architecture](#kubevirt-architecture)
   - [Definition](#definition)
   - [Production Importance](#production-importance)
   - [Simple Analogy](#simple-analogy)
   - [Administrator Tasks (Step-by-Step)](#administrator-tasks-step-by-step)
3. [Live Migration](#live-migration)
   - [Definition](#definition-1)
   - [Production Importance](#production-importance-1)
   - [Simple Analogy](#simple-analogy-1)
   - [Administrator Tasks (Step-by-Step)](#administrator-tasks-step-by-step-1)
4. [Multus CNI (Networking)](#multus-cni-networking)
   - [Definition](#definition-2)
   - [Production Importance](#production-importance-2)
   - [Simple Analogy](#simple-analogy-2)
   - [Administrator Tasks (Step-by-Step)](#administrator-tasks-step-by-step-2)
5. [CDI (Containerized Data Importer)](#cdi-containerized-data-importer)
   - [Definition](#definition-3)
   - [Production Importance](#production-importance-3)
   - [Simple Analogy](#simple-analogy-3)
   - [Administrator Tasks (Step-by-Step)](#administrator-tasks-step-by-step-3)
6. [Node Maintenance Mode](#node-maintenance-mode)
   - [Definition](#definition-4)
   - [Production Importance](#production-importance-4)
   - [Simple Analogy](#simple-analogy-4)
   - [Administrator Tasks (Step-by-Step)](#administrator-tasks-step-by-step-4)
7. [Conclusion](#conclusion)

---

## Introduction

Welcome to your journey into Red Hat OpenShift Virtualization! This guide is designed specifically for beginners who want to understand how OpenShift Virtualization works in real-world production environments. As you read through each topic, you'll gain practical knowledge about the core components and learn the daily responsibilities of OpenShift administrators managing virtualized workloads.

---

## KubeVirt Architecture

### Definition

KubeVirt is an extension to Kubernetes that enables running and managing traditional virtual machines alongside containers by treating VMs as native Kubernetes objects.

### Production Importance

In production environments, KubeVirt Architecture is critical because it allows organizations to modernize their infrastructure gradually without forcing immediate application rewrites. Legacy applications that cannot be containerized can run as VMs on the same platform as modern containerized applications, providing unified management, reducing infrastructure costs, and enabling teams to leverage Kubernetes capabilities like automated scaling, self-healing, and declarative configuration for both VMs and containers. This architectural consolidation improves resource utilization, simplifies operations, and provides a clear migration path from traditional virtualization to cloud-native infrastructure.

### Simple Analogy

Think of KubeVirt like a modern apartment building that was originally designed only for studio apartments (containers), but the architect added a clever extension that also allows traditional family apartments (VMs) to exist in the same building. Both types of residents use the same lobby, elevators, security system, and building management, even though their living spaces are structured differently inside. The building manager doesn't need separate management systems for different apartment types.

### Administrator Tasks (Step-by-Step)

#### 1. Install and Enable OpenShift Virtualization Operator

**What:** Deploy the OpenShift Virtualization operator through the OperatorHub in the OpenShift web console or via CLI.

**Why:** The operator manages all KubeVirt components automatically, ensuring they're properly installed, configured, and updated. Without it, you'd have no virtualization capabilities in your cluster.

**How:**
- Navigate to OperatorHub in OpenShift Console
- Search for "OpenShift Virtualization"
- Click Install and select installation options (namespace, update channel)
- Wait for operator installation to complete

#### 2. Create HyperConverged Custom Resource

**What:** Deploy a HyperConverged CR which triggers the installation of all necessary KubeVirt components (virt-api, virt-controller, virt-handler).

**Why:** This single resource automatically configures and deploys the entire virtualization stack with production-ready defaults, saving hours of manual configuration work.

**How:**
- Access the OpenShift Virtualization operator page
- Create a HyperConverged instance using the provided template
- Verify all components are running: `oc get pods -n openshift-cnv`

#### 3. Verify Node Capabilities and Label Nodes

**What:** Check that worker nodes have virtualization extensions enabled (Intel VT-x or AMD-V) and apply appropriate labels for VM scheduling.

**Why:** VMs require hardware virtualization support to run efficiently; without it, performance degrades significantly. Labels help Kubernetes scheduler place VMs on appropriate nodes.

**How:**
- Check virtualization support: `oc debug node/<node-name> -- chroot /host lscpu | grep Virtualization`
- Apply labels if needed: `oc label node <node-name> kubevirt.io/schedulable=true`

#### 4. Configure Storage Classes for Virtual Machines

**What:** Identify or create storage classes that will be used for VM disks and ensure they support the required access modes.

**Why:** VMs need persistent storage for their disks. The storage class determines performance characteristics, replication, and whether VMs can be live migrated (requires ReadWriteMany for migration).

**How:**
- List available storage classes: `oc get storageclass`
- Verify access modes and provisioner types
- Set appropriate default storage class for VM workloads

#### 5. Monitor KubeVirt Component Health

**What:** Regularly check the status of core KubeVirt components including virt-api, virt-controller, and virt-handler pods.

**Why:** These components are the foundation of virtualization functionality. If any fail, VM operations (creation, migration, management) will be impacted. Early detection prevents production incidents.

**How:**
- Check pod status: `oc get pods -n openshift-cnv`
- Review logs for errors: `oc logs -n openshift-cnv <pod-name>`
- Monitor resource consumption of KubeVirt components

#### 6. Set Up Resource Quotas and Limits

**What:** Define ResourceQuota and LimitRange objects in namespaces where VMs will run to control resource consumption.

**Why:** VMs can consume significant CPU and memory. Without quotas, a single team could exhaust cluster resources, impacting other workloads. This ensures fair resource distribution and prevents resource starvation.

**How:**
- Create ResourceQuota for namespace: `oc create quota vm-quota --hard=requests.memory=64Gi,limits.memory=128Gi`
- Apply LimitRange to set default VM resource limits
- Monitor quota usage regularly

---

## Live Migration

### Definition

Live Migration is the process of moving a running virtual machine from one physical node to another without shutting it down, maintaining active connections and application availability.

### Production Importance

Live Migration is essential for production because it enables zero-downtime maintenance, upgrades, and load balancing. When you need to patch a server, replace hardware, or rebalance cluster resources, live migration allows you to evacuate VMs from nodes without service interruptions, maintaining SLAs and user experience. This capability transforms planned maintenance from a disruptive event requiring downtime windows into routine operations that can occur during business hours, dramatically improving operational flexibility and system availability.

### Simple Analogy

Imagine you're talking to someone on your phone while walking through a building, and as you move, your call seamlessly switches between different cell towers without the call dropping or even stuttering. Live migration works the same way—your VM keeps running and serving users while the underlying "tower" (physical server) changes beneath it, and users never notice the transition.

### Administrator Tasks (Step-by-Step)

#### 1. Verify Live Migration Prerequisites

**What:** Confirm that your cluster meets all requirements for live migration: shared storage with ReadWriteMany access, sufficient network bandwidth, and compatible node configurations.

**Why:** Live migration requires transferring VM memory state and maintaining disk access during migration. Without proper storage and network setup, migrations will fail or cause service disruption.

**How:**
- Verify storage class supports RWX: `oc get storageclass -o json | grep accessModes`
- Check network connectivity between nodes
- Ensure nodes have similar CPU features: `oc get nodes -o json | grep cpu`

#### 2. Configure Live Migration Settings

**What:** Customize the KubeVirt CR to set migration parameters like bandwidth limits, parallel migrations, and timeout values.

**Why:** Default settings may not suit your production environment. Limiting bandwidth prevents migration from saturating networks, while timeout adjustments accommodate large VMs or slower storage.

**How:**
- Edit HyperConverged CR: `oc edit hyperconverged kubevirt-hyperconverged -n openshift-cnv`
- Set migration configuration parameters:
  ```yaml
  spec:
    liveMigrationConfig:
      bandwidthPerMigration: "64Mi"
      parallelMigrationsPerCluster: 5
      progressTimeout: 150
  ```

#### 3. Test Live Migration in Non-Production

**What:** Perform controlled migrations of test VMs to validate the process and measure performance impact before using in production.

**Why:** Testing reveals potential issues like network bottlenecks, storage performance problems, or incompatible node configurations before they cause production incidents. You also establish baseline migration times for capacity planning.

**How:**
- Create test VM: `oc create -f test-vm.yaml`
- Trigger manual migration: `virtctl migrate <vm-name>`
- Monitor migration progress: `oc get virtualmachineinstancemigration`
- Measure downtime and performance impact

#### 4. Initiate Manual Live Migration

**What:** Manually trigger migration of specific VMs from one node to another using the virtctl command or OpenShift console.

**Why:** When you need to perform maintenance on a specific node, drain workloads due to performance issues, or rebalance cluster load, manual migration gives you precise control over VM placement.

**How:**
- List running VMs and their nodes: `oc get vmi -o wide`
- Trigger migration: `virtctl migrate <vm-name> --namespace <namespace>`
- Monitor status: `oc describe vmi <vm-name>`

#### 5. Configure Automated Migration Policies

**What:** Set up eviction strategies and node drain policies that automatically trigger migrations when nodes enter maintenance or experience issues.

**Why:** Automation ensures VMs are safely evacuated during planned maintenance without requiring manual intervention for each VM. This reduces human error and speeds up operations in large environments.

**How:**
- Enable eviction strategy in VM spec:
  ```yaml
  spec:
    evictionStrategy: LiveMigrate
  ```
- Configure node drain before maintenance: `oc adm drain <node-name> --delete-emptydir-data`

#### 6. Monitor and Troubleshoot Failed Migrations

**What:** Track migration events, identify failures, and analyze logs to resolve issues preventing successful migrations.

**Why:** Failed migrations can leave nodes unable to enter maintenance mode or VMs stuck in unstable states. Quick troubleshooting prevents extended maintenance windows and potential service disruption.

**How:**
- Check migration status: `oc get virtualmachineinstancemigration`
- Review migration events: `oc describe virtualmachineinstancemigration <migration-name>`
- Analyze virt-handler logs: `oc logs -n openshift-cnv <virt-handler-pod>`
- Common issues: insufficient resources on target node, storage access problems, network connectivity

#### 7. Establish Migration Performance Baselines

**What:** Document typical migration times, resource consumption, and success rates for different VM sizes and workload types.

**Why:** Baselines help you capacity plan, set realistic maintenance windows, identify performance degradation, and optimize migration configurations for your specific environment.

**How:**
- Record migration duration for various VM sizes
- Measure network utilization during migrations
- Track CPU and memory overhead on source and target nodes
- Document any application-specific behavior during migration

---

## Multus CNI (Networking)

### Definition

Multus CNI is a Container Network Interface plugin that enables attaching multiple network interfaces to pods and virtual machines in Kubernetes, allowing workloads to connect to different networks simultaneously.

### Production Importance

In production, Multus is critical because enterprise applications often require network segmentation for security, compliance, and performance reasons. A database VM might need one interface for management traffic, another for application traffic, and a third for backup traffic—each on isolated networks with different security policies and bandwidth guarantees. Without Multus, you're limited to a single network per VM, forcing all traffic through one interface and preventing proper network isolation, which is unacceptable for regulated industries (finance, healthcare) or high-security environments.

### Simple Analogy

Think of Multus like a computer with multiple network cards installed. Your office computer might have one network cable for the corporate internet, another for the secure internal financial systems network, and a third for the isolated backup network. Each network card connects to a different network for different purposes, and they don't interfere with each other. Multus gives your VMs and pods the same capability—multiple "network cards" for different networks.

### Administrator Tasks (Step-by-Step)

#### 1. Verify Multus Installation and Readiness

**What:** Confirm that Multus CNI is installed and properly functioning in your OpenShift cluster as part of the OpenShift Virtualization deployment.

**Why:** Multus is automatically installed with OpenShift but must be operational before you can configure additional networks. Verifying early prevents configuration issues later.

**How:**
- Check Multus daemon set: `oc get ds multus -n openshift-multus`
- Verify Multus pods are running on all nodes: `oc get pods -n openshift-multus`
- Check for Multus configuration: `oc get network-attachment-definitions --all-namespaces`

#### 2. Create Network Attachment Definitions (NADs)

**What:** Define NetworkAttachmentDefinition custom resources that specify additional network configurations (bridge, SR-IOV, macvlan) for VMs to attach to.

**Why:** NADs are the configuration templates that tell Multus how to create and configure additional network interfaces. Each NAD represents a specific network with its own addressing, VLAN, and connectivity rules.

**How:**
- Create a bridge NAD for VM networks:
  ```yaml
  apiVersion: k8s.cni.cncf.io/v1
  kind: NetworkAttachmentDefinition
  metadata:
    name: database-network
    namespace: production-vms
  spec:
    config: '{
      "cniVersion": "0.3.1",
      "type": "bridge",
      "bridge": "br1",
      "vlan": 100,
      "ipam": {
        "type": "static"
      }
    }'
  ```
- Apply the configuration: `oc create -f network-attachment.yaml`

#### 3. Configure Linux Bridge for VM Networks

**What:** Set up Linux bridges on worker nodes to provide Layer 2 connectivity between VMs and external networks or VLANs.

**Why:** Bridges allow VMs to communicate with physical network infrastructure just like traditional VMs. This is essential when VMs need to be accessible from outside the cluster or participate in existing VLAN-based network architectures.

**How:**
- Create NodeNetworkConfigurationPolicy:
  ```yaml
  apiVersion: nmstate.io/v1
  kind: NodeNetworkConfigurationPolicy
  metadata:
    name: br1-vlan100
  spec:
    nodeSelector:
      node-role.kubernetes.io/worker: ""
    desiredState:
      interfaces:
        - name: br1
          type: linux-bridge
          state: up
          bridge:
            port:
              - name: eth1
                vlan:
                  id: 100
  ```

#### 4. Attach Additional Networks to Virtual Machines

**What:** Modify VM specifications to include additional network interfaces that reference the NetworkAttachmentDefinitions you created.

**Why:** This is how you actually assign the extra networks to your VMs. A VM can have its default pod network plus multiple additional networks for segmented traffic.

**How:**
- Edit VM specification to add networks:
  ```yaml
  spec:
    template:
      spec:
        domain:
          devices:
            interfaces:
            - name: default
              masquerade: {}
            - name: database-net
              bridge: {}
        networks:
        - name: default
          pod: {}
        - name: database-net
          multus:
            networkName: database-network
  ```

#### 5. Configure SR-IOV for High-Performance Networking

**What:** Set up Single Root I/O Virtualization to provide VMs with direct access to physical network adapters, bypassing the kernel for maximum performance.

**Why:** For latency-sensitive applications (trading platforms, real-time analytics, high-throughput databases), SR-IOV provides near-native network performance by giving VMs dedicated hardware resources.

**How:**
- Install SR-IOV Network Operator
- Create SriovNetworkNodePolicy to configure physical NICs
- Create SriovNetwork resource defining the network
- Attach SR-IOV network to VMs in their spec

#### 6. Implement Network Isolation and Security Policies

**What:** Apply NetworkPolicy resources to control traffic flow between VMs and define which networks can communicate with each other.

**Why:** Multiple networks without proper isolation defeats the purpose. Network policies enforce segmentation at the software level, preventing unauthorized access between network segments even if they're on the same physical infrastructure.

**How:**
- Create NetworkPolicy for VM namespace:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: database-isolation
  spec:
    podSelector:
      matchLabels:
        app: database-vm
    policyTypes:
    - Ingress
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: application-tier
  ```

#### 7. Monitor and Troubleshoot Multi-Network Connectivity

**What:** Use diagnostic tools to verify network connectivity, troubleshoot interface attachments, and monitor traffic across different networks.

**Why:** With multiple networks, troubleshooting becomes more complex. You need to verify which interfaces are active, check routing between networks, and diagnose connectivity issues specific to each network attachment.

**How:**
- Access VM console: `virtctl console <vm-name>`
- Check interface configuration inside VM: `ip addr show`
- Verify network attachment from host: `oc exec <virt-launcher-pod> -- ip link`
- Test connectivity: `ping` from VM to verify each network path
- Review CNI logs: `oc logs -n openshift-multus <multus-pod>`

---

## CDI (Containerized Data Importer)

### Definition

CDI (Containerized Data Importer) is a utility that automates importing, uploading, and cloning virtual machine disk images into Kubernetes persistent volumes, making them ready for use by VMs.

### Production Importance

CDI is vital in production because manually managing VM disk images at scale is error-prone and time-consuming. When you need to provision hundreds of VMs from golden images, migrate existing VMs from VMware or other platforms, or enable developers to quickly spin up pre-configured environments, CDI automates the entire process. It handles format conversions, validates images, manages storage efficiently, and integrates with Kubernetes storage classes—turning what would be hours of manual work into declarative, repeatable operations. This automation reduces human error, speeds up provisioning, and enables self-service VM deployment.

### Simple Analogy

Think of CDI like an automated printing and copying service in an office building. Instead of manually carrying USB drives around and copying files to each computer (traditional VM image deployment), you upload a document once to a central system, and it automatically converts it to the right format, makes copies, and delivers them to the right computers on the right floors. CDI does the same with VM images—you point it at an image source, and it automatically downloads, converts, and stores it in the right format on the right storage system.

### Administrator Tasks (Step-by-Step)

#### 1. Verify CDI Installation and Components

**What:** Confirm that CDI operator and its components (cdi-controller, cdi-apiserver, cdi-uploadproxy) are installed and running properly.

**Why:** CDI is installed with OpenShift Virtualization, but you must verify it's operational before attempting to import images. Non-functional CDI components will cause all image operations to fail.

**How:**
- Check CDI pods: `oc get pods -n openshift-cnv | grep cdi`
- Verify CDI operator: `oc get deployment -n openshift-cnv cdi-operator`
- Check CDI CR status: `oc get cdi cdi -o yaml`

#### 2. Create DataVolume for Image Import

**What:** Define a DataVolume custom resource that specifies the source image location (URL, registry, upload), target storage class, and desired disk size.

**Why:** DataVolume is the declarative way to request disk image imports in Kubernetes. It abstracts the complexity of image handling, storage provisioning, and format conversion into a single resource definition.

**How:**
- Create DataVolume to import from URL:
  ```yaml
  apiVersion: cdi.kubevirt.io/v1beta1
  kind: DataVolume
  metadata:
    name: rhel8-golden-image
    namespace: vm-images
  spec:
    source:
      http:
        url: "https://cloud.example.com/images/rhel8.qcow2"
    pvc:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
      storageClassName: fast-ssd
  ```
- Apply and monitor: `oc create -f datavolume.yaml && oc get dv -w`

#### 3. Upload Local VM Disk Images

**What:** Use virtctl or the web console to upload VM disk images directly from your local machine to the cluster storage.

**Why:** When migrating existing VMs or testing custom images that aren't hosted on a URL, direct upload is the most straightforward method to get images into the cluster.

**How:**
- Upload using virtctl: `virtctl image-upload dv <datavolume-name> --image-path=/path/to/image.qcow2 --size=30Gi --storage-class=fast-ssd --namespace=vm-images`
- Monitor upload progress: `oc get dv <datavolume-name> -w`
- Verify PVC creation: `oc get pvc`

#### 4. Clone Existing Data Volumes

**What:** Create new DataVolumes that clone existing VM disks, useful for creating multiple VMs from a golden image or backing up VM disks.

**Why:** Cloning is faster and more efficient than re-importing images. When provisioning multiple VMs from the same template, cloning the base disk locally within the cluster is significantly faster than downloading from external sources repeatedly.

**How:**
- Create cloning DataVolume:
  ```yaml
  apiVersion: cdi.kubevirt.io/v1beta1
  kind: DataVolume
  metadata:
    name: vm-database-01-disk
  spec:
    source:
      pvc:
        namespace: vm-images
        name: rhel8-golden-image
    pvc:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
  ```

#### 5. Configure Image Import from Container Registries

**What:** Set up DataVolume sources that pull VM disk images stored as container images in registries like Quay, Harbor, or Docker Hub.

**Why:** Storing VM images in container registries leverages existing registry infrastructure, image scanning, access controls, and distribution capabilities. It also enables CI/CD pipelines to build and publish VM images alongside container images.

**How:**
- Create DataVolume from registry:
  ```yaml
  spec:
    source:
      registry:
        url: "docker://quay.io/company/rhel8-database:latest"
        pullMethod: node
    pvc:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 30Gi
  ```

#### 6. Implement Golden Image Management Strategy

**What:** Establish a process for maintaining, versioning, and distributing approved VM images across the organization using CDI and DataVolumes.

**Why:** In production, you need controlled, tested, hardened images rather than ad-hoc imports. A golden image strategy ensures security patches, compliance requirements, and organizational standards are baked into VM base images.

**How:**
- Create dedicated namespace for golden images: `oc create namespace golden-images`
- Import approved images with version labels
- Set up RBAC to control who can modify golden images
- Document image contents and update schedule
- Use cloning from golden images for all new VMs

#### 7. Monitor and Troubleshoot Image Import Operations

**What:** Track DataVolume creation progress, identify failed imports, and resolve issues related to image formatting, storage, or network connectivity.

**Why:** Image imports can fail for many reasons: network timeouts, insufficient storage, unsupported formats, or incorrect URLs. Quick diagnosis prevents delays in VM provisioning and helps identify infrastructure issues.

**How:**
- Check DataVolume status: `oc get dv -A`
- Examine DataVolume events: `oc describe dv <datavolume-name>`
- Review importer pod logs: `oc logs <importer-pod-name>`
- Common issues to check:
  - Network connectivity to image source
  - Storage class provisioner availability
  - Sufficient quota in namespace
  - Image format compatibility (raw, qcow2 supported)

---

## Node Maintenance Mode

### Definition

Node Maintenance Mode is a process that safely cordons a node, evacuates all running workloads (including VMs through live migration), and prepares it for maintenance operations without causing service disruptions.

### Production Importance

Node Maintenance Mode is essential for maintaining production systems without downtime. When you need to apply OS patches, upgrade firmware, replace hardware, or perform routine maintenance, you cannot simply power off nodes—that would terminate workloads and violate SLAs. Maintenance mode provides the orchestration to gracefully move all workloads to healthy nodes, verify the evacuations completed successfully, and then allow safe maintenance operations. This capability transforms infrastructure maintenance from a high-risk, after-hours emergency activity into routine, safe operations that can occur during business hours, dramatically improving operational agility and reducing unplanned downtime.

### Simple Analogy

Think of node maintenance mode like closing one lane of a busy highway for repairs. Highway workers don't just throw up a barrier and stop traffic—they use signs well in advance to redirect vehicles to other lanes, ensure all cars have safely moved over, set up cones and barriers, and only then begin work. Similarly, maintenance mode doesn't just shut down a node—it redirects workloads to other nodes, waits for safe migration, marks the node as unavailable, and then permits maintenance. Traffic keeps flowing, just on different lanes.

### Administrator Tasks (Step-by-Step)

#### 1. Verify Cluster Has Sufficient Capacity

**What:** Check that remaining nodes have enough CPU, memory, and storage capacity to absorb workloads from the node entering maintenance.

**Why:** If the cluster lacks capacity, live migrations will fail, VMs will become unschedulable, or nodes will become overloaded—causing performance degradation or outages. Capacity verification prevents maintenance from causing cascading failures.

**How:**
- Check node resource allocation: `oc describe nodes | grep -A 5 "Allocated resources"`
- Review VM resource requests: `oc get vmi -A -o custom-columns=NAME:.metadata.name,CPU:.spec.domain.resources.requests.cpu,MEMORY:.spec.domain.resources.requests.memory`
- Calculate if remaining nodes can handle additional load
- Verify storage capacity for additional VM disks

#### 2. Place Node in Maintenance Mode Using Node Maintenance Operator

**What:** Create a NodeMaintenance custom resource that triggers automated cordon, drain, and workload evacuation for the specified node.

**Why:** The Node Maintenance Operator automates the complex orchestration of safely evacuating a node, including proper handling of VMs via live migration, waiting for successful evacuations, and tracking maintenance state.

**How:**
- Install Node Maintenance Operator if not present
- Create NodeMaintenance resource:
  ```yaml
  apiVersion: nodemaintenance.medik8s.io/v1beta1
  kind: NodeMaintenance
  metadata:
    name: worker-node-3-maintenance
  spec:
    nodeName: worker-node-3
    reason: "Monthly OS patching"
  ```
- Apply: `oc create -f node-maintenance.yaml`
- Monitor status: `oc get nodemaintenance -w`

#### 3. Manually Cordon and Drain Node (Alternative Method)

**What:** Use native Kubernetes commands to mark the node as unschedulable and evict all pods and VMs to other nodes.

**Why:** When the Node Maintenance Operator isn't available or for quick manual control, the cordon/drain commands provide direct node evacuation capabilities.

**How:**
- Cordon node to prevent new workloads: `oc adm cordon <node-name>`
- Verify node status: `oc get nodes` (look for SchedulingDisabled)
- Drain node with VM migration: `oc adm drain <node-name> --ignore-daemonsets --delete-emptydir-data --force`
- Monitor VM migrations: `oc get vmi -o wide --watch`

#### 4. Monitor Workload Migration Progress

**What:** Track the status of pods and VMs being evacuated from the node, ensuring all critical workloads successfully migrate before proceeding with maintenance.

**Why:** Blindly starting maintenance while migrations are still in progress can terminate workloads unexpectedly. Monitoring ensures all services are safely running elsewhere before you touch the hardware.

**How:**
- Check pods remaining on node: `oc get pods --all-namespaces --field-selector spec.nodeName=<node-name>`
- Monitor VM migrations: `oc get virtualmachineinstancemigration -w`
- Verify VMs have moved: `oc get vmi -o wide`
- Check for stuck or failed migrations: `oc get vmi -o json | jq '.items[] | select(.status.migrationState!=null)'`

#### 5. Perform Maintenance Operations

**What:** Once the node is fully drained and cordoned, execute the planned maintenance tasks: apply OS patches, upgrade firmware, replace hardware, or perform configuration changes.

**Why:** This is the actual purpose of maintenance mode—safely accessing the physical or virtual infrastructure without workloads running. The previous steps were all preparation for this moment.

**How:**
- SSH into node: `ssh core@<node-ip>`
- For OS updates: `rpm-ostree upgrade` and reboot
- For firmware updates: follow vendor procedures
- For hardware replacement: shut down node, swap hardware, restart
- Document all changes for audit and troubleshooting

#### 6. Return Node to Service (Uncordon)

**What:** Remove the maintenance state, uncordon the node, and allow the scheduler to begin placing workloads back on the node.

**Why:** After successful maintenance, the node should rejoin the pool of available resources. Uncordoning makes it available for scheduling again, restoring full cluster capacity.

**How:**
- If using Node Maintenance Operator: Delete NodeMaintenance resource: `oc delete nodemaintenance <maintenance-name>`
- If manual: Uncordon node: `oc adm uncordon <node-name>`
- Verify node is Ready and schedulable: `oc get nodes`
- Monitor workload distribution: `oc get pods -A -o wide`

#### 7. Verify Node Health and Workload Distribution

**What:** After returning to service, confirm the node is healthy, accepting workloads, and that cluster workload distribution is balanced appropriately.

**Why:** Maintenance operations can sometimes introduce issues that only manifest after the node rejoins. Early detection of problems prevents them from escalating into production incidents.

**How:**
- Check node status and conditions: `oc describe node <node-name>`
- Verify all node components healthy: `oc get pods -n openshift-cnv -o wide | grep <node-name>`
- Monitor for new pods scheduling to node: `oc get events --field-selector involvedObject.name=<node-name> --watch`
- Review node logs for errors: `oc adm node-logs <node-name> -u kubelet`
- Check VM distribution across cluster: `oc get vmi -A -o wide`

---

## Conclusion

Congratulations on completing this comprehensive guide to Red Hat OpenShift Virtualization! You've learned the fundamental concepts that power modern virtualized infrastructure in Kubernetes environments.

As a beginner, you now understand:

- **KubeVirt Architecture** – How virtual machines run as native Kubernetes objects alongside containers
- **Live Migration** – How to move VMs between nodes without downtime for seamless maintenance
- **Multus CNI** – How to provide multiple network interfaces for proper network segmentation
- **CDI** – How to efficiently import and manage VM disk images at scale
- **Node Maintenance Mode** – How to safely perform infrastructure maintenance without service disruptions

These aren't just theoretical concepts—they're the daily operational responsibilities of OpenShift administrators managing production virtualization platforms. Each topic represents real workflows you'll encounter when managing infrastructure at scale.

**Next Steps for Your Learning Journey:**

1. Set up a practice OpenShift environment using CodeReady Containers or a cloud provider
2. Deploy the OpenShift Virtualization operator and create your first VM
3. Practice each administrator task in a safe test environment
4. Experiment with failures and troubleshooting scenarios
5. Join the OpenShift and KubeVirt communities to learn from experienced practitioners



---

**Author:** Sajal Jana  
**Last Updated:** February 16, 2026
