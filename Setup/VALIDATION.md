# Validation — K3s + KubeVirt

Use this file independently when you need to prove that a host, cluster, or VM platform is ready. Run checks in order for a new installation; individual sections can be used on an existing system.

## 1. Hardware virtualization

Architecture:

```bash
uname -m
```

CPU virtualization:

```bash
lscpu | grep -E 'Virtualization|VT-x|AMD-V'
```

KVM device:

```bash
ls -l /dev/kvm
```

KVM check:

```bash
kvm-ok
```

If `/dev/kvm` is missing or `kvm-ok` reports that acceleration cannot be used, fix the host or nested-virtualization configuration before installing KubeVirt.

## 2. K3s

Service:

```bash
sudo systemctl is-active k3s
```

Node:

```bash
kubectl get nodes -o wide
```

Expected: the node is `Ready`.

System pods:

```bash
kubectl get pods -n kube-system
```

## 3. Kubernetes access

```bash
kubectl auth can-i get pods
kubectl cluster-info
```

The current user should be able to communicate with the cluster without `sudo`.

## 4. Storage

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -A
```

At least one usable StorageClass should be available for the VM disk workflow.

## 5. KubeVirt

```bash
kubectl get kv -n kubevirt
kubectl get pods -n kubevirt
```

Then:

```bash
kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

The KubeVirt resource should report availability and the namespace should have healthy components.

## 6. virtctl

```bash
virtctl version
```

If it is installed but not found:

```bash
which virtctl
echo "$PATH"
```

## 7. CDI

```bash
kubectl get pods -n cdi
```

CDI components should be healthy before using DataVolumes or disk import/upload workflows.

## 8. kubevirt-manager

```bash
kubectl get pods -n kubevirt-manager
kubectl get svc kubevirt-manager -n kubevirt-manager
```

Check the deployed image:

```bash
kubectl get deployment kubevirt-manager -n kubevirt-manager \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

If using the source setup, this should show the pinned `kubevirtmanager/kubevirt-manager:1.5.4` image.

## 9. End-to-end VM validation

After creating a VM:

```bash
kubectl get vm
kubectl get vmi
```

Start it if necessary:

```bash
virtctl start <vm-name>
```

Check its state:

```bash
kubectl get vm <vm-name>
kubectl get vmi <vm-name>
```

Open the console:

```bash
virtctl console <vm-name>
```

A VM that reaches `Running` and provides a working console demonstrates the basic KubeVirt execution path.

## 10. Final validation checklist

```text
[ ] CPU virtualization available
[ ] /dev/kvm exists
[ ] kvm-ok succeeds
[ ] K3s service is active
[ ] Kubernetes node is Ready
[ ] kubectl works without sudo
[ ] StorageClass exists
[ ] KubeVirt is Available
[ ] KubeVirt pods are healthy
[ ] virtctl works
[ ] CDI pods are healthy
[ ] kubevirt-manager pod is healthy
[ ] kubevirt-manager Service is reachable
[ ] First VM starts
[ ] First VM console works
```

If any item fails, stop at that layer and use [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md).
