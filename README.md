# Karmada Multi-Cluster Setup with a Shared Ingress

A step-by-step guide to installing the [Karmada](https://github.com/karmada-io/karmada) control plane with Helm, exposing its API server through an NGINX Ingress, and joining a Kind member cluster in **agent** (pull) mode so workloads can be propagated to it.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Step 1: Install the Karmada Control Plane](#step-1-install-the-karmada-control-plane)
- [Step 2: Expose karmada-apiserver with an Ingress](#step-2-expose-karmada-apiserver-with-an-ingress)
- [Step 3: Extract the Karmada kubeconfig](#step-3-extract-the-karmada-kubeconfig)
- [Step 4: Install the Karmada Agent on a Member Cluster](#step-4-install-the-karmada-agent-on-a-member-cluster)
- [Step 5: Propagate a Workload](#step-5-propagate-a-workload)
- [Configuration Reference](#configuration-reference)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [References](#references)
- [License](#license)
- [Author](#author)

## Overview

1. Install Karmada core components (API server, scheduler, controller-manager, ...) in the `karmada-system` namespace of a host cluster.
2. Publish `karmada-apiserver` through an NGINX Ingress using SSL passthrough, so member clusters can reach it by hostname.
3. Install the Karmada agent (`installMode: "agent"`) on a member cluster, pointing it at the Ingress hostname.
4. Create a `PropagationPolicy` and a Deployment in the Karmada API server to schedule workloads onto member clusters.

## Prerequisites

- A host Kubernetes cluster with an NGINX Ingress controller (the Ingress uses `ingressClassName: nginx`).
- A member cluster (the guide uses Kind clusters).
- `kubectl`, `helm`, and optionally `yq` / `jq` for extracting the kubeconfig.
- A DNS name for the Karmada API server and a TLS secret for it. The examples use a placeholder hostname; replace `karmada.example.com` and the TLS secret name with your own.
- The Karmada Helm chart. The guide installs it from `./charts/karmada` (a checkout of the Karmada repository); the repository also contains `karmada-0.0.0.tgz`. TODO: confirm which chart source the author intends.

## Repository Structure

| Path | Description |
|------|-------------|
| `README.md` | This guide |
| `karmada-ingress.yml` | Ingress for `karmada-apiserver` (port 5443, SSL passthrough, backend protocol HTTPS) |
| `values.yaml` | Helm values for the Karmada agent (`installMode: "agent"`); contains kubeconfig material and must be filled in with your own values (see [Security Notes](#security-notes)) |
| `rbac.yaml` | ClusterRoleBinding granting `cluster-admin` to `system:anonymous` (see [Security Notes](#security-notes)) |
| `examples/propagationpolicy.yaml` | Example `PropagationPolicy` (also contains commented-out alternatives) |
| `examples/workload-test.yml` | Example nginx Deployment used with the policy |
| `karmada-0.0.0.tgz` | Packaged Karmada Helm chart |
| `(Not needed)-kind-member-*.yaml` | Earlier `Cluster` / impersonator Secret examples for registering member clusters in push mode; marked "Not needed" by the author |

## Step 1: Install the Karmada Control Plane

```bash
helm repo add karmada-charts https://raw.githubusercontent.com/karmada-io/karmada/master/charts
helm install karmada -n karmada-system --create-namespace --dependency-update ./charts/karmada
```

This installs the Karmada API server, scheduler, controller-manager and related components in the `karmada-system` namespace.

## Step 2: Expose karmada-apiserver with an Ingress

Apply `karmada-ingress.yml` (edit the host and TLS secret first):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
  name: karmada
  namespace: karmada-system
spec:
  ingressClassName: nginx
  rules:
  - host: karmada.example.com
    http:
      paths:
      - backend:
          service:
            name: karmada-apiserver
            port:
              number: 5443
        path: /
        pathType: ImplementationSpecific
  tls:
  - hosts:
    - karmada.example.com
    secretName: <your-tls-secret>
```

If you run into issues, the author's notes suggest these additional annotations:

```yaml
nginx.ingress.kubernetes.io/upstream-vhost: karmada-apiserver.karmada-system.svc
nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
nginx.ingress.kubernetes.io/proxy-buffer-size: 8k
nginx.ingress.kubernetes.io/proxy-buffers-number: "4"
nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/secure-backends: "true"
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

## Step 3: Extract the Karmada kubeconfig

The Helm deployment creates a secret (`karmada-kubeconfig` in `karmada-system`) holding the Karmada API server kubeconfig. Decode it to a file:

```bash
# kubectl + jq
kubectl get secret karmada-kubeconfig -n karmada-system -o json | jq -r '.data.kubeconfig' | base64 -d > karmada.config

# kubectl only (jsonpath)
kubectl get secret karmada-kubeconfig -n karmada-system -o jsonpath='{.data.kubeconfig}' | base64 -d > karmada.config
```

(On macOS use `base64 -D`.) Check the file:

```bash
kubectl --kubeconfig=karmada.config config view
```

The kubeconfig points at the in-cluster address `https://karmada-apiserver.karmada-system.svc.cluster.local:5443`. Replace the `server:` value with your Ingress URL (`https://karmada.example.com`), then verify:

```bash
export KUBECONFIG=karmada.config
kubectl config get-contexts
```

Treat `karmada.config` as a credential; do not commit it.

## Step 4: Install the Karmada Agent on a Member Cluster

Create a `values.yaml` for the agent (use the values from `karmada.config`; never commit real ones):

```yaml
installMode: "agent"
agent:
  clusterName: "kind-whitelotus"              # name of the member cluster
  clusterEndpoint: "https://127.0.0.1:46339"  # member cluster endpoint
  kubeconfig:
    caCrt: |                                  # certificate-authority-data from karmada.config
      -----BEGIN CERTIFICATE-----
      xxx
      -----END CERTIFICATE-----
    crt: |                                    # client-certificate-data
      -----BEGIN CERTIFICATE-----
      xxx
      -----END CERTIFICATE-----
    key: |                                    # client-key-data
      -----BEGIN RSA PRIVATE KEY-----
      xxx
      -----END RSA PRIVATE KEY-----
    server: "https://karmada.example.com"
```

Install it against the member cluster:

```bash
helm install karmada karmada/karmada --namespace karmada-system --create-namespace -f values.yaml
```

Verify from the Karmada side; the member cluster should show as a member with status `Ready`:

```bash
export KUBECONFIG=karmada.config
kubectl config get-contexts
kubectl get clusters
```

Verify the agent on the member cluster:

```bash
export KUBECONFIG=whitelotus.config   # member cluster kubeconfig
kubectl get pods -n karmada-system
```

## Step 5: Propagate a Workload

Apply these to the Karmada API server (see `examples/`):

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-policy
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment-23
  placement:
    clusterAffinity:
      clusterNames:
        - kind-member3
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-23
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Make sure `clusterNames` in the policy matches the member cluster name you registered. The repository examples use different names and resource names than this snippet (`examples/propagationpolicy.yaml` uses `nginx-deployment-233` and `kind-white123` / `kind-member3`), so keep policy and Deployment names consistent.

## Configuration Reference

| Setting | Where | Meaning |
|---------|-------|---------|
| `installMode` | agent `values.yaml` | `agent` installs only the Karmada agent on a member cluster |
| `agent.clusterName` | agent `values.yaml` | Name under which the member cluster registers |
| `agent.clusterEndpoint` | agent `values.yaml` | API endpoint of the member cluster |
| `agent.kubeconfig.caCrt` / `crt` / `key` | agent `values.yaml` | CA, client certificate and key used to talk to karmada-apiserver |
| `agent.kubeconfig.server` | agent `values.yaml` | Public URL of karmada-apiserver (the Ingress host) |
| `ssl-passthrough` | Ingress annotation | Lets TLS terminate at karmada-apiserver (port 5443) |

## Troubleshooting

- Ingress or TLS problems: try the extra annotations listed in [Step 2](#step-2-expose-karmada-apiserver-with-an-ingress).
- Wrong server address: the generated kubeconfig uses the in-cluster service address; replace it with the Ingress URL.
- Member cluster not listed: check `kubectl get pods -n karmada-system` on the member cluster and `kubectl get clusters` against Karmada. TODO: no further troubleshooting is documented in the repository.

## Security Notes

- `values.yaml` and the `(Not needed)-...impersonator.yaml` file hold certificate/key and token material. Never commit real credentials; treat anything that was committed as compromised and rotate it.
- `rbac.yaml` binds `system:anonymous` to `cluster-admin`, which gives unauthenticated users full cluster access. Do not apply it to any cluster you care about.
- Keep `karmada.config` private.

## References

- [Karmada](https://github.com/karmada-io/karmada)
- [Karmada documentation](https://karmada.io/docs/)
- [Kind](https://kind.sigs.k8s.io/)

## License

No license file is included. The repository bundles a packaged copy of the Apache-2.0 licensed Karmada chart (`karmada-0.0.0.tgz`); TODO: the owner should decide on a license for the original content.

## Author

[Umesh Khatiwada](https://github.com/umesh-khatiwada)
