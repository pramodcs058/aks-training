# Lab 16 – Apply Pod Security Admission (Baseline) on AKS

## Lab details

| Item | Description |
|---|---|
| **Module** | Module 6 – Cluster & Workload Security |
| **Suggested duration** | 15 minutes |
| **Execution environment** | Azure Portal → Cloud Shell → Bash |
| **Difficulty** | Basic |

## Problem statement

Create a Kubernetes namespace and apply the Pod Security Admission (PSA) `baseline` enforcement label. Deploy a workload that complies with the baseline policy, then attempt to deploy a workload that violates it and inspect the admission response.

## Objective

Understand Pod Security Standards and how namespace-level security enforcement controls which Pods can be created.

## Expected outcome

Participants will be able to apply the `baseline` Pod Security Admission policy to a namespace, deploy a compliant Pod, identify a baseline violation, and interpret the rejection returned by the Kubernetes API server.

## Concepts

- **Pod Security Standards (PSS):** Kubernetes defines three security profiles: `privileged` (unrestricted), `baseline` (prevents known privilege escalations), and `restricted` (stronger hardening for security-sensitive workloads).
- **Pod Security Admission (PSA):** The built-in admission controller evaluates Pod specifications against the selected security profile when Pods are created.
- **Enforce mode:** The namespace label `pod-security.kubernetes.io/enforce=baseline` causes the API server to reject Pods that violate the baseline profile.
- **Baseline versus Restricted:** Baseline blocks settings such as `privileged: true`; it does not require every Restricted control, such as running as a non-root user.

> **AKS note:** Pod Security Admission is enabled by default in AKS. This lab assumes an existing supported AKS cluster and a user identity with permission to create namespaces and Pods.

## Prerequisites

- Access to an existing AKS cluster.
- Azure Portal access to open **Cloud Shell** and select **Bash**.
- `kubectl` configured for the target AKS cluster. The commands below include an optional credential-refresh step.
- Permission to create and delete Kubernetes namespaces and Pods.

---

## Lab steps

### Step 1 – Open Cloud Shell and connect to AKS

In the Azure Portal, select **Cloud Shell**, choose **Bash**, and confirm the active subscription and Kubernetes context.

```bash
az account show --output table
kubectl config current-context
kubectl cluster-info
```

**Command explanation:** `az account show` confirms the active Azure subscription. `kubectl config current-context` and `kubectl cluster-info` confirm which Kubernetes cluster receives your commands.

If the expected AKS context is not active, set the resource group and cluster name, then retrieve the credentials:

```bash
RESOURCE_GROUP="<RESOURCE_GROUP_NAME>"
AKS_CLUSTER_NAME="<AKS_CLUSTER_NAME>"
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER_NAME" --overwrite-existing
kubectl config current-context
```

Replace the placeholders with the names of your AKS resource group and cluster.

### Step 2 – Create a dedicated namespace

A namespace provides a scope in which the Pod Security Admission policy will be applied. This lab uses a separate namespace so that the policy does not change settings for existing training workloads.

```bash
NAMESPACE="psa-baseline-lab"
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
```

**Command explanation:** The command creates the namespace if it does not exist. Using `--dry-run=client -o yaml | kubectl apply -f -` makes the operation safe to repeat.

### Step 3 – Enforce the Baseline policy

Apply the `enforce=baseline` label to the namespace.

```bash
kubectl label namespace "$NAMESPACE" pod-security.kubernetes.io/enforce=baseline --overwrite
```

**Command explanation:** The label tells Pod Security Admission to reject any new Pod in this namespace that violates the Baseline Pod Security Standard. `--overwrite` updates the label if it already exists.

Verify the label:

```bash
kubectl get namespace "$NAMESPACE" --show-labels
```

**Expected observation:** The namespace labels should include:

```text
pod-security.kubernetes.io/enforce=baseline
```

### Step 4 – Deploy a Pod that complies with Baseline

Create a simple NGINX Pod without privileged settings or other baseline violations.

```bash
cat <<'MANIFEST' > baseline-allowed-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-allowed
  namespace: psa-baseline-lab
  labels:
    app: baseline-allowed
spec:
  containers:
    - name: nginx
      image: nginx:stable-alpine
      ports:
        - containerPort: 80
MANIFEST
```

Apply the manifest and wait for the Pod to become Ready:

```bash
kubectl apply -f baseline-allowed-pod.yaml
kubectl wait --for=condition=Ready pod/baseline-allowed -n "$NAMESPACE" --timeout=120s
kubectl get pod baseline-allowed -n "$NAMESPACE" -o wide
```

**Command explanation:** `kubectl apply` submits the Pod specification to the API server. `kubectl wait` waits for the Pod readiness condition, and `kubectl get pod` displays its status and node placement.

**Expected observation:** The Pod should be created and reach `Running` with `1/1` containers ready. The Baseline profile permits ordinary container workloads such as this NGINX Pod.

### Step 5 – Create a Pod that violates Baseline

The following Pod sets `privileged: true`. Privileged containers are prohibited by the Baseline profile.

