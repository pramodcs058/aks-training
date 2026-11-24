# Lab 19 – Basic AKS Workload Troubleshooting Using kubectl

## Lab details

| Item | Description |
|---|---|
| **Module** | Module 7 – Azure Monitoring, Logging & Cost Management |
| **Lab type** | Basic |
| **Estimated duration** | 15–20 minutes |
| **Execution environment** | Azure Portal – Cloud Shell – Bash |
| **Cluster** | Existing AKS cluster |

## Problem statement

Deploy a sample application, inspect its Pod status, retrieve application logs using `kubectl logs`, and examine events using `kubectl describe` and `kubectl get events`.

## Objectives

- Understand how to inspect Kubernetes workload and Pod status.
- Retrieve application logs to understand what a container is doing.
- Use Pod details and Kubernetes events to identify common startup and runtime issues.
- Recognize a `CrashLoopBackOff` condition and perform a basic recovery.

## Expected outcome

Participants will be able to use `kubectl get`, `kubectl describe`, `kubectl logs` and `kubectl get events` to investigate common workload issues in AKS.

---

## Prerequisites

- Access to an existing AKS cluster.
- Permission to create and delete Deployments, Pods, Services and Namespaces in the cluster.
- Azure Cloud Shell opened in **Bash** mode from the Azure portal.
- The cluster must be in a healthy state and have access to pull the container images used in this lab.

> **Note:** Run all commands in this lab from Azure Portal → Cloud Shell → Bash. Replace the resource group and cluster name with the values provided by your trainer.

## Step 1 – Connect to the AKS cluster

Set the AKS details and retrieve cluster credentials.

```bash
RESOURCE_GROUP="<your-resource-group>"
AKS_NAME="<your-aks-cluster-name>"
NAMESPACE="aks-troubleshooting-lab"

az account show --output table
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME"
kubectl config current-context
kubectl get nodes
```

**Explanation:** `az aks get-credentials` retrieves the Kubernetes credentials and merges them into the Cloud Shell kubeconfig. `kubectl get nodes` confirms that the current context can communicate with the cluster.

**Expected result:** The current AKS context is displayed and the cluster nodes are listed.

## Step 2 – Create a lab namespace

```bash
kubectl create namespace "$NAMESPACE"
kubectl get namespace "$NAMESPACE"
```

**Explanation:** A namespace provides an isolated scope for the lab resources. All resources created in this exercise will be placed in this namespace.

## Step 3 – Deploy a sample NGINX application

Create a Deployment with one NGINX replica.

```bash
kubectl create deployment nginx-demo --image=nginx:alpine --replicas=1 -n "$NAMESPACE"
kubectl rollout status deployment/nginx-demo -n "$NAMESPACE" --timeout=120s
```

**Explanation:** A Deployment manages the desired number of application Pods. The rollout command waits for the Deployment to complete its rollout or reports a timeout if the Pod does not become ready.

Check the Pod status:

```bash
kubectl get deployments -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
```

**Explanation:** `kubectl get pods` displays readiness, status, restart count and age. A healthy NGINX Pod should eventually show `1/1` under READY and `Running` under STATUS.

Capture the Pod name for the following commands:

```bash
POD_NAME=$(kubectl get pods -n "$NAMESPACE" -l app=nginx-demo -o jsonpath='{.items[0].metadata.name}')
echo "$POD_NAME"
```

**Explanation:** The label selector finds the Pod created by the Deployment. The command stores its generated name in `POD_NAME`, avoiding the need to type the full Pod name manually.

## Step 4 – Inspect Pod details using kubectl describe

```bash
kubectl describe pod "$POD_NAME" -n "$NAMESPACE"
```

**Explanation:** `kubectl describe pod` displays the Pod's configuration, node placement, container state, restart count, conditions and recent events. Review the **State**, **Ready**, **Restart Count** and **Events** sections.

**Expected observation:** The NGINX container should be running and ready. Events commonly include successful scheduling, image pulling, container creation and container start messages.

## Step 5 – Retrieve application logs

```bash
kubectl logs "$POD_NAME" -n "$NAMESPACE" --tail=50
```

**Explanation:** `kubectl logs` retrieves the container's standard output and standard error. The NGINX image may show startup messages; access-log entries appear after the application receives HTTP requests.

To follow new log entries continuously, use:

```bash
kubectl logs "$POD_NAME" -n "$NAMESPACE" --follow
```

Press **Ctrl+C** to stop following the logs. This stops the log display only; it does not stop the application Pod.

## Step 6 – Create a controlled application failure

To practise troubleshooting, create a small Pod whose process intentionally exits with an error. This produces a repeatable `CrashLoopBackOff` condition without changing the NGINX Deployment.

```bash
cat <<'EOF' | kubectl apply -n "$NAMESPACE" -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-demo
  labels:
    app: crash-demo
spec:
  containers:
    - name: demo-container
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - echo "Application starting"; echo "ERROR: simulated startup failure"; exit 1
EOF
```

**Explanation:** The container prints a startup message and an error, then exits with a non-zero exit code. The default Pod restart policy is `Always`, so Kubernetes restarts the container; repeated failures eventually result in `CrashLoopBackOff`.

Wait briefly, then inspect the Pod:

```bash
sleep 15
kubectl get pod crash-demo -n "$NAMESPACE" -o wide
```

