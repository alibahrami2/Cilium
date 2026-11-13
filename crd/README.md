# Cilium CNI Offline Deployment

This repository provides the Custom Resource Definitions (CRDs) and modular Helm values required to deploy **Cilium CNI** (with eBPF `kube-proxy` replacement, Gateway API, Envoy, and Hubble) in an air-gapped Kubernetes environment using the internal Nexus Helm repository.

## Repository Structure

```text
.
├── crd/
│   └── all-cilium-crds.yaml     # Pre-packaged Cilium CRDs
├── values.yaml                  # Base configuration (private registry images, Hubble, eBPF, Gateway API)
├── values-l2.yaml               # Override for L2 Announcements (ARP-based LoadBalancer/Gateway IPs)
├── values-hostnetwork.yaml      # Override for HostNetwork mode (exposes Gateway directly on node IPs)
└── README.md
```

## Choosing a Deployment Mode

The configuration is split into a shared base file (`values.yaml`) and two mode-specific override files so you can choose how external traffic enters the cluster:

| Feature | L2 Mode (`values-l2.yaml`) | HostNetwork Mode (`values-hostnetwork.yaml`) |
| :--- | :--- | :--- |
| **How it works** | Cilium responds to ARP requests on the local network to advertise dedicated virtual IPs (`LoadBalancer` / `Gateway`). | `cilium-envoy` binds directly to the host network interface (`hostNetwork: true`) and listens on node IPs. |
| **When to use** | You have a free subnet range of IP addresses dedicated to Kubernetes `LoadBalancer` and `Gateway` resources. | You do **not** have spare IP addresses for `LoadBalancer` pools, or you route traffic from an external LB (HAProxy/Nginx) directly to node IPs. |
| **Port requirements** | Virtual IPs use their own ports; no conflict with node host ports. | Ports defined in your `Gateway` (e.g., `80`, `443`) must be free on the target Kubernetes nodes. |

## Prerequisites

- A running Kubernetes cluster provisioned without an existing CNI plugin.
- `kubectl` and `helm` (v3+) installed with cluster-admin permissions.
- Network and DNS access to `https://registry.aradarpanet.ir` and `kube.aradarpanet.ir`.

## Installation Steps

### 1. Clone the Repository

```bash
git clone [https://github.com/alibahrami2/Cilium.git](https://github.com/alibahrami2/Cilium.git)
cd Cilium
```

### 2. Apply Cilium CRDs

Register the Cilium Custom Resource Definitions before installing the Helm chart:

```bash
kubectl apply -f crd/all-cilium-crds.yaml
```

> **Tip:** If `kubectl apply` returns a `metadata.annotations: Too long` error on large CRD manifests, run:
> ```bash
> kubectl apply --server-side -f crd/all-cilium-crds.yaml
> ```

### 3. Add and Update the Internal Helm Repository

```bash
helm repo add arad [https://registry.aradarpanet.ir/repository/Helm-Hosted/](https://registry.aradarpanet.ir/repository/Helm-Hosted/)
helm repo update
```

### 4. Install Cilium

Choose **one** of the following deployment modes:

#### Option A: L2 Announcements Mode

```bash
helm install cilium arad/cilium \
  -n kube-system \
  -f values.yaml \
  -f values-l2.yaml
```

#### Option B: HostNetwork Mode

```bash
helm install cilium arad/cilium \
  -n kube-system \
  -f values.yaml \
  -f values-hostnetwork.yaml
```

*(If your internal Nexus TLS certificate is not trusted at the OS level, append `--insecure-skip-tls-verify` to both `helm repo add` and `helm install`.)*

## Verification

1. **Verify Cilium Pods:**
   Ensure all `cilium`, `cilium-envoy`, and `cilium-operator` pods are `Running`:
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium
   ```

2. **Verify Node Status:**
   Confirm that all nodes have transitioned to `Ready`:
   ```bash
   kubectl get nodes -o wide
   ```

3. **Check Cilium Health (Optional — requires `cilium` CLI):**
   ```bash
   cilium status
   ```

## Upgrading, Switching Modes & Uninstalling

- **Upgrade or switch between L2 and HostNetwork modes:**
  Run `helm upgrade` with the desired mode file, then restart the `cilium` and `cilium-envoy` DaemonSets so network bindings take effect:
  ```bash
  helm upgrade cilium arad/cilium -n kube-system -f values.yaml -f values-hostnetwork.yaml
  kubectl rollout restart ds -n kube-system cilium cilium-envoy
  ```

- **Uninstall Cilium:**
  ```bash
  helm uninstall cilium -n kube-system
  ```
