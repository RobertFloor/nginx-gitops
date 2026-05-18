# Nginx GitOps on OpenShift (ArgoCD) — Setup Guide

## Overview

This repo demonstrates a **3-level App of Apps** GitOps pattern with a root ArgoCD (`argo-root`) that bootstraps two environment-level ArgoCD instances.

```
infra/
  argo-root/
    argocd-instance.yaml   ← manually installed root ArgoCD
    appproject.yaml        ← AppProject for the uber app
  uber-app.yaml            ← manually applied once to argo-root

uber/                      ← uber app watches this directory
  argo-test-instance.yaml  ← creates ArgoCD in argo-test namespace
  argo-staging-instance.yaml ← creates ArgoCD in argo-staging namespace
  argo-test-appproject.yaml
  argo-staging-appproject.yaml
  argo-test.yaml           ← parent app; picked up by argo-test ArgoCD
  argo-staging.yaml        ← parent app; picked up by argo-staging ArgoCD

environments/
  argo-test/               ← watched by argo-test ArgoCD
    rolebinding.yaml       ← grants argo-test access to nginx-test
    nginx.yaml             ← points to manifests/nginx-test
    webapp.yaml            ← points to manifests/webapp-test
  argo-staging/            ← watched by argo-staging ArgoCD
    rolebinding.yaml       ← grants argo-staging access to nginx-staging
    nginx.yaml             ← points to manifests/nginx-staging
    webapp.yaml            ← points to manifests/webapp-staging

manifests/
  nginx-test/              ← nginx reverse proxy (namespace: nginx-test)
  nginx-staging/           ← nginx reverse proxy (namespace: nginx-staging)
  webapp-test/             ← Python webapp, orange TEST badge
  webapp-staging/          ← Python webapp, green STAGING badge
```

**Traffic flow:** `Browser → Route → nginx (proxy) → Python webapp`

**How the levels work:**
1. **`argo-root`** ArgoCD (manually installed) runs the uber app
2. **Uber app** watches `uber/` and creates ArgoCD instances + parent Application CRs for each environment
3. **`argo-test`** ArgoCD picks up its parent app, which deploys nginx + webapp to `nginx-test`
4. **`argo-staging`** ArgoCD picks up its parent app, which deploys nginx + webapp to `nginx-staging`

---

## Prerequisites

- OpenShift cluster (CRC or otherwise)
- `kubectl` / `oc` CLI configured
- Git repo pushed to GitHub

---

## Step 0 — Install the OpenShift GitOps Operator

Via the OpenShift web console: go to **OperatorHub** and install **OpenShift GitOps**.

Or via CLI:

```bash
kubectl create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

---

## Step 1 — Create namespaces

All namespaces must be created manually. The `argocd.argoproj.io/managed-by` label tells the OpenShift GitOps operator which ArgoCD instance gets RBAC access to that namespace.

```bash
# Root ArgoCD namespace (no managed-by label — argo-root owns itself)
kubectl create namespace argo-root

# Environment ArgoCD namespaces — managed by argo-root so it can create resources there
kubectl create namespace argo-test
kubectl label namespace argo-test argocd.argoproj.io/managed-by=argo-root

kubectl create namespace argo-staging
kubectl label namespace argo-staging argocd.argoproj.io/managed-by=argo-root

# Workload namespaces — managed by their respective environment ArgoCD
kubectl create namespace nginx-test
kubectl label namespace nginx-test argocd.argoproj.io/managed-by=argo-test

