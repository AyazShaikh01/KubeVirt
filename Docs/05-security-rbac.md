# 5. Security, RBAC & Namespaces

*Part of the [KubeVirt Notes](./README.md) series. Previous: [Virtual Machines](./04-virtual-machines.md) · Next: [Reference & Glossary →](./06-reference-glossary.md)*

> Quick recap: KubeVirt objects are ordinary Kubernetes objects (see [Architecture](./02-architecture.md#kubernetes-vs-kubevirt-who-owns-what)), which is exactly why this whole doc is short — there's no separate VM security model to learn. This doc is what actually makes a multi-team KubeVirt deployment safe, not just functional.

## The core principle

KubeVirt does not invent a new authorization or isolation model for VMs. It uses Kubernetes' existing namespace and RBAC systems, the same way it reuses scheduling and storage:

```text
User → Kubernetes RBAC → KubeVirt resources
```

If you already know Kubernetes RBAC, you already know most of what you need here. The only KubeVirt-specific piece is *which permissions exist* for VM operations (below) — not *how* permissions work.

## Namespaces

VM resources live in Kubernetes namespaces exactly like any other object:

```text
Namespace A
├── VM
├── VM
└── DataVolume

Namespace B
├── VM
└── DataVolume
```

This is the primary isolation boundary between teams or applications — give each team its own namespace and their VMs, DataVolumes, and related objects are naturally scoped and separated by the same mechanism you'd use for Pods and Deployments.

One distinction worth keeping straight: **KubeVirt's own control-plane components** (`virt-api`, `virt-controller`, `virt-operator` — see [Components](./03-components.md#the-control-plane)) live in their own dedicated namespace (typically something like `kubevirt`), separate from the namespaces holding actual VM workloads. Don't conflate "the namespace KubeVirt runs in" with "the namespaces where your VMs live" — they're deliberately different, same as core Kubernetes system components living in `kube-system` while your app Pods live elsewhere.

## RBAC

Kubernetes RBAC controls *who can do what to which resources*, and KubeVirt resources are governed by exactly that system — no separate VM permission model to administer:

```text
User → Kubernetes RBAC → KubeVirt resources (VirtualMachine, VMI, DataVolume, ...)
```

**Operations RBAC can gate**, all as ordinary verb/resource rules:

- viewing VMs (`get`/`list`/`watch` on `virtualmachines`)
- creating or modifying VMs (`create`/`update`/`patch`/`delete`)
- accessing a VM's console or VNC (a VM subresource — see [Virtual Machines](./04-virtual-machines.md#accessing-a-running-vm))
- other VM-specific operations exposed as subresources (start/stop/migrate/pause)

KubeVirt ships predefined **ClusterRoles** for common access levels (roughly: view-only, and edit/admin-level access to VM resources), meant to be referenced from your own `RoleBinding`/`ClusterRoleBinding` objects rather than reinvented per cluster. The exact role names and granularity are version-dependent — check `kubectl get clusterroles | grep kubevirt` on your cluster or the [KubeVirt user guide](https://kubevirt.io/user-guide/) rather than assuming a specific set, since this is one of the areas that shifts across releases.

**Practical pattern for a multi-tenant PoC:**

1. One namespace per team/tenant.
2. A `RoleBinding` in each namespace binding users/groups to the appropriate KubeVirt ClusterRole, scoped to that namespace only.
3. Console/VNC access reviewed separately if it matters for your threat model — it's effectively interactive shell-adjacent access to a full guest OS, which is a meaningfully different risk than "can edit a Deployment spec."

## What this does *not* cover

Worth being explicit about, since it's easy to assume RBAC is a complete security story:

- **Guest OS security** (patching, hardening the OS inside the VM) is entirely your responsibility — Kubernetes RBAC governs the Kubernetes object, not what happens once you're inside the guest.
- **Network policy** (which VMs/Pods can talk to which) is a separate mechanism — standard Kubernetes `NetworkPolicy` objects apply to VM Pods the same way they apply to any Pod, but they're not part of RBAC and need to be set up independently.
- **Node-level isolation** (which nodes VMs can be scheduled on, hardware-level tenancy) is handled by standard Kubernetes scheduling constraints (taints/tolerations, node selectors), not by anything KubeVirt-specific.

Security in a KubeVirt cluster is the sum of Kubernetes RBAC + namespace isolation + NetworkPolicy + guest-OS hardening + node scheduling constraints — the same four-or-five-layer picture as securing any Kubernetes workload, just with a guest OS added to the "your responsibility" list.

## Where to go next

- Need the component names referenced by ClusterRoles above? → [03-components.md](./03-components.md)
- Want to see console/VNC access in the context of actually operating a VM? → [04-virtual-machines.md](./04-virtual-machines.md#accessing-a-running-vm)
- Fast lookup of every term? → [06-reference-glossary.md](./06-reference-glossary.md)
