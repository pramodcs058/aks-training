# Lab 13 – Manually Scale a Kubernetes Deployment on AKS

## Lab details

| Item | Description |
|---|---|
| **Module** | Module 5 – Overview on Scaling & High Availability in AKS |
| **Lab type** | Basic |
| **Estimated duration** | 15 minutes |
| **Environment** | Azure Portal – Cloud Shell (Bash) |
| **Workload** | NGINX Deployment |

## Problem statement

Deploy an application with one replica and manually scale the Deployment to three replicas using `kubectl`. Verify the number of running Pods.

## Objectives

- Understand how a Kubernetes Deployment maintains the desired number of Pod replicas.
- Manually scale an application by changing the replica count.
- Verify the Deployment and its Pods after scaling.

## Expected outcome

Participants will be able to create a Deployment with one replica, scale it to three replicas using `kubectl scale`, and verify that all three Pods are running and ready.

---

## Concepts

- **Deployment:** A Kubernetes controller that manages ReplicaSets and ensures that the required number of application Pods are maintained.
- **Replica:** One running instance of an application's Pod. Increasing replicas allows more instances of the workload to run.
- **Manual scaling:** Changing the desired replica count directly, for example from one to three, using `kubectl scale`.
- **Reconciliation:** Kubernetes continuously compares the actual state with the desired state and creates or removes Pods to match it.

## Prerequisites

- Access to an existing AKS cluster.
- Permission to read and create workloads in the cluster and namespace.
- Azure Portal access and an active **Bash** Cloud Shell session.
- Sufficient cluster capacity to schedule three small NGINX Pods.

> **Important:** This lab creates a dedicated namespace and Deployment. Do not use a production Deployment for this exercise. Manual changes to a Deployment managed by a HorizontalPodAutoscaler (HPA) can be overridden by the HPA.

---

## Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Open **Cloud Shell** from the top navigation bar.
3. Select **Bash** as the shell environment.
4. Set the resource group and AKS cluster names to match your environment:

```bash
RESOURCE_GROUP="<your-resource-group-name>"
AKS_CLUSTER="<your-aks-cluster-name>"
```

Replace the sample values before running the commands.

5. Verify the active Azure subscription:

```bash
az account show --output table
```

This displays the subscription and account currently selected in Cloud Shell.

6. Retrieve AKS credentials and configure the current kubectl context:

```bash
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER" --overwrite-existing
```

This downloads the cluster credentials and adds or updates the AKS context in the kubeconfig file used by `kubectl`.

7. Verify the connection:

```bash
kubectl get nodes
kubectl config current-context
```

Confirm that the nodes are in the `Ready` state and that the current context refers to the intended AKS cluster.

## Step 2 – Create a dedicated namespace

```bash
NAMESPACE="aks-scaling-lab"
kubectl create namespace "$NAMESPACE"
```

A namespace isolates the resources created for this exercise from workloads in other namespaces.

## Step 3 – Deploy NGINX with one replica

```bash
kubectl create deployment nginx-scale-demo --image=nginx:stable --replicas=1 --port=80 --namespace="$NAMESPACE"
```

This creates a Deployment named `nginx-scale-demo` with a desired replica count of one. The container listens on port 80; no Service or external access is created in this lab.

Check the Deployment:

```bash
kubectl get deployment nginx-scale-demo --namespace="$NAMESPACE"
```

Check the Pod:

```bash
kubectl get pods --namespace="$NAMESPACE" -l app=nginx-scale-demo -o wide
```

**Expected observation:** The Deployment should show `1/1` ready, and one Pod should reach the `Running` state with `1/1` containers ready.

Wait for the initial Pod to become ready before continuing:

```bash
kubectl rollout status deployment/nginx-scale-demo --namespace="$NAMESPACE" --timeout=120s
```

This waits for the Deployment rollout to complete and reports an error if it does not become ready within the specified timeout.

## Step 4 – Scale the Deployment from one to three replicas

```bash
kubectl scale deployment/nginx-scale-demo --replicas=3 --current-replicas=1 --namespace="$NAMESPACE"
```

