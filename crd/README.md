# Cilium CNI Offline Deployment

This repository contains the Custom Resource Definitions (CRDs) and custom Helm values required to deploy **Cilium CNI** in an air-gapped / offline Kubernetes environment using the internal Nexus Helm repository.

## Repository Structure

```text
.
├── crd/
│   └── all-cilium-crds.yaml   # Pre-packaged Cilium CRDs
├── values2.yml                # Custom Helm values configured for the private registry
└── README.md
```

## Prerequisites

- A running Kubernetes cluster provisioned without an existing CNI plugin.
- `kubectl` and `helm` (v3+) installed and configured with cluster-admin privileges.
- Network and DNS access from the deployment node to `https://registry.aradarpanet.ir`.
- Internal Root CA certificate trusted by the OS (if using HTTPS without `--insecure-skip-tls-verify`).

## Installation Steps

### 1. Clone the Repository

```bash
git clone [https://github.com/alibahrami2/Cilium.git](https://github.com/alibahrami2/Cilium.git)
cd Cilium
```

### 2. Apply Cilium CRDs

Install the Cilium Custom Resource Definitions manually before deploying the Helm chart to ensure all required API resources are registered in the cluster:

```bash
kubectl apply -f crd/all-cilium-crds.yaml
```

> **Tip:** If `kubectl apply` reports a `metadata.annotations: Too long` error on large CRD manifests, use server-side apply instead:
> ```bash
> kubectl apply --server-side -f crd/all-cilium-crds.yaml
> ```

### 3. Add and Update the Internal Helm Repository

Register the private Nexus Helm repository and fetch the latest chart index:

```bash
helm repo add arad [https://registry.aradarpanet.ir/repository/Helm-Hosted/](https://registry.aradarpanet.ir/repository/Helm-Hosted/)
helm repo update
```

### 4. Install Cilium via Helm

Deploy Cilium into the `kube-system` namespace using `values2.yml`:

```bash
helm install cilium arad/cilium \
  -n kube-system \
  -f values2.yml
```

*(If your internal Nexus certificate is not added to the system CA trust store, append `--insecure-skip-tls-verify` to both `helm repo add` and `helm install`.)*

## Verification

1. **Check Cilium Pods:**
   Verify that the `cilium` DaemonSet pods and `cilium-operator` pods are in the `Running` state:
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium
   ```

2. **Check Node Readiness:**
   Confirm that all cluster nodes have transitioned from `NotReady` to `Ready`:
   ```bash
   kubectl get nodes -o wide
   ```

3. **Check Cilium Status (Optional — requires `cilium` CLI):**
   ```bash
   cilium status
   ```

## Upgrading & Uninstalling

- **Upgrade configuration:**
  ```bash
  helm repo update
  helm upgrade cilium arad/cilium -n kube-system -f values.yaml
  ```

- **Uninstall Cilium:**
  ```bash
  helm uninstall cilium -n kube-system
  ```
