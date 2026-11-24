# Lab 24 – Troubleshoot Three Broken Application Deployments on AKS

## Lab information

| Item | Details |
|---|---|
| **Environment** | Azure Kubernetes Service (AKS) |
| **Execution environment** | Azure Portal → Cloud Shell → Bash |
| **Suggested duration** | 60–75 minutes |
| **Difficulty** | Intermediate |
| **Namespace** | `aks-troubleshooting-capstone` |

## Scenario

Three application deployments have been created in an AKS cluster, but none is working correctly. Your task is to investigate each deployment, identify its root cause, apply the appropriate fix and verify that the application becomes healthy.

The lab intentionally introduces three different failure conditions:

1. **ImagePullBackOff** – the Deployment references a container image tag that does not exist.
2. **CrashLoopBackOff** – the container starts, prints an application error and exits repeatedly.
3. **Pending** – the Pod has a `nodeSelector` that does not match any node in the cluster.

Use the same troubleshooting sequence for each workload: **observe → inspect → identify the cause → correct → verify**.

## Learning objectives

By completing this lab, you will be able to:

- Inspect Deployments, ReplicaSets and Pods using `kubectl`.
- Use `kubectl describe` and Kubernetes events to understand scheduling and container startup failures.
- Retrieve current and previous container logs using `kubectl logs`.
- Identify image-reference, application-startup and scheduling configuration problems.
- Apply corrective changes and verify that the Deployment reaches its desired replica count.

## Prerequisites

- An existing AKS cluster that is running and accessible.
- Permission to create namespaces, Deployments and Pods in the cluster.
- Azure Portal access and an active Azure Cloud Shell session using **Bash**.
- The cluster must be able to pull the public images used in this lab (`nginx:alpine` and `busybox:1.36`).

> **Important:** Run all commands in the same Cloud Shell Bash session so that the `NS` variable remains available. This lab creates test workloads in a dedicated namespace. Do not run it in a production namespace.

---

## Step 1 – Connect to AKS and prepare the lab namespace

Open **Azure Portal → Cloud Shell**, select **Bash**, and connect to the AKS cluster if the session is not already configured.

```bash
az account show --output table
az aks get-credentials --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --overwrite-existing
kubectl config current-context
kubectl get nodes -o wide
```

Replace `<RESOURCE_GROUP>` and `<AKS_CLUSTER_NAME>` with the names of your AKS resource group and cluster. `az aks get-credentials` downloads the cluster access configuration into the Cloud Shell kubeconfig; `kubectl` uses the selected context for subsequent commands.

Create a dedicated namespace and set a Bash variable for the remaining commands.

```bash
export NS=aks-troubleshooting-capstone
kubectl create namespace "$NS" --dry-run=client -o yaml | kubectl apply -f -
kubectl get namespace "$NS"
```

A **namespace** provides a logical boundary for the lab resources. All three Deployments and their Pods will be created in this namespace.

---

# Scenario 1 – Fix an ImagePullBackOff Deployment

## Step 2 – Deploy an application with an invalid image tag

The Deployment below uses a deliberately invalid NGINX image tag. Kubernetes can create the Pod, but the kubelet cannot download the requested image from the container registry.

```bash
cat <<'EOF' > deployment-imagepull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-imagepull
  namespace: aks-troubleshooting-capstone
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-imagepull
  template:
    metadata:
      labels:
        app: web-imagepull
    spec:
      containers:
        - name: nginx
          image: nginx:lab-invalid-tag
          ports:
            - name: http
              containerPort: 80
EOF

kubectl apply -f deployment-imagepull.yaml
kubectl get deployment web-imagepull -n "$NS"
kubectl get pods -n "$NS" -l app=web-imagepull -w
```

Wait until the Pod shows `ErrImagePull` or `ImagePullBackOff`, then press **Ctrl+C** to stop watching. `ImagePullBackOff` means Kubernetes is retrying an image pull after a previous attempt failed.

## Step 3 – Inspect the Pod and identify the root cause

