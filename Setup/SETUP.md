# Setup — K3s + KubeVirt

This document installs the complete base platform. It intentionally uses one path: **K3s → KubeVirt → virtctl → CDI → kubevirt-manager**.

## 0. Target

- Ubuntu Server / Linux host
- x86_64/amd64
- KVM available
- Single-node K3s for the initial setup
- KubeVirt for VMs
- CDI for VM disk import/upload workflows
- kubevirt-manager for the web UI

> Do not mix this guide with the kubeadm path from older notes. K3s already provides the Kubernetes runtime/networking foundation used here.

## 1. Prepare the host

Update the system:

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic tools:

```bash
sudo apt install -y curl wget git jq vim qemu-utils cpu-checker
```

Check the OS and architecture:

```bash
cat /etc/os-release
uname -m
```

Expected architecture for the commands below: `x86_64`.

## 2. Verify hardware virtualization

Check CPU virtualization support:

```bash
lscpu | grep -E 'Virtualization|VT-x|AMD-V'
```

Check KVM:

```bash
ls -l /dev/kvm
kvm-ok
```

If the host is itself a VM, enable nested virtualization in the outer hypervisor before continuing.

For a complete checklist, see [`VALIDATION.md`](VALIDATION.md#hardware-virtualization).

## 3. Install K3s

Install K3s:

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the service:

```bash
sudo systemctl status k3s
```

Check the node:

```bash
sudo k3s kubectl get nodes
```

## 4. Configure kubectl for the current user

Create a user-owned kubeconfig:

```bash
mkdir -p ~/.kube
sudo k3s kubectl config view --raw > ~/.kube/config
chmod 600 ~/.kube/config
export KUBECONFIG=$HOME/.kube/config
```

Persist it for future shells:

```bash
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

Confirm:

```bash
kubectl get nodes
kubectl get pods -A
```

## 5. Check storage before KubeVirt

List StorageClasses:

```bash
kubectl get storageclass
```

List persistent volumes:

```bash
kubectl get pv
```

Do not add a distributed storage system at this stage unless the deployment specifically requires one. Keep the first installation simple.

## 6. Install KubeVirt

Get the stable KubeVirt release:

```bash
export KUBEVIRT_VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
echo "$KUBEVIRT_VERSION"
```

Install the operator:

```bash
wget https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-operator.yaml
kubectl apply -f kubevirt-operator.yaml
```

Install the KubeVirt custom resource:

```bash
wget https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/kubevirt-cr.yaml
kubectl apply -f kubevirt-cr.yaml
```

Wait for KubeVirt:

```bash
kubectl -n kubevirt wait kv kubevirt --for condition=Available
```

Check it:

```bash
kubectl get kv -n kubevirt
kubectl get pods -n kubevirt
```

## 7. Install virtctl

Use the same KubeVirt version:

```bash
wget https://github.com/kubevirt/kubevirt/releases/download/${KUBEVIRT_VERSION}/virtctl-${KUBEVIRT_VERSION}-linux-amd64
chmod +x virtctl-${KUBEVIRT_VERSION}-linux-amd64
sudo mv virtctl-${KUBEVIRT_VERSION}-linux-amd64 /usr/local/bin/virtctl
```

Check:

```bash
virtctl version
```

## 8. Install CDI

Install the CDI operator and custom resource. The source setup notes currently pin CDI to `v1.62.0`; keep the version consistent across a rebuild rather than mixing releases.

```bash
export CDI_VERSION=v1.62.0
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/${CDI_VERSION}/cdi-cr.yaml
```

Wait and check:

```bash
kubectl get pods -n cdi -w
```

When healthy, stop the watch with `Ctrl+C` and run:

```bash
kubectl get pods -n cdi
```

## 9. Install kubevirt-manager

Apply the bundled deployment:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubevirt-manager/kubevirt-manager/main/kubernetes/bundled.yaml
```

Pin the manager image to the known working image tag used by the source setup:

```bash
kubectl set image deployment/kubevirt-manager -n kubevirt-manager \
  kubevirtmgr=kubevirtmanager/kubevirt-manager:1.5.4
```

Wait for rollout:

```bash
kubectl rollout status deployment/kubevirt-manager -n kubevirt-manager
```

Check:

```bash
kubectl get pods -n kubevirt-manager
kubectl get svc -n kubevirt-manager
```

## 10. Expose kubevirt-manager

For a simple lab, use NodePort:

```bash
kubectl patch svc kubevirt-manager -n kubevirt-manager \
  -p '{"spec":{"type":"NodePort"}}'
```

Find the assigned port:

```bash
kubectl get svc kubevirt-manager -n kubevirt-manager
```

Open:

```text
http://<server-ip>:<node-port>
```

Do not assume a fixed NodePort. Always read the current value from the Service.

## 11. Final platform check

Run:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
kubectl get kv -n kubevirt
kubectl get pods -n kubevirt
kubectl get pods -n cdi
kubectl get pods -n kubevirt-manager
virtctl version
```

The platform is ready for VM work when the node is `Ready`, KubeVirt is `Available`, required pods are healthy, CDI is running, the manager is running, and `virtctl` works.

Continue with [`VALIDATION.md`](VALIDATION.md) before creating production workloads. For VM creation, use the VM section there as the standalone operational starting point; for failures, use [`TROUBLESHOOTING.md`](TROUBLESHOOTING.md).