**Expected observation:** The Pod may initially show `Error` or `Running` while the first attempt completes. After repeated restarts it should show `CrashLoopBackOff`. Timing can vary because Kubernetes applies an increasing restart back-off delay.

## Step 7 – Inspect the failed Pod and its events

Describe the failing Pod:

```bash
kubectl describe pod crash-demo -n "$NAMESPACE"
```

**Explanation:** Review the container's **Last State**, **Reason**, **Exit Code**, **Restart Count** and the **Events** section. A non-zero exit code and repeated `BackOff` events indicate that the container keeps terminating and Kubernetes is delaying subsequent restarts.

Retrieve the logs from the current container instance:

```bash
kubectl logs crash-demo -n "$NAMESPACE"
```

Retrieve logs from the previous terminated container instance:

```bash
kubectl logs crash-demo -n "$NAMESPACE" --previous
```

**Explanation:** The `--previous` option retrieves logs from the most recently terminated container instance, which is especially useful when a container restarts before you can inspect its current logs.

List recent namespace events:

```bash
kubectl get events -n "$NAMESPACE" --sort-by='.metadata.creationTimestamp'
```

Show only Warning events:

```bash
kubectl get events -n "$NAMESPACE" --field-selector type=Warning --sort-by='.metadata.creationTimestamp'
```

**Explanation:** Events provide cluster-generated information about scheduling, image pulls, container starts, failures and restart back-off. Use them together with logs and `describe`; events alone do not always identify the application-level root cause.

## Step 8 – Recover the failed workload

The failure is caused by the demo command exiting with code `1`. Delete the failed Pod and recreate it with a command that stays running.

```bash
kubectl delete pod crash-demo -n "$NAMESPACE"
```

```bash
cat <<'EOF' | kubectl apply -n "$NAMESPACE" -f -
apiVersion: v1
kind: Pod
metadata:
  name: crash-demo
  labels:
    app: crash-demo
spec:
  containers:
    - name: demo-container
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - echo "Application started successfully"; sleep 3600
EOF
```

Verify the Pod:

```bash
kubectl get pod crash-demo -n "$NAMESPACE"
kubectl logs crash-demo -n "$NAMESPACE"
```

**Explanation:** A Pod's container command is not changed in place for this exercise. The failed Pod is deleted and recreated with a corrected command. The new container prints a success message and remains running.

## Step 9 – Review the troubleshooting commands

| Command | Purpose |
|---|---|
| `kubectl get pods -n <namespace>` | Check Pod status, readiness and restart count. |
| `kubectl get pods -o wide -n <namespace>` | Also display Pod IP and node placement. |
| `kubectl describe pod <pod> -n <namespace>` | Inspect Pod configuration, conditions, container state and related events. |
| `kubectl logs <pod> -n <namespace>` | Retrieve the current container's standard output and standard error. |
| `kubectl logs <pod> -n <namespace> --previous` | Retrieve logs from the previous terminated container instance. |
| `kubectl get events -n <namespace>` | Review recent Kubernetes events in the namespace. |

## Step 10 – Clean up lab resources

```bash
kubectl delete namespace "$NAMESPACE"
```

**Explanation:** Deleting the lab namespace removes the NGINX Deployment, Services (if any), Pods and other namespaced resources created during this exercise.

---

## Common Pod states to recognize

| Pod status | Meaning | First checks |
|---|---|---|
| `Pending` | The Pod has not yet been scheduled or its required resources are not ready. | `kubectl describe pod` – inspect scheduling and volume events. |
| `ImagePullBackOff` | Kubernetes cannot pull the container image. | Check image name, tag, registry access and Pod events. |
| `ErrImagePull` | An image pull attempt failed. | Inspect the `Failed` or `BackOff` events with `kubectl describe pod`. |
| `CrashLoopBackOff` | The container repeatedly exits and Kubernetes is delaying restarts. | Check `kubectl logs --previous` and `kubectl describe pod`. |
| `Running` but not `Ready` | The container is running but has not passed its readiness condition. | Inspect readiness probes, container logs and Pod events. |

## Completion checklist

- [ ] Connected to the AKS cluster from Azure Cloud Shell (Bash).
- [ ] Created a dedicated namespace.
- [ ] Deployed the NGINX application and inspected its Pod status.
- [ ] Retrieved application logs using `kubectl logs`.
- [ ] Created and identified a Pod in `CrashLoopBackOff`.
- [ ] Used `kubectl describe` and `kubectl get events` to inspect the failure.
- [ ] Retrieved the previous container instance logs using `kubectl logs --previous`.
- [ ] Recreated the failed Pod with a corrected command.
- [ ] Deleted the lab namespace.

## Knowledge check

1. What information does `kubectl get pods` provide?
2. What is the difference between `kubectl logs <pod>` and `kubectl logs <pod> --previous`?
3. Which sections of `kubectl describe pod` are useful when a container repeatedly restarts?
4. What does the `CrashLoopBackOff` status indicate?
5. Why should application logs and Kubernetes events be examined together?

## Official references

- [Microsoft Learn – Troubleshoot application connection issues in AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connection-issues-application-hosted-aks-cluster)
- [Kubernetes – Debugging Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes – kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
- [Kubernetes – kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [Kubernetes – kubectl events](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_events/)