```bash
cat <<'MANIFEST' > baseline-violating-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: baseline-violating
  namespace: psa-baseline-lab
  labels:
    app: baseline-violating
spec:
  containers:
    - name: nginx
      image: nginx:stable-alpine
      securityContext:
        privileged: true
MANIFEST
```

Submit the manifest:

```bash
kubectl apply -f baseline-violating-pod.yaml
```

**Expected observation:** The API server should reject the request with a `Forbidden` error similar to:

```text
Error from server (Forbidden): error when creating "baseline-violating-pod.yaml":
pods "baseline-violating" is forbidden: violates PodSecurity "baseline:latest":
privileged (container "nginx" must not set securityContext.privileged=true)
```

The exact wording and policy version shown in the message can vary with the Kubernetes version.

**Command explanation:** `kubectl apply` sends the Pod specification to the Kubernetes API server. The PSA admission controller evaluates it before the Pod is stored; because `privileged: true` violates Baseline, the API server rejects the request.

### Step 6 – Verify the admission result

List the Pods in the lab namespace:

```bash
kubectl get pods -n "$NAMESPACE"
```

**Expected observation:** Only `baseline-allowed` should appear. The `baseline-violating` Pod should not exist because the API server rejected its creation.

You can confirm the rejected Pod is absent:

```bash
kubectl get pod baseline-violating -n "$NAMESPACE"
```

**Expected observation:** Kubernetes returns `NotFound` for `baseline-violating`.

> **Important:** Inspect the `Forbidden` response returned by `kubectl apply` to understand why admission failed. A rejected Pod is never created, so `kubectl describe pod` cannot show it, and you should not expect a normal Pod event for that rejected object.

### Step 7 – Review the namespace policy

```bash
kubectl describe namespace "$NAMESPACE"
kubectl get namespace "$NAMESPACE" --show-labels
```

Confirm that the namespace still has `pod-security.kubernetes.io/enforce=baseline` and that the compliant Pod remains available.

---

## Validation checklist

- [ ] Connected to the intended AKS cluster from Azure Cloud Shell (Bash).
- [ ] Created the `psa-baseline-lab` namespace.
- [ ] Applied and verified `pod-security.kubernetes.io/enforce=baseline`.
- [ ] Successfully deployed the compliant NGINX Pod.
- [ ] Submitted the Pod manifest containing `privileged: true`.
- [ ] Observed the API server reject the violating Pod with a `Forbidden` / `violates PodSecurity` response.
- [ ] Verified that the rejected Pod was not created.

## Troubleshooting

| Issue | Possible cause | Resolution |
|---|---|---|
| `kubectl` connects to the wrong cluster | Incorrect kubeconfig context | Run `kubectl config get-contexts` and select the intended context with `kubectl config use-context <CONTEXT_NAME>`, or refresh credentials using `az aks get-credentials`. |
| `Forbidden` while creating the namespace | The signed-in identity lacks namespace permissions | Ask the cluster administrator to grant the required Kubernetes RBAC permissions. |
| The violating Pod is created | The manifest was applied to a different namespace, or the Baseline enforce label is missing | Check `metadata.namespace` in the manifest and verify the namespace label with `kubectl get namespace "$NAMESPACE" --show-labels`. |
| The compliant Pod does not become Ready | Image pull, scheduling, or node issue | Run `kubectl describe pod baseline-allowed -n "$NAMESPACE"` and `kubectl logs baseline-allowed -n "$NAMESPACE"` to inspect the cause. |
| Error says the namespace already exists | The lab namespace was created in an earlier run | Continue with Step 3, or delete the namespace and recreate it if it contains no resources you need. |

## Cleanup

Delete the lab namespace and all resources created inside it:

```bash
kubectl delete namespace "$NAMESPACE"
```

Optionally remove the local manifest files from Cloud Shell:

```bash
rm -f baseline-allowed-pod.yaml baseline-violating-pod.yaml
```

## Knowledge check

1. What is the purpose of the `pod-security.kubernetes.io/enforce=baseline` namespace label?
2. What is the difference between the `privileged`, `baseline`, and `restricted` Pod Security Standards?
3. Why does the API server return `Forbidden` when the privileged Pod is submitted?
4. Why can you not use `kubectl describe` to inspect the rejected Pod?
5. Does the Baseline profile require every container to run as a non-root user?

## Key takeaways

- Pod Security Admission applies Pod Security Standards at the namespace level.
- `enforce=baseline` blocks Pods that violate the Baseline security requirements.
- A regular container workload can be admitted, while a Pod that requests privileged mode is rejected.
- Admission rejection happens before the Pod object is persisted in the cluster.
- Baseline is less restrictive than Restricted; passing Baseline does not mean a workload satisfies every hardening requirement.

## References

- [Microsoft Learn – Use Pod Security Admission in AKS](https://learn.microsoft.com/en-us/azure/aks/use-psa)
- [Kubernetes – Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes – Enforce Pod Security Standards with Namespace Labels](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Kubernetes – Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
