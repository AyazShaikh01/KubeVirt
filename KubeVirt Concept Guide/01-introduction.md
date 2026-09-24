# 1. Introduction

*Part of the [KubeVirt Notes](./README.md) series. Next: [Architecture →](./02-architecture.md)*

## What is KubeVirt?

KubeVirt is a Kubernetes extension that adds **virtual machine management** to a cluster. It does not replace Kubernetes and does not turn Kubernetes into a hypervisor platform — it adds the specific pieces Kubernetes is missing (VM objects, VM lifecycle logic, VM execution) while reusing everything Kubernetes already does well.

```text
Kubernetes                          Kubernetes + KubeVirt
    │                                       │
    └── Pods                    ┌───────────┴───────────┐
          └── Containers        │                       │
                              Pods               Virtual Machines
                                │                       │
                            Containers              Guest OS
```

The result: VMs become **Kubernetes objects**, managed with `kubectl`/`virtctl`, scheduled by the Kubernetes scheduler, secured by Kubernetes RBAC, and stored on Kubernetes storage — sitting right next to your Pods and Deployments in the same cluster.

## Why does KubeVirt exist?

Not every workload can be containerized cleanly. Some things:

- depend on a full, unmodified operating system
- are legacy applications nobody wants to touch
- need kernel-level or hardware-adjacent access a container can't give
- run Windows or another OS your container runtime doesn't target

Without KubeVirt, that usually means running two separate stacks:

```text
                    Infrastructure
                         │
              ┌──────────┴──────────┐
              │                     │
        Kubernetes                VM Platform
              │                     │
          Containers                VMs
```

Two platforms means two APIs, two RBAC models, two networking approaches, two storage systems, two sets of tooling and on-call knowledge. KubeVirt's pitch is to collapse that into one platform:

```text
                 Kubernetes
                      │
          ┌───────────┴───────────┐
          │                       │
      Containers                 VMs
          │                       │
         Pods                  KubeVirt
```

**The goal is not to make VMs act like containers.** A KubeVirt VM is still a real VM, running a real guest OS, on real QEMU/KVM. The goal is to make that VM *manageable* through the same platform, APIs, and operational habits you already use for containers.

## The core mental model

> **Kubernetes manages the platform. KubeVirt adds the ability to run and manage virtual machines on top of that platform.**

Kubernetes already knows how to manage workloads, nodes, networking, storage, users/permissions, and desired state. KubeVirt adds on top of that:

- VM-specific API objects
- VM lifecycle management (start/stop/restart/migrate)
- VM-specific controllers
- node-level components that actually run the VM
- integration with the underlying virtualization stack (QEMU/KVM)

```text
                    Kubernetes
                        │
        ┌───────────────┼────────────────┐
        │               │                │
     Containers       Storage         Networking
        │               │                │
        └───────────────┼────────────────┘
                        │
                     KubeVirt
                        │
              Virtual Machine
                        │
                    QEMU / KVM
                        │
                   Guest OS
```

A useful test for "is this a Kubernetes thing or a KubeVirt thing?": **ask whether Kubernetes already knows how to do it.** If yes (scheduling, storage primitives, networking, namespaces, RBAC), KubeVirt reuses it rather than reinventing it. If no (VM objects, VM lifecycle, actually running QEMU), that's what KubeVirt adds. This split is used as the organizing principle throughout this whole note set — see it worked through in detail in [Architecture](./02-architecture.md#kubernetes-vs-kubevirt-who-owns-what).

## Where does the VM actually run?

Kubernetes doesn't become a hypervisor. The real virtualization stack sits underneath, and KubeVirt's job is connecting it to the cluster:

```text
Kubernetes
    │
    └── virt-launcher Pod
            │
            └── VM process
                  │
                QEMU   ← creates/runs the virtual machine
                  │
                KVM    ← lets QEMU use hardware virtualization
                  │
              CPU / Memory
                  │
              Guest OS
```

Every running VM is backed by an ordinary-looking Pod (a **virt-launcher Pod**) that contains the process actually running the VM. This is the seam between "Kubernetes workload" and "real virtual machine" — covered in depth in [Architecture](./02-architecture.md) and [Components](./03-components.md#virt-launcher).

## What KubeVirt is not

Worth stating up front, since it prevents most confusion later:

- **Not** a replacement for Kubernetes — it depends on it entirely.
- **Not** a full hypervisor platform separate from Kubernetes — no separate control plane to learn.
- **Not** a way to make VMs behave like containers — they stay real VMs.
- **Not** a GUI product by default — it's an API and a set of Kubernetes-native controllers; UIs are optional add-ons.
- **Not** a replacement for every storage/networking technology — it integrates with what Kubernetes already provides.

## Where to go next

- Want to see how the pieces fit together mechanically? → [02-architecture.md](./02-architecture.md)
- Want the exhaustive list of components (storage, CDI, networking)? → [03-components.md](./03-components.md)
- Just want to create and manage a VM? → [04-virtual-machines.md](./04-virtual-machines.md)
- Need the fast-reference version of everything? → [06-reference-glossary.md](./06-reference-glossary.md)
