# 6. Reference & Glossary

*Part of the [KubeVirt Notes](./README.md) series. Previous: [Security & RBAC](./05-security-rbac.md) · Back to [Start](./README.md)*

Fast-lookup doc — no new concepts, everything here is defined in full elsewhere and linked. Use this when you already know the material and just need to jog your memory or check a term.

## The five things to remember

**1. KubeVirt extends Kubernetes. It does not replace it.** Every other point follows from this. Details: [Introduction](./01-introduction.md).

**2. VMs are Kubernetes resources.** You manage them through the Kubernetes API, with `kubectl`/`virtctl`, same mental model as Pods and Deployments.

```text
VirtualMachine
VirtualMachineInstance
```

**3. Kubernetes still does the platform work.** Scheduling, networking primitives, storage primitives, namespaces, RBAC — all unchanged. KubeVirt adds only the VM-specific behavior on top. Details: [Architecture](./02-architecture.md#kubernetes-vs-kubevirt-who-owns-what).

**4. The VM is a real VM.** The guest OS isn't pretending to be a container — it's a genuine QEMU/KVM virtual machine.

```text
virt-launcher → QEMU → KVM → Guest OS
```

**5. KubeVirt connects two worlds.**

```text
Kubernetes world              VM world

Pods                         Guest OS
Deployments                  Virtual hardware
Services                     VM disks
Kubernetes networking        VM networking
Kubernetes RBAC              VM access
       │                          │
       └──────── KubeVirt ────────┘
```

## One-diagram summary

```text
                         USER
                           │
                           ▼
                    Kubernetes API
                           │
                           ▼
                  ┌─────────────────┐
                  │     KubeVirt    │
                  │  VM objects     │
                  │  Controllers    │
                  │  VM lifecycle   │
                  └────────┬────────┘
                           │
                           ▼
                   Kubernetes Scheduler
                           │
                           ▼
                    Kubernetes Node
                    ┌──────┴──────┐
                    │ virt-handler│
                    │     ▼       │
                    │ virt-launcher
                    │     ▼       │
                    │    QEMU     │
                    │     ▼       │
                    │    KVM      │
                    │     ▼       │
                    │  Guest OS   │
                    └─────────────┘

Storage:      Disk source → CDI → PVC → VM disk
Networking:   Kubernetes network → VM network interface → Guest OS
Security:     Kubernetes RBAC → VM permissions
```

## Glossary

| Term | Meaning | See also |
|---|---|---|
| **KubeVirt** | Kubernetes extension for running and managing VMs | [Intro](./01-introduction.md) |
| **VirtualMachine (VM)** | Persistent Kubernetes resource; desired state of a VM | [VMs §VM vs VMI](./04-virtual-machines.md#virtualmachine-vs-virtualmachineinstance) |
| **VirtualMachineInstance (VMI)** | The actual running VM instance; exists only while running | [VMs §VM vs VMI](./04-virtual-machines.md#virtualmachine-vs-virtualmachineinstance) |
| **virt-api** | Control-plane component; KubeVirt API layer (console, VNC, VM ops) | [Components](./03-components.md#virt-api) |
| **virt-controller** | Control-plane component; watches VMs, drives desired state | [Components](./03-components.md#virt-controller) |
| **virt-operator** | Control-plane component; manages KubeVirt's own install/lifecycle | [Components](./03-components.md#virt-operator) |
| **virt-handler** | Node agent (DaemonSet); does per-node VM setup | [Components](./03-components.md#virt-handler) |
| **virt-launcher** | Node agent (Pod); actually runs the VM via QEMU/KVM | [Components](./03-components.md#virt-launcher) |
| **virtctl** | CLI companion to `kubectl` for VM-specific ops (start/stop/migrate/console) | [Components](./03-components.md#virtctl) |
| **QEMU** | Process that creates/runs the virtual machine and virtual hardware | [Intro](./01-introduction.md#where-does-the-vm-actually-run) |
| **KVM** | Linux kernel capability giving QEMU hardware-assisted virtualization | [Intro](./01-introduction.md#where-does-the-vm-actually-run) |
| **CRD** | Kubernetes mechanism KubeVirt uses to add its new object types | [Architecture](./02-architecture.md) |
| **CDI** | Containerized Data Importer — companion project for VM disk import | [Components](./03-components.md#cdi--containerized-data-importer) |
| **DataVolume** | Object managing persistent VM disk data preparation (often via CDI) | [Components](./03-components.md#storage) |
| **PVC** | Kubernetes PersistentVolumeClaim; underlying storage for VM disks | [Components](./03-components.md#storage) |
| **containerDisk** | VM disk image packaged as a container image; ephemeral, not persistent | [Components](./03-components.md#storage) |
| **cloud-init** | Standard mechanism for configuring a guest OS on first boot | [VMs](./04-virtual-machines.md) |
| **InstanceType** | Reusable VM CPU/memory sizing definition | [Components](./03-components.md#reusable-vm-configuration-objects) |
| **Preference** | Reusable VM device/firmware preference definition | [Components](./03-components.md#reusable-vm-configuration-objects) |
| **VM Template** | Reusable full VM blueprint (sizing + storage + networking) | [Components](./03-components.md#reusable-vm-configuration-objects) |
| **Service** | Kubernetes resource exposing network access to a VM, same as a Pod | [Components](./03-components.md#services--exposing-vms) |
| **Masquerade** | Default VM networking mode: NAT through the Pod network | [Components](./03-components.md#networking) |
| **Bridge** | VM networking mode: more direct connection to the Pod network | [Components](./03-components.md#networking) |
| **Multus** | Mechanism for attaching additional networks to a VM beyond the default | [Components](./03-components.md#networking) |
| **RBAC** | Kubernetes authorization system; governs all KubeVirt resources too | [Security & RBAC](./05-security-rbac.md) |
| **Namespace** | Logical boundary for Kubernetes/KubeVirt resources; primary isolation unit | [Security & RBAC](./05-security-rbac.md#namespaces) |
| **Live Migration** | Moving a running VM between nodes with minimal disruption | [VMs §Live migration](./04-virtual-machines.md#live-migration) |

## What each layer provides, at a glance

**Kubernetes provides:** cluster management, Pod management, scheduling, namespaces, RBAC, storage primitives, networking primitives, Services, the Kubernetes API.

**KubeVirt provides:** Kubernetes-native VM resources, VM lifecycle management, VM-specific controllers, VM execution integration, VM networking integration, VM storage integration, VM access mechanisms (console/VNC), migration capabilities, integration with Kubernetes RBAC/namespaces.

**CDI provides:** VM disk import, disk cloning workflows, disk upload workflows, DataVolume-based disk preparation.

## The final mental model

```text
                 KUBERNETES
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Containers    Platform      Services
                     │
                  KubeVirt
                     │
              Virtual Machines
                     │
                 QEMU / KVM
                     │
                 Guest OS
```

**Kubernetes remains the platform. KubeVirt adds VM capabilities to that platform.** Everything in this note set is an elaboration of that one sentence.

---

*This completes the KubeVirt Notes set. Back to [README](./README.md) for the reading paths, or jump to any doc: [1 — Introduction](./01-introduction.md) · [2 — Architecture](./02-architecture.md) · [3 — Components](./03-components.md) · [4 — Virtual Machines](./04-virtual-machines.md) · [5 — Security & RBAC](./05-security-rbac.md)*
