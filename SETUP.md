# Nginx GitOps on OpenShift (ArgoCD) — Setup Guide

## Overview

This repo demonstrates a **3-level App of Apps** GitOps pattern with two ArgoCD environments on OpenShift.

```
uber/                              ← uber app watches this
  argo-test.yaml                   ← parent app for test environment
  argo-staging.yaml                ← parent app for staging environment

environments/
  argo-test/                       ← child apps for test environment
    nginx.yaml                     ← points to manifests/nginx-test
    webapp.yaml                    ← points to manifests/webapp-test
  argo-staging/                    ← child apps for staging environment
    nginx.yaml                     ← points to manifests/nginx-staging
    webapp.yaml                    ← points to manifests/webapp-staging

manifests/
  nginx-test/                      ← nginx reverse proxy (namespace: nginx-test)
  nginx-staging/                   ← nginx reverse proxy (namespace: nginx-staging)
  webapp-test/                     ← Python webapp, orange TEST badge
  webapp-staging/                  ← Python webapp, green STAGING badge

infra/
  argo-test/
    argocd-instance.yaml           ← ArgoCD CR for test environment
    appproject.yaml                ← allows argo-test, argo-staging, nginx-test
    rolebinding.yaml               ← grants ArgoCD access to nginx-test
  argo-staging/
    argocd-instance.yaml           ← ArgoCD CR for staging environment
    appproject.yaml                ← allows argo-staging, nginx-staging
    rolebinding.yaml               ← grants ArgoCD access to nginx-staging
  uber-app.yaml                    ← applied manually to argo-test ArgoCD
```

**Traffic flow:** `Browser → Route → nginx (proxy) → Python webapp`

**How the levels work:**
1. **Uber app** (on `argo-test` ArgoCD) watches `uber/` and creates one parent Application per environment
2. **Parent apps** watch `environments/<env>/` and create nginx + webapp Applications
3. **Child apps** deploy from their environment-specific `manifests/` directory with hardcoded namespaces

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

## Step 1 — Deploy ArgoCD instances

### argo-test

```bash
kubectl create namespace argo-test
kubectl apply -f infra/argo-test/argocd-instance.yaml
kubectl get argocd argocd -n argo-test -w
```

### argo-staging

```bash
kubectl create namespace argo-staging
kubectl apply -f infra/argo-staging/argocd-instance.yaml
kubectl get argocd argocd -n argo-staging -w
```

Wait until both show `STATUS: Available`.

Get ArgoCD URLs and admin passwords:

```bash
kubectl get route argocd-server -n argo-test
kubectl get route argocd-server -n argo-staging
kubectl get secret argocd-cluster -n argo-test -o jsonpath='{.data.admin\.password}' | base64 -d
kubectl get secret argocd-cluster -n argo-staging -o jsonpath='{.data.admin\.password}' | base64 -d
```

---

## Step 2 — Set up the test environment

```bash
# Create and label the namespace
kubectl create namespace nginx-test
kubectl label namespace nginx-test argocd.argoproj.io/managed-by=argo-test

# Apply AppProject and RoleBinding
kubectl apply -f infra/argo-test/appproject.yaml
kubectl apply -f infra/argo-test/rolebinding.yaml
```

---

## Step 3 — Set up the staging environment

```bash
# Create and label the namespace
kubectl create namespace nginx-staging
kubectl label namespace nginx-staging argocd.argoproj.io/managed-by=argo-staging

# Apply AppProject and RoleBinding
kubectl apply -f infra/argo-staging/appproject.yaml
kubectl apply -f infra/argo-staging/rolebinding.yaml
```

---

## Step 4 — Apply the uber app

The uber app runs on `argo-test` ArgoCD and creates parent Application CRs for both environments:

```bash
kubectl apply -f infra/uber-app.yaml
```

From this point everything is GitOps-driven:
- Uber app creates `argo-test` and `argo-staging` parent apps
- `argo-test` ArgoCD picks up the `argo-test` parent and deploys nginx + webapp to `nginx-test`
- `argo-staging` ArgoCD picks up the `argo-staging` parent and deploys nginx + webapp to `nginx-staging`

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `destination not allowed` | Namespace missing from AppProject | Check `infra/<env>/appproject.yaml` destinations |
| `namespace is not managed` | Namespace not labeled | `kubectl label namespace <ns> argocd.argoproj.io/managed-by=<argo-ns>` |
| `can not be managed when in namespaced mode` | `Namespace` manifest in git | Remove it — create namespace manually |
| nginx `Permission denied` / CrashLoopBackOff | Standard nginx runs as root | Use `nginxinc/nginx-unprivileged` on port 8080 |
| webapp `Permission denied` / CrashLoopBackOff | Python image runs as root | Set `runAsNonRoot: true` and `HOME=/tmp` |
| Pods stuck in `Pending` | Insufficient memory on CRC | Reduce replicas to 1 |
| UI shows "No applications" | `kubeadmin` not in `system:cluster-admins` group | RBAC pre-configured in `argocd-instance.yaml` |
| Staging apps not appearing | `argo-staging` ArgoCD not watching `argo-staging` namespace | Ensure `argo-staging` ArgoCD is installed in `argo-staging` namespace |

---

## Teardown

```bash
# Delete the uber app — cascades to parent apps, then child apps, then workloads
# Wait for all resources to be pruned before continuing
kubectl delete -f infra/uber-app.yaml

# Delete AppProjects and RoleBindings
kubectl delete -f infra/argo-test/appproject.yaml
kubectl delete -f infra/argo-test/rolebinding.yaml
kubectl delete -f infra/argo-staging/appproject.yaml
kubectl delete -f infra/argo-staging/rolebinding.yaml

# Delete workload namespaces
kubectl delete namespace nginx-test
kubectl delete namespace nginx-staging

# Delete ArgoCD instances and their namespaces
kubectl delete -f infra/argo-test/argocd-instance.yaml
kubectl delete -f infra/argo-staging/argocd-instance.yaml
kubectl delete namespace argo-test
kubectl delete namespace argo-staging
```

---

## Verify

```bash
# argo-test ArgoCD — should show uber-app, argo-test parent, nginx-test, webapp-test
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
# argo-test
NAME          SYNC STATUS   HEALTH STATUS
uber-app      Synced        Healthy
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