```bash
POD1=$(kubectl get pods -n "$NS" -l app=web-imagepull -o jsonpath='{.items[0].metadata.name}')
echo "$POD1"
kubectl get pod "$POD1" -n "$NS" -o wide
kubectl describe pod "$POD1" -n "$NS"
kubectl get events -n "$NS" --sort-by='.metadata.creationTimestamp'
kubectl get deployment web-imagepull -n "$NS" -o yaml
```

**What to inspect:**

- In `kubectl get pods`, check the `STATUS` and `READY` columns.
- In `kubectl describe pod`, inspect the **Containers** section and the **Events** at the bottom.
- Look for events such as `Failed` or `Failed to pull image`, and check the exact image reference in the Pod specification.
- `kubectl get events` provides a namespace-wide view of recent scheduling and image-pull events.

The intended root cause is the invalid image reference `nginx:lab-invalid-tag`. Image-pull failures happen before the container starts, so application logs are normally unavailable for this failure.

## Step 4 – Correct the image and verify the Deployment

Update the Deployment to use the valid public image `nginx:alpine`.

```bash
kubectl set image deployment/web-imagepull nginx=nginx:alpine -n "$NS"
kubectl rollout status deployment/web-imagepull -n "$NS" --timeout=180s
kubectl get deployment web-imagepull -n "$NS"
kubectl get pods -n "$NS" -l app=web-imagepull -o wide
```

**Expected result:** The Deployment should show `READY 1/1`, and its Pod should reach `Running` with `1/1` containers ready.

`kubectl set image` updates the container image in the Deployment's Pod template. Kubernetes creates a new ReplicaSet and replaces the failing Pod as part of the rollout.

---

# Scenario 2 – Fix a CrashLoopBackOff Deployment

## Step 5 – Deploy an application that exits repeatedly

This Deployment uses BusyBox and an intentionally incorrect startup command. The container prints an error message and exits with a non-zero exit code. Because a Deployment's Pods normally use `restartPolicy: Always`, Kubernetes restarts the container and it eventually enters `CrashLoopBackOff`.

```bash
cat <<'EOF' > deployment-crashloop.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-crashloop
  namespace: aks-troubleshooting-capstone
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-crashloop
  template:
    metadata:
      labels:
        app: web-crashloop
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["/bin/sh", "-c"]
          args:
            - 'echo "ERROR: required application configuration was not loaded"; sleep 2; exit 1'
EOF

kubectl apply -f deployment-crashloop.yaml
kubectl get pods -n "$NS" -l app=web-crashloop -w
```

Wait for the restart count to increase and the status to show `CrashLoopBackOff`, then press **Ctrl+C**. Depending on image-pull and restart timing, the Pod may briefly show `Error` or `Running` before it reaches `CrashLoopBackOff`.

## Step 6 – Inspect the container state, logs and events

```bash
POD2=$(kubectl get pods -n "$NS" -l app=web-crashloop -o jsonpath='{.items[0].metadata.name}')
echo "$POD2"
kubectl get pod "$POD2" -n "$NS"
kubectl describe pod "$POD2" -n "$NS"
kubectl logs "$POD2" -n "$NS" --tail=50
kubectl logs "$POD2" -n "$NS" --previous --tail=50
kubectl get events -n "$NS" --sort-by='.metadata.creationTimestamp'
kubectl get deployment web-crashloop -n "$NS" -o yaml
```

**What to inspect:**

- In `kubectl describe pod`, inspect **State**, **Last State**, **Reason**, **Exit Code** and **Restart Count** for the container.
- `kubectl logs` retrieves output from the current container instance.
- `kubectl logs --previous` retrieves logs from the previous terminated instance, which is useful when a container is restarting.
- Inspect the Deployment's `command` and `args` fields to understand how the container is started.

The intended root cause is the startup command that deliberately exits with code `1`. The previous container logs should contain the message:

```text
ERROR: required application configuration was not loaded
```

`CrashLoopBackOff` is a restart back-off condition, not the underlying cause. Use the termination details and logs to identify why the process exits.

## Step 7 – Correct the startup command and verify recovery

Replace the failing command with one that prints a success message and keeps the container running.

```bash
kubectl patch deployment web-crashloop -n "$NS" --type='strategic' -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","command":["/bin/sh","-c"],"args":["echo Application started successfully; exec sleep 3600"]}]}}}}'
kubectl rollout status deployment/web-crashloop -n "$NS" --timeout=180s
kubectl get deployment web-crashloop -n "$NS"
kubectl get pods -n "$NS" -l app=web-crashloop -o wide
```

