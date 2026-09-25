# Troubleshooting & Rollback — K3s + KubeVirt

Use this file when a setup or validation step fails. Work from the lowest failed layer upward; do not reinstall the whole stack for a component-level problem.

## 1. First diagnostic commands

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
```

For a failing pod:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
```

For a failing deployment:

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
```

## 2. Hardware / KVM problems

### `/dev/kvm` does not exist

Check:

```bash
ls -l /dev/kvm
lscpu | grep -E 'Virtualization|VT-x|AMD-V'
```

If the host is a VM, verify nested virtualization in the outer hypervisor. Do not proceed with KubeVirt until KVM is available.

### `kvm-ok` fails

```bash
kvm-ok
```

Resolve BIOS/UEFI virtualization or nested virtualization first.

## 3. K3s problems

Check:

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -n 100 --no-pager
```

Restart only after identifying a likely service-level issue:

```bash
sudo systemctl restart k3s
```

Then:

```bash
kubectl get nodes
kubectl get pods -A
```

### kubectl works only with sudo

Check:

```bash
echo "$KUBECONFIG"
ls -l ~/.kube/config
```

Recreate the user kubeconfig if needed:

```bash
mkdir -p ~/.kube
sudo k3s kubectl config view --raw > ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=$HOME/.kube/config
```

## 4. Node is `NotReady`

Start with:

```bash
kubectl describe node <node-name>
kubectl get pods -n kube-system
kubectl get events -A --sort-by=.lastTimestamp
```

Do not install KubeVirt until the Kubernetes node is healthy.

## 5. KubeVirt is not Available

```bash
kubectl get kv -n kubevirt
kubectl get pods -n kubevirt
kubectl get events -n kubevirt --sort-by=.lastTimestamp
```

Inspect the failing pod:

```bash
kubectl describe pod <pod> -n kubevirt
kubectl logs <pod> -n kubevirt
```

If the problem points to virtualization, return to the hardware checks in this file and [`VALIDATION.md`](VALIDATION.md#hardware-virtualization).

## 6. CDI is not healthy

```bash
kubectl get pods -n cdi
kubectl get events -n cdi --sort-by=.lastTimestamp
```

Inspect a failing pod:

```bash
kubectl describe pod <pod> -n cdi
kubectl logs <pod> -n cdi
```

If disk imports fail, also check:

```bash
kubectl get storageclass
kubectl get pvc -A
kubectl get pv
```

## 7. kubevirt-manager is not working

Check:

```bash
kubectl get pods -n kubevirt-manager
kubectl get svc -n kubevirt-manager
kubectl rollout status deployment/kubevirt-manager -n kubevirt-manager
```

Check the image:

```bash
kubectl get deployment kubevirt-manager -n kubevirt-manager \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

The source setup pins the image to:

```text
kubevirtmanager/kubevirt-manager:1.5.4
```

If the UI loads without styling/assets, verify that the deployment was not left on the `nightly` image and repin it using the setup command.

### NodePort is unreachable

Find the current port:

```bash
kubectl get svc kubevirt-manager -n kubevirt-manager
```

Check the service endpoints:

```bash
kubectl get endpoints kubevirt-manager -n kubevirt-manager
```

Then check host firewall/network rules for the assigned NodePort.

## 8. VM will not start

```bash
kubectl get vm
kubectl get vmi
kubectl describe vm <vm-name>
kubectl get events -A --sort-by=.lastTimestamp
```

Check the VM disk objects:

```bash
kubectl get dv,pvc,pv
```

Check KubeVirt pods if the error looks cluster-wide:

```bash
kubectl get pods -n kubevirt
```

## 9. VM starts but has no console/network

Console:

```bash
virtctl console <vm-name>
```

Check the VMI:

```bash
kubectl get vmi <vm-name> -o yaml
```

For networking, first verify the basic network path before adding Multus or other advanced networking components.

## 10. Rollback: kubevirt-manager

Remove the manager resources installed by the bundled manifest:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubevirt-manager/kubevirt-manager/main/kubernetes/bundled.yaml
```

Verify:

```bash
kubectl get ns kubevirt-manager
kubectl get pods -A | grep -i kubevirt-manager
```

## 11. Rollback: CDI

Delete the CDI resources using the same version used during installation:

```bash
export CDI_VERSION=v1.62.0
kubectl delete -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml
kubectl delete -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml
```

Check:

```bash
kubectl get pods -n cdi
```

Do not delete PVCs or VM disks unless you intentionally want to remove their data.

## 12. Rollback: KubeVirt

If you need to remove KubeVirt, use the same release manifests used during installation:

```bash
kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml
kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml
```

Verify:

```bash
kubectl get pods -n kubevirt
kubectl get crd | grep kubevirt
```

Treat KubeVirt removal as a destructive maintenance operation and verify what VM/storage resources you still need before deleting anything.

## 13. Rollback: K3s

Do not use a full K3s uninstall as the first troubleshooting step. K3s removal destroys the Kubernetes installation and can remove cluster state.

Before removing K3s, back up anything important and confirm that the goal is a complete rebuild.

For service-level issues, prefer:

```bash
sudo systemctl status k3s
sudo journalctl -u k3s -n 200 --no-pager
```

## 14. Recovery rule

Use this order:

```text
Hardware/KVM
    ↓
K3s
    ↓
Kubernetes node/network
    ↓
StorageClass
    ↓
KubeVirt
    ↓
CDI
    ↓
kubevirt-manager
    ↓
VM
    ↓
VM networking/storage
```

Fix the first failed layer before debugging the layers above it.
