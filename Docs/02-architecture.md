# 2. Architecture

*Part of the [KubeVirt Notes](./README.md) series. Previous: [Introduction](./01-introduction.md) · Next: [Components →](./03-components.md)*

> Quick recap if you're jumping in here: KubeVirt extends Kubernetes with VM-specific API objects, controllers, and node agents, while Kubernetes keeps doing scheduling, storage, networking, and RBAC. Full context in [Introduction](./01-introduction.md).

## The architectural principle

> **Add only the VM-specific pieces. Reuse Kubernetes wherever possible.**

Everything about KubeVirt's design follows from this one rule. There are exactly three categories of addition:

```text
1. API objects  (VirtualMachine, VirtualMachineInstance, DataVolume, ...)
       │
       ▼
2. Controllers  (virt-operator, virt-controller)
       │
       ▼
3. Node-level VM components  (virt-handler, virt-launcher)
```

## Kubernetes vs KubeVirt: who owns what

This table is the single most useful reference for understanding where a given piece of functionality lives:

| Area | Kubernetes | KubeVirt |
|---|---|---|
| Containers | Manages them | Uses Kubernetes |
| Pods | Manages them | Uses Pods to host VM processes |
| Scheduling | Provides it | Uses Kubernetes scheduling |
| Storage | Provides PVCs and storage primitives | Connects VM disks to them |
| Networking | Provides the cluster network | Connects VM interfaces to it |
| Namespaces | Provides them | Uses them for VM resources |
| RBAC | Provides permissions | Uses Kubernetes RBAC for VM resources |
| VM API | Not provided | Adds VM-specific API objects |
| VM lifecycle | Not provided | Manages start/stop/run behavior |
| VM execution | Not provided | Integrates with QEMU/KVM |

**Kubernetes provides the platform. KubeVirt provides the virtualization capability.** Keep this split in mind — it resolves most "wait, who's responsible for this?" questions later, especially in [Components](./03-components.md) and [Security & RBAC](./05-security-rbac.md).

## High-level layout

```text
                    User
                     │
                     │ Kubernetes API
                     ▼
             ┌─────────────────┐
             │ Kubernetes API  │
             └────────┬────────┘
                      │
                      ▼
             ┌─────────────────┐
             │    KubeVirt     │   ← control plane (runs as Deployments)
             │                 │
             │  virt-api       │
             │  virt-controller│
             │  virt-operator  │
             └────────┬────────┘
                      │
                      ▼
              Kubernetes Scheduler
                      │
                      ▼
              ┌──────────────────┐
              │ Kubernetes Node  │   ← node agents (run as a DaemonSet + Pod)
              │                  │
              │ virt-handler     │
              │      │           │
              │      ▼           │
              │ virt-launcher    │
              │      │           │
              │      ▼           │
              │     QEMU         │
              │      │           │
              │     KVM          │
              │      │           │
              │    Guest OS      │
              └──────────────────┘
```

Two distinct tiers, matching how any Kubernetes controller-based system is shaped:

- **Control plane** (cluster-wide, runs as Deployments): `virt-operator`, `virt-api`, `virt-controller`. These decide *what should happen*.
- **Node agents** (per-node, `virt-handler` as a DaemonSet, `virt-launcher` per running VM): these make it *actually happen* on a specific node.

Full responsibilities of each component are in [Components](./03-components.md#the-control-plane) — this doc focuses on how they interact.

## How a VM actually starts

This is more useful to internalize than memorizing component names in isolation — it's the same reconcile loop pattern used everywhere in Kubernetes, just applied to VMs.

**Step 1 — Desired state is declared.** A user creates a `VirtualMachine` object: "I want this VM to exist, running."

**Step 2 — KubeVirt notices.** `virt-controller` watches `VirtualMachine` objects. If one should be running, it creates the corresponding `VirtualMachineInstance` (VMI) — the object representing the actual running instance.

```text
VirtualMachine  →  VirtualMachineInstance
```

**Step 3 — Kubernetes schedules it.** The VMI's execution environment is a Pod. The normal Kubernetes scheduler places that Pod on a node, exactly like any other workload.

```text
VMI → Pod → Kubernetes Scheduler → Node
```

**Step 4 — The node prepares the VM.** `virt-handler`, running on the selected node, picks up the node-level work — networking and storage setup for that specific VM.

**Step 5 — The VM is launched.** The `virt-launcher` Pod starts the actual virtualization process:

```text
virt-launcher → QEMU → KVM → Guest OS
```

**Step 6 — KubeVirt keeps watching.** It doesn't start the VM and walk away. Like every Kubernetes controller, it runs a continuous reconcile loop:

```text
Desired state → Observe → Compare → Act → Observe again
```

This loop is *why* things like restart-on-failure and consistent state reporting work without any manual intervention — the same mechanism you'd already trust for Deployments is now doing it for VMs.

## The object model, structurally

KubeVirt's most important new objects, and how they relate:

```text
VirtualMachine               (desired state, persists across stop/start)
    │
    └── VirtualMachineInstance   (the live running thing, exists only while running)

VirtualMachineInstanceReplicaSet   (multiple identical VMIs — rarely used directly; VirtualMachine is the normal way to manage a VM)
```

Supporting objects (covered fully in [Components](./03-components.md) and [Virtual Machines](./04-virtual-machines.md)) handle storage (`DataVolume`), sizing (`InstanceType`), device defaults (`Preference`), templates, snapshots, and migration. The important structural idea: **a VM is represented entirely through Kubernetes API objects** — there's no separate, external "VM inventory" to keep in sync.

The VM/VMI split itself — arguably *the* core KubeVirt concept — is explained in full in [Virtual Machines §VM vs VMI](./04-virtual-machines.md#virtualmachine-vs-virtualmachineinstance), since that's where you'll actually use the distinction day to day.

## Where to go next

- Want the full responsibility breakdown of every component named above? → [03-components.md](./03-components.md)
- Ready to actually create/manage a VM using this model? → [04-virtual-machines.md](./04-virtual-machines.md)
- Need a one-screen diagram of the whole system? → [06-reference-glossary.md](./06-reference-glossary.md#one-diagram-summary)
