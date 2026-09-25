# K3s + KubeVirt Lab

A small, staged guide for building a K3s-based KubeVirt platform with `virtctl`, CDI, and kubevirt-manager.

## Documentation map

| File | Purpose | Use it when |
|---|---|---|
| [`SETUP.md`](SETUP.md) | Full installation from host preparation to kubevirt-manager | Installing or rebuilding the platform |
| [`VALIDATION.md`](VALIDATION.md) | Hardware, cluster, KubeVirt, storage, UI, and VM checks | Verifying a stage or proving the platform is healthy |
| [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md) | Diagnostics, common failures, recovery, and rollback | Something fails or you need to undo a stage |
| `README.md` | Navigation and workflow | Starting from the repository root |

## Recommended order

```text
README.md
   │
   ├── SETUP.md
   │      └── install the platform
   │
   ├── VALIDATION.md
   │      └── verify each stage and the finished platform
   │
   └── TROUBLESHOOTING.md
          └── diagnose, recover, or roll back failures
```

## Platform scope

The repository builds this initial stack:

```text
Linux host
  └─ KVM
      └─ K3s
          ├─ KubeVirt
          ├─ virtctl
          ├─ CDI
          └─ kubevirt-manager
                 └─ Virtual Machines
```

The first target is a simple single-node/lab installation. Advanced storage, networking, HA, monitoring, and multi-node production design should be added only after the base platform passes validation.

## Quick links

- **Install:** [`SETUP.md`](SETUP.md)
- **Validate:** [`VALIDATION.md`](VALIDATION.md)
- **Troubleshoot / rollback:** [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md)

## Documentation rule

Run the validation checks after each major installation stage. If a stage fails, stop there and use `TROUBLESHOOTING.md` before continuing. This keeps failures isolated and makes rebuilds predictable.
