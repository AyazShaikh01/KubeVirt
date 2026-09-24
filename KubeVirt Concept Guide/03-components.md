# 3. Components

*Part of the [KubeVirt Notes](./README.md) series. Previous: [Architecture](./02-architecture.md) · Next: [Virtual Machines →](./04-virtual-machines.md)*

> Quick recap: KubeVirt's additions split into control-plane components (decide what should happen) and node agents (make it happen). See [Architecture](./02-architecture.md) for how they interact in the VM-start sequence. This doc is the exhaustive reference for each piece — what it does, and what it depends on from Kubernetes.

## The control plane

### `virt-operator`

**Purpose:** manages KubeVirt itself — the meta-component.

It owns the lifecycle of the KubeVirt installation and its other components: install, upgrade, and keep them configured correctly. It implements the standard Kubernetes operator pattern — encoding the operational knowledge needed to run KubeVirt so a human doesn't have to do it by hand.

```text
virt-operator
      │
      └── "Keep KubeVirt installed and configured correctly."
```

### `virt-api`

**Purpose:** the KubeVirt API layer.

Handles KubeVirt-specific API operations, including things a plain Kubernetes API server has no concept of — like opening a VM's serial console or VNC session.

```text
User → virt-api → KubeVirt API operations (console, VNC, VM subresources)
```

### `virt-controller`

**Purpose:** the main decision-making controller for VM resources.

Watches `VirtualMachine`/`VirtualMachineInstance` objects and drives the cluster toward the requested state — this is the component that turns "VM should be running" into an actual VMI object and a scheduled workload. It also participates in operations like live migration (coordinating the process, not doing the data transfer itself).

```text
User: "VM should be running"
        │
        ▼
virt-controller notices
        │
        ▼
VMI is created → VM workload is prepared
```

## The node agents

### `virt-handler`

**Purpose:** node-level VM work, one instance per node (runs as a DaemonSet).

Watches for VMIs assigned to its node and keeps the actual state in sync with what's declared — including node-centric operations like configuring networking or storage for that specific VM according to its spec.

```text
virt-controller decides which node → virt-handler executes VM setup on that node
```

### `virt-launcher`

**Purpose:** the Pod environment in which a VM actually runs.

There is normally one `virt-launcher` Pod per running VMI. This is the seam where "Kubernetes workload" becomes "real virtual machine" — the Pod runs libvirt plus QEMU to provide the actual virtualization environment.

```text
VMI → virt-launcher Pod → QEMU → KVM → Guest OS
```

### `virtctl`

**Purpose:** the CLI companion to `kubectl` for VM-specific operations.

`kubectl` can manage KubeVirt objects like any other Kubernetes resource, but VMs are stateful in ways plain Kubernetes workloads aren't — they can be paused, live-migrated, and need console/VNC access. `virtctl` provides that missing vocabulary: start, stop, pause, unpause, migrate, console, vnc.

## Storage

Storage is where VMs diverge most from ordinary stateless containers: a VM has an OS disk it boots from, reads, writes, and expects to persist. Kubernetes already provides the storage primitive — the **PersistentVolumeClaim (PVC)** — and KubeVirt's job is connecting VM disks to it.

```text
Storage backend → PersistentVolume → PersistentVolumeClaim → VM disk → Guest OS
```

**Common disk approaches:**

- **`containerDisk`** — a VM disk image packaged as a container image, pulled from a registry like any other image. Good for testing and disposable workloads; not persistent storage, since it's read-only and backed by the image layer.
- **`DataVolume`** — the workflow for getting real disk data into persistent storage (a PVC) that survives VM restarts. This is what you use for anything that needs to keep its data.

```text
containerDisk:  Container registry → containerDisk → VM        (disposable)
DataVolume:     Disk image → DataVolume → PVC → VM              (persistent)
```