kubectl create namespace nginx-staging
kubectl label namespace nginx-staging argocd.argoproj.io/managed-by=argo-staging
```

---

## Step 2 — Install the root ArgoCD

```bash
kubectl apply -f infra/argo-root/argocd-instance.yaml
kubectl get argocd argocd -n argo-root -w
```

Wait until `STATUS: Available`.

Get the URL and admin password:

```bash
kubectl get route argocd-server -n argo-root
kubectl get secret argocd-cluster -n argo-root -o jsonpath='{.data.admin\.password}' | base64 -d
```

---

## Step 3 — Apply the AppProject and uber app

```bash
kubectl apply -f infra/argo-root/appproject.yaml
kubectl apply -f infra/uber-app.yaml
```

From this point everything is GitOps-driven:

1. Uber app creates `argo-test` and `argo-staging` ArgoCD instances, AppProjects, and parent Application CRs
2. `argo-test` ArgoCD picks up its parent app and deploys nginx + webapp to `nginx-test`
3. `argo-staging` ArgoCD picks up its parent app and deploys nginx + webapp to `nginx-staging`

Get the environment ArgoCD URLs and passwords once they spin up:

```bash
kubectl get route argocd-server -n argo-test
kubectl get route argocd-server -n argo-staging
kubectl get secret argocd-cluster -n argo-test -o jsonpath='{.data.admin\.password}' | base64 -d
kubectl get secret argocd-cluster -n argo-staging -o jsonpath='{.data.admin\.password}' | base64 -d
```

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `destination not allowed` | Namespace missing from AppProject | Check AppProject destinations |
| `namespace is not managed` | Namespace not labeled | `kubectl label namespace <ns> argocd.argoproj.io/managed-by=<argo-ns>` |
| `can not be managed when in namespaced mode` | `Namespace` manifest in git | Remove it — create namespaces manually |
| nginx `Permission denied` / CrashLoopBackOff | Standard nginx runs as root | Use `nginxinc/nginx-unprivileged` on port 8080 |
| webapp `Permission denied` / CrashLoopBackOff | Python image runs as root | Set `runAsNonRoot: true` and `HOME=/tmp` |
| Pods stuck in `Pending` | Insufficient memory on CRC | Reduce replicas to 1 |
| UI shows "No applications" | `kubeadmin` not in `system:cluster-admins` group | RBAC pre-configured in ArgoCD CRs via `rbac.policy` |
| Staging apps not appearing | `argo-staging` ArgoCD not watching `argo-staging` namespace | Ensure `argo-staging` ArgoCD is installed in `argo-staging` namespace |
| Namespace stuck in `Terminating` | Application finalizers blocking deletion | Remove finalizers: `kubectl patch application <name> -n <ns> --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'` |

---

## Teardown

```bash
# Remove finalizers from all Applications first — otherwise deleting the namespace
# will deadlock (the finalizer needs the ArgoCD controller which is also being deleted)
for ns in argo-root argo-test argo-staging; do
  for app in $(kubectl get application -n $ns -o name 2>/dev/null); do
    kubectl patch $app -n $ns --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'
  done
done

# Delete workload namespaces
kubectl delete namespace nginx-test nginx-staging

# Delete ArgoCD instances and their namespaces
kubectl delete -f infra/argo-root/argocd-instance.yaml
kubectl delete namespace argo-test argo-staging argo-root
```

---

## Verify

```bash
# argo-root ArgoCD — should show uber-app
kubectl get application -n argo-root

# argo-test ArgoCD — should show argo-test parent, nginx-test, webapp-test
kubectl get application -n argo-test

# argo-staging ArgoCD — should show argo-staging parent, nginx-staging, webapp-staging
kubectl get application -n argo-staging

# Workloads
kubectl get pods -n nginx-test
kubectl get pods -n nginx-staging

# Routes
kubectl get route nginx -n nginx-test
kubectl get route nginx -n nginx-staging
```

Expected output:

```
# argo-root
NAME        SYNC STATUS   HEALTH STATUS
uber-app    Synced        Healthy

# argo-test
NAME          SYNC STATUS   HEALTH STATUS
argo-test     Synced        Healthy
nginx-test    Synced        Healthy
webapp-test   Synced        Healthy

# argo-staging
NAME             SYNC STATUS   HEALTH STATUS
argo-staging     Synced        Healthy
nginx-staging    Synced        Healthy
webapp-staging   Synced        Healthy
```

Open the Route URLs in a browser — the test environment shows an orange **TEST** badge, staging shows a green **STAGING** badge.
