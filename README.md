# Kubernetes Cluster Security and Access Control

## Overview

This project focuses on securing a Kubernetes cluster using production-level security practices.
The goal is to control who can access resources, block unsafe workloads, and scan container images before they are used.

This project applies access control, pod security rules, policy enforcement, and image security checks.

---

## Step 1: Create Role for Namespace Access

Create `role.yaml`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-user-role
  namespace: dev
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update"]
```

Apply role.

```bash
kubectl apply -f role.yaml
```

---

## Step 2: Bind Role to User

Create `rolebinding.yaml`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-user-binding
  namespace: dev
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-user-role
  apiGroup: rbac.authorization.k8s.io
```

Apply binding.

```bash
kubectl apply -f rolebinding.yaml
```

---

## Step 3: Verify RBAC Permissions

Check permissions.

```bash
kubectl auth can-i get pods -n dev --as=dev-user
kubectl auth can-i delete pods -n dev --as=dev-user
kubectl auth can-i get nodes --as=dev-user
```

---

## Step 4: Enforce Pod Security Standards

Apply restricted policy to namespace.

```bash
kubectl label namespace dev \
pod-security.kubernetes.io/enforce=restricted
```

Verify labels.

```bash
kubectl get namespace dev --show-labels
```

---

## Step 5: Test Pod Security Enforcement

Try creating an unsafe pod.

```bash
kubectl run insecure-pod \
--image=nginx \
--namespace=dev \
--overrides='{"spec":{"hostNetwork":true}}'
```

Pod creation should be denied.

---

## Step 6: Install Kyverno Policy Engine

Install Kyverno.

```bash
kubectl apply -f https://raw.githubusercontent.com/kyverno/kyverno/main/config/release/install.yaml
```

Verify installation.

```bash
kubectl get pods -n kyverno
```

---

## Step 7: Create Image Policy

Create `policy.yaml`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: enforce
  rules:
  - name: validate-image-tag
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "latest image tag is not allowed"
      pattern:
        spec:
          containers:
          - image: "*:*"
            name: "*"
```

Apply policy.

```bash
kubectl apply -f policy.yaml
```

---

## Step 8: Test Policy Enforcement

Try creating a pod with latest tag.

```bash
kubectl run test-pod --image=nginx:latest -n dev
```

Pod creation should be denied.

---

## Step 9: Install Trivy Image Scanner

Install Trivy.

```bash
sudo apt install -y wget
wget https://github.com/aquasecurity/trivy/releases/latest/download/trivy_0.49.1_Linux-64bit.deb
sudo dpkg -i trivy_0.49.1_Linux-64bit.deb
```

---

## Step 10: Scan Container Image

Scan an image.

```bash
trivy image nginx:latest
```

The scan output shows vulnerabilities in the image.

---