Full lifecycle detail (attaching, resizing, hotplug) belongs with VM management in [Virtual Machines](./04-virtual-machines.md#storage-in-practice) — this section is the concept-level map.

## CDI — Containerized Data Importer

**CDI** is a separate, companion project used alongside KubeVirt specifically for VM disk preparation. Its job is narrow and specific:

> Take VM disk data from a source, and place it into Kubernetes storage.

Sources it supports: HTTP URLs, container registries, another PVC (cloning), manually uploaded images, or blank disks.

```text
                Disk Source
                     │
       ┌─────────────┼─────────────┐
       │             │             │
      HTTP        Registry       PVC
       │             │             │
       └─────────────┼─────────────┘
                     ▼
                   CDI
                     │
                     ▼
                DataVolume
                     │
                     ▼
                    PVC
                     │
                     ▼
                  VM Disk
```

**Why is this a separate project rather than built into core KubeVirt?** Kubernetes provides storage, but has no concept of "import a VM image from a URL." That's a virtualization-specific problem, so it's solved by a purpose-built companion rather than folded into either Kubernetes or core KubeVirt:

```text
Kubernetes → provides storage
CDI        → prepares/imports VM disk data
KubeVirt   → uses the resulting storage as VM disks
```

## Networking

A container's network interface belongs to the Pod. A VM's network interface belongs to the **guest operating system** — so KubeVirt has to bridge one more layer than a container does:

```text
Kubernetes network → VM network setup → Virtual NIC → Guest OS
```

**Binding types** (how the VM's virtual NIC connects to the Pod network):

- **Masquerade** — the VM talks through the Pod network using NAT. Simplest default option; the VM effectively looks like it's behind the Pod's IP.

  ```text
  VM → Virtual NIC → NAT/Masquerade → Pod network → Kubernetes network
  ```

- **Bridge** — the VM interface connects more directly to the Pod network rather than through NAT. Gives the VM different network visibility, but has different migration and addressing considerations than masquerade — check current behavior against [kubevirt.io/user-guide](https://kubevirt.io/user-guide/) before relying on specifics, since binding options have evolved across releases.

- **Multus / additional networks** — for when a VM needs more than the single default cluster network (e.g., a dedicated storage network or an external VLAN). Multus lets a Pod (and therefore a VM) attach extra network interfaces beyond the primary one.

  ```text
                    VM
                   /  \
                  /    \
          Cluster Network   Additional Network (via Multus)
  ```

The right binding depends entirely on environment and requirements — there's no universally-correct default beyond "start with masquerade unless you have a specific reason not to."

## Services — exposing VMs

A VM can be exposed with an ordinary Kubernetes **Service**, exactly like a Deployment would be. This matters because it means all the normal Kubernetes traffic-routing concepts apply to VM workloads without modification:

```text
Client → Kubernetes Service → VM
```

Services give stable access to a VM workload even as the underlying instance changes (e.g., after a migration or restart) — same value proposition as for Pods. Depending on the cluster, standard Service types apply (ClusterIP, LoadBalancer, etc.).

> A VM does not have to live outside Kubernetes networking just because it's a VM.

## Reusable VM configuration objects

Covered here because they're supporting *components* of the system; using them day-to-day is in [Virtual Machines](./04-virtual-machines.md#configuration-with-instancetypes-and-preferences).

- **InstanceType** — reusable CPU/memory sizing, so you're not repeating `2 CPU / 4 GiB` across every VM spec.
- **Preference** — reusable device/firmware preferences (disk bus, NIC model, firmware type) — a separate axis from sizing.
- **Template** — a fuller VM blueprint combining storage, networking, DataVolumes, and configuration for spinning up multiple similar VMs.

```text
InstanceType → How big should the VM be?
Preference   → How should the VM's devices behave?
Template     → The complete reusable blueprint
```

## Where to go next

- Ready to put these pieces to use creating and running actual VMs? → [04-virtual-machines.md](./04-virtual-machines.md)
- Need to lock storage/network access down by team or namespace? → [05-security-rbac.md](./05-security-rbac.md)
- Just need the term-by-term lookup? → [06-reference-glossary.md](./06-reference-glossary.md)