This changes the desired replica count from one to three. The `--current-replicas=1` precondition helps prevent the command from scaling if another change has already altered the current replica count.

Wait for all three replicas to become ready:

```bash
kubectl rollout status deployment/nginx-scale-demo --namespace="$NAMESPACE" --timeout=180s
```

## Step 5 – Verify the scaled Deployment and Pods

1. Check the Deployment summary:

```bash
kubectl get deployment nginx-scale-demo --namespace="$NAMESPACE"
```

**Expected observation:** `READY` should show `3/3`, with `UP-TO-DATE` and `AVAILABLE` showing `3`.

2. List the Pods managed by the Deployment:

```bash
kubectl get pods --namespace="$NAMESPACE" -l app=nginx-scale-demo -o wide
```

**Expected observation:** Three Pods should be listed. Each should show `1/1` in the `READY` column and `Running` in the `STATUS` column.

3. Confirm the desired and actual replica counts:

```bash
kubectl get deployment nginx-scale-demo --namespace="$NAMESPACE" -o jsonpath='Desired replicas: {.spec.replicas}{"\n"}Ready replicas: {.status.readyReplicas}{"\n"}Available replicas: {.status.availableReplicas}{"\n"}'
```

The desired, ready and available replica counts should each be `3` once the rollout is complete.

4. Inspect the Deployment and its ReplicaSet:

```bash
kubectl describe deployment nginx-scale-demo --namespace="$NAMESPACE"
kubectl get replicasets --namespace="$NAMESPACE" -l app=nginx-scale-demo
```

The Deployment description shows its desired and available replicas. The ReplicaSet is the controller that creates and maintains the three NGINX Pods for this Deployment.

---

## Expected result summary

| Resource | Expected result |
|---|---|
| Namespace | `aks-scaling-lab` created |
| Deployment | `nginx-scale-demo` |
| Initial replicas | 1 |
| Final desired replicas | 3 |
| Final ready replicas | 3 |
| Running Pods | 3 |

## Troubleshooting

| Issue | Checks / action |
|---|---|
| Pod remains `Pending` | Run `kubectl describe pod <pod-name> --namespace="$NAMESPACE"` and check Events for insufficient CPU or memory, taints, or scheduling constraints. |
| Pod shows `ImagePullBackOff` | Run `kubectl describe pod <pod-name> --namespace="$NAMESPACE"` and review Events for image-pull or registry connectivity errors. |
| Pod is not ready | Run `kubectl describe pod <pod-name> --namespace="$NAMESPACE"` and `kubectl logs <pod-name> --namespace="$NAMESPACE"`. |
| Deployment does not reach `3/3` | Run `kubectl get pods --namespace="$NAMESPACE" -o wide` and `kubectl describe deployment nginx-scale-demo --namespace="$NAMESPACE"` to inspect Pod status and Events. |
| Scale command reports a replica precondition conflict | Check the current count with `kubectl get deployment nginx-scale-demo --namespace="$NAMESPACE"`; rerun the scale command with the verified current count only if appropriate. |

## Cleanup

Delete the lab namespace and all resources created inside it:

```bash
kubectl delete namespace "$NAMESPACE"
```

Confirm that the namespace is terminating or has been removed:

```bash
kubectl get namespace "$NAMESPACE"
```

A `NotFound` response confirms that the namespace has been deleted.

## Knowledge check

1. What is the role of a Deployment in Kubernetes?
2. What does the `--replicas=3` option change?
3. Which Kubernetes controller creates and maintains the Pods for a Deployment?
4. Which commands help verify that all three replicas are ready?
5. Why should you avoid manually scaling a Deployment that is managed by an HPA?

## Key takeaway

A Kubernetes Deployment maintains the desired number of Pod replicas. Using `kubectl scale`, an operator can manually change that desired count; Kubernetes then creates or removes Pods until the actual state matches the requested state.

## References

- [Kubernetes – Manually scale a Deployment](https://kubernetes.io/docs/tasks/run-application/scale-deployment/)
- [Kubernetes – `kubectl scale` reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/)
- [Microsoft Learn – Connect to an AKS cluster using kubectl](https://learn.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal#connect-to-the-cluster)