**Expected result:** The new Pod should reach `Running` and `1/1` Ready, and the Deployment should show `1/1` available replica.

The patch updates the Pod template's container command and arguments. This triggers a rollout, replacing the repeatedly failing Pod with a new Pod that remains running.

---

# Scenario 3 – Fix a Pending Deployment

## Step 8 – Deploy an application with an unsatisfied node selector

The manifest below specifies a node hostname that does not exist in the cluster. The scheduler cannot find a node that matches the selector, so the Pod remains `Pending`.

```bash
cat <<'EOF' > deployment-pending.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-pending
  namespace: aks-troubleshooting-capstone
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-pending
  template:
    metadata:
      labels:
        app: web-pending
    spec:
      nodeSelector:
        kubernetes.io/hostname: aks-lab-node-does-not-exist
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - name: http
              containerPort: 80
EOF

kubectl apply -f deployment-pending.yaml
kubectl get pods -n "$NS" -l app=web-pending -w
```

Wait for the Pod to remain in `Pending`, then press **Ctrl+C**. A Pod in `Pending` has not completed scheduling and/or startup; inspect its conditions and events to determine which step is blocked.

## Step 9 – Inspect scheduling details and identify the root cause

```bash
POD3=$(kubectl get pods -n "$NS" -l app=web-pending -o jsonpath='{.items[0].metadata.name}')
echo "$POD3"
kubectl get nodes -L kubernetes.io/hostname
kubectl get pod "$POD3" -n "$NS" -o wide
kubectl describe pod "$POD3" -n "$NS"
kubectl get events -n "$NS" --sort-by='.metadata.creationTimestamp'
kubectl get deployment web-pending -n "$NS" -o yaml
```

**What to inspect:**

- Compare the actual node hostnames displayed by `kubectl get nodes -L kubernetes.io/hostname` with the value in the Deployment's `nodeSelector`.
- In `kubectl describe pod`, inspect the **Events** section for a `FailedScheduling` event.
- The event message should indicate that no node matches the Pod's node selector or node affinity.

The intended root cause is the node selector value `aks-lab-node-does-not-exist`, which does not match any node label in the cluster. This is a scheduling configuration problem; the container image does not need to be changed.

## Step 10 – Remove the invalid node selector and verify scheduling

Remove the `nodeSelector` field from the Deployment's Pod template.

```bash
kubectl patch deployment web-pending -n "$NS" --type=json -p='[{"op":"remove","path":"/spec/template/spec/nodeSelector"}]'
kubectl rollout status deployment/web-pending -n "$NS" --timeout=180s
kubectl get deployment web-pending -n "$NS"
kubectl get pods -n "$NS" -l app=web-pending -o wide
```

**Expected result:** The scheduler should place the Pod on an eligible node. The Pod should reach `Running` and `1/1` Ready, and the Deployment should show `1/1` available replica.

If the Pod remains Pending after removing the selector, inspect the new Pod's events again. There may be a separate scheduling constraint, such as insufficient CPU or memory, taints, or node availability; do not assume the original issue is still the cause.

---

## Step 11 – Review all three repaired Deployments

Run the following commands to confirm that each Deployment has one available replica and that all three Pods are ready.

```bash
kubectl get deployments -n "$NS"
kubectl get replicasets -n "$NS"
kubectl get pods -n "$NS" -o wide
kubectl get events -n "$NS" --sort-by='.metadata.creationTimestamp'
```

**Expected final state:**

| Deployment | Original failure | Root cause | Corrective action | Expected final state |
|---|---|---|---|---|
| `web-imagepull` | `ImagePullBackOff` | Invalid image tag | Updated image to `nginx:alpine` | `1/1 Running` |
| `web-crashloop` | `CrashLoopBackOff` | Startup command exits with code `1` | Replaced command with a long-running process | `1/1 Running` |
| `web-pending` | `Pending` | Node selector matches no node | Removed invalid node selector | `1/1 Running` |

To verify each Deployment's rollout explicitly:

