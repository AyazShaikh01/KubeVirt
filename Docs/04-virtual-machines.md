# 4. Virtual Machines — Create, Manage, & Lifecycle

*Part of the [KubeVirt Notes](./README.md) series. Previous: [Components](./03-components.md) · Next: [Security & RBAC →](./05-security-rbac.md)*

> Quick recap: KubeVirt represents VMs entirely as Kubernetes API objects, reconciled by `virt-controller`/`virt-handler`/`virt-launcher` down to real QEMU/KVM processes. See [Architecture](./02-architecture.md) for that flow. This doc is the practical, day-to-day layer on top of it.

## VirtualMachine vs VirtualMachineInstance

This is the single most important distinction in KubeVirt, so it's worth its own section even though it's introduced in [Architecture](./02-architecture.md#the-object-model-structurally).

**`VirtualMachine` (VM)** — the persistent resource. It describes *"this VM should exist, with this configuration"* and exists whether the VM is currently running or stopped. This is the object you create, edit, and generally think of as "my VM."

**`VirtualMachineInstance` (VMI)** — the actual running instance. It exists only while the VM is running; it represents *"this is the VM that is running right now."* Stop the VM and the VMI disappears; start it again and a new VMI is created.

```text
VirtualMachine
     │
     ├── stopped  → no running VMI
     │
     └── running
            │
            ▼
           VMI
            │
            ▼
       virt-launcher
            │
            ▼
          Guest OS
```

**Analogy** (useful for intuition, not literally identical):

| Kubernetes | KubeVirt |
|---|---|
| Workload declaration | `VirtualMachine` |
| Pod | `VirtualMachineInstance` |
| ReplicaSet | `VirtualMachineInstanceReplicaSet` |

In almost all real usage you manage the `VirtualMachine` object and let KubeVirt handle creating/destroying the VMI as you start and stop it — you rarely touch a VMI directly.

## Lifecycle states

```text
                ┌─────────────┐
                │   Stopped   │
                └──────┬──────┘
                       │ Start
                       ▼
                ┌─────────────┐
                │ Provisioning│   (disks/config being prepared)
                └──────┬──────┘
                       ▼
                ┌─────────────┐
                │   Starting  │
                └──────┬──────┘
                       ▼
                ┌─────────────┐
                │   Running   │
                └──────┬──────┘
                       │ Stop
                       ▼
                ┌─────────────┐
                │   Stopped   │
                └─────────────┘
```

A running VM can additionally enter **Migration** — moving to another node while staying up (detailed below) — and return to Running once complete.

The distinction to hold onto throughout: **VM = persistent resource and desired state. VMI = the currently running instance.** Every lifecycle operation below is really "change what the VM object wants, and let the controllers reconcile the VMI to match."

## Creating a VM

A `VirtualMachine` spec needs more than just an OS image — it describes the whole virtual hardware environment:

```text
VirtualMachine
│
├── CPU
├── Memory
├── Disks
├── Network
├── Devices
└── Boot configuration
```

In practice, creation follows the same pattern as any Kubernetes object: write a manifest, `kubectl apply -f vm.yaml` (or `virtctl create vm ...` for common shapes), and let the controllers take it from there. The [startup sequence](./02-architecture.md#how-a-vm-actually-starts) from Architecture applies exactly as written — declare the object, `virt-controller` creates a VMI, Kubernetes schedules the Pod, `virt-handler` preps the node, `virt-launcher` starts QEMU/KVM.

### Configuration with InstanceTypes and Preferences

Rather than repeating CPU/memory sizing across every VM manifest:

```text
VM A → 2 CPU / 4 GiB
VM B → 2 CPU / 4 GiB
VM C → 2 CPU / 4 GiB
```

define it once as a reusable **InstanceType** (e.g. `"medium"` = 2 CPU / 4 GiB) and reference it from each VM. Pair it with a **Preference** for device/firmware defaults (disk bus, NIC model, firmware type) — a separate, orthogonal axis from sizing:

```text
InstanceType → How big should the VM be?
Preference   → How should the VM's devices behave?
```

For fleets of similar VMs, a **Template** bundles InstanceType, Preference, storage, and networking into one reusable blueprint. Full definitions of all three are in [Components](./03-components.md#reusable-vm-configuration-objects); this is how you'd actually use them when writing a VM spec.

## Storage in practice

Two practical paths, both landing on a PVC underneath (concepts in [Components](./03-components.md#storage)):

- **Ephemeral/testing:** use a `containerDisk` — point at a container image, done. No persistence, resets to the base image on restart.
- **Persistent:** define a `DataVolume`, which — optionally via [CDI](./03-components.md#cdi--containerized-data-importer) — imports or clones disk data into a PVC that the VM then boots from and writes to. This is the path for anything you need to survive a restart: databases, stateful legacy apps, anything with real data.

```text
Disk image → DataVolume → PVC → VM
```

Additional data disks can be attached the same way — a VM isn't limited to a single disk.

## Networking in practice

By default, a VM gets masquerade networking (NAT through the Pod network) unless you configure otherwise — see [Components](./03-components.md#networking) for the full comparison of masquerade vs bridge vs Multus. Practically:

- Leave it as masquerade unless you have a specific reason to change it.
- Add a Multus network interface when the VM needs to reach something outside the default cluster network (a storage fabric, an external VLAN).
- Expose the VM to other workloads with a normal Kubernetes **Service** — same manifests, same `ClusterIP`/`LoadBalancer` types you'd use for any Pod.

## Accessing a running VM

Unlike a container, you often need interactive access to the guest OS itself, not just logs:

- **Console** — serial console access via `virtctl console <vm-name>`, useful for boot-time debugging or headless troubleshooting.
- **VNC** — graphical console access via `virtctl vnc <vm-name>`, for anything that needs a GUI (Windows guests, installers).

Both are served through `virt-api` (see [Components](./03-components.md#virt-api)) and are governed by the same RBAC rules as any other VM operation — see [Security & RBAC](./05-security-rbac.md#what-rbac-actually-controls).

## Common lifecycle operations

| Operation       | Effect                                                                   |
| --------------- | ------------------------------------------------------------------------ |
| Start           | VM object's desired state → running; VMI is created                      |
| Stop            | VM object's desired state → stopped; VMI is deleted, guest shuts down    |
| Restart         | Stop then start; new VMI, fresh guest boot                               |
| Pause / Unpause | Freezes/resumes the running VM process without a full stop/start         |
| Migrate         | Moves the running VMI to another node without stopping the guest (below) |

`virtctl` is the natural tool for all of these (`virtctl start`, `virtctl stop`, `virtctl migrate`, ...) since plain `kubectl` has no native vocabulary for VM-specific, stateful operations.

## Live migration

Moving a **running** VM from one node to another while it keeps running — not a stop/copy/start cycle:

```text
Node A                         Node B

┌─────────┐                   ┌─────────┐
│   VM    │  ──────────────►  │   VM    │
└─────────┘     Migration     └─────────┘
    │                              │
    └────────── keeps running ─────┘
```

Useful for node maintenance, rebalancing load, and generally reducing disruption during infrastructure changes.

**What migration needs in practice:**

- at least two nodes capable of running the VM
- storage accessible from both the source and target node (a PVC with an access mode that supports it — e.g. `ReadWriteMany`, or a storage backend that otherwise supports live attach from multiple nodes)
- compatible networking on both nodes
- sufficient CPU/memory headroom on the target
- enough network bandwidth to transfer VM state in a reasonable window

If storage or networking can't satisfy these, migration simply isn't possible for that VM — this is one of the more common reasons a PoC environment can create VMs fine but can't migrate them; check storage access modes first.

## Where to go next

- Need to control *who* can do any of this, and isolate it by team? → [05-security-rbac.md](./05-security-rbac.md)
- Want the underlying component responsibilities behind each operation above? → [03-components.md](./03-components.md)
- Fast lookup of every term used here? → [06-reference-glossary.md](./06-reference-glossary.md)