```bash
kubectl rollout status deployment/web-imagepull -n "$NS"
kubectl rollout status deployment/web-crashloop -n "$NS"
kubectl rollout status deployment/web-pending -n "$NS"
```

## Troubleshooting command reference

| Command | Purpose |
|---|---|
| `kubectl get deployments -n "$NS"` | Check desired, updated, ready and available replicas. |
| `kubectl get replicasets -n "$NS"` | Inspect ReplicaSets managed by Deployments. |
| `kubectl get pods -n "$NS" -o wide` | View Pod status, restart count, IP address and node placement. |
| `kubectl describe pod <POD_NAME> -n "$NS"` | Inspect Pod conditions, container state, image, scheduling details and recent events. |
| `kubectl logs <POD_NAME> -n "$NS"` | Read the current container's standard output and standard error. |
| `kubectl logs <POD_NAME> -n "$NS" --previous` | Read logs from the previous terminated container instance. |
| `kubectl get events -n "$NS" --sort-by='.metadata.creationTimestamp'` | Review recent namespace events in creation-time order. |
| `kubectl get deployment <DEPLOYMENT_NAME> -n "$NS" -o yaml` | Inspect the Deployment's Pod template, image, command, arguments and scheduling settings. |
| `kubectl rollout status deployment/<DEPLOYMENT_NAME> -n "$NS"` | Wait for and verify the Deployment rollout. |

## Common failure states – quick interpretation

| Pod status / condition | What it indicates | First checks |
|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | The node cannot retrieve the specified container image. | Image repository and tag, registry authentication, registry reachability and events. |
| `CrashLoopBackOff` | The container repeatedly terminates and Kubernetes is delaying subsequent restarts. | `kubectl describe pod`, current and previous logs, exit code, command/arguments, configuration and probes. |
| `Pending` | The Pod has not completed scheduling or is waiting for a required startup dependency. | Pod conditions and `FailedScheduling` events, node selectors, affinity, taints, resource requests and PVC status. |

> **Troubleshooting principle:** A Pod status is a symptom, not a complete diagnosis. Always correlate the Pod's current state with its events, container termination details, logs and workload configuration before applying a fix.

## Cleanup

Delete the dedicated namespace and all lab resources created inside it.

```bash
kubectl delete namespace "$NS"
```

Verify that the namespace has been removed:

```bash
kubectl get namespace "$NS"
```

The expected result is `NotFound` after deletion completes. If the namespace remains in `Terminating`, inspect remaining resources before taking further action.

## Knowledge check

1. What is the difference between `ErrImagePull` and `ImagePullBackOff`?
2. Why are application logs generally unavailable when a container is failing to start because its image cannot be pulled?
3. When should `kubectl logs --previous` be used?
4. Which fields in `kubectl describe pod` help identify why a container is restarting?
5. What does a `FailedScheduling` event tell you about a Pending Pod?
6. How did the invalid `nodeSelector` prevent the scheduler from placing the Pod?
7. Why should you inspect the Deployment specification as well as the Pod when troubleshooting a Deployment-managed workload?
8. What evidence confirms that a repaired Deployment has recovered successfully?

## Completion checklist

- [ ] Connected to the intended AKS cluster from Azure Cloud Shell using Bash.
- [ ] Created and used the dedicated lab namespace.
- [ ] Diagnosed and repaired the `ImagePullBackOff` Deployment.
- [ ] Diagnosed and repaired the `CrashLoopBackOff` Deployment.
- [ ] Diagnosed and repaired the `Pending` Deployment.
- [ ] Verified that all three Deployments have one available replica.
- [ ] Used `kubectl get`, `kubectl describe`, `kubectl logs` and `kubectl get events` during troubleshooting.
- [ ] Deleted the lab namespace and its resources.

## References

- [Microsoft Learn – AKS workload deployment triage](https://learn.microsoft.com/en-us/azure/architecture/operator-guides/aks/aks-triage-deployment)
- [Microsoft Learn – Troubleshoot Pod workload restarts in AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/troubleshoot-pod-workload-restart)
- [Microsoft Learn – Troubleshoot application connection issues in AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connection-issues-application-hosted-aks-cluster)
- [Kubernetes – Debugging Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-pod-replication-controller/)
- [Kubernetes – Debugging running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
