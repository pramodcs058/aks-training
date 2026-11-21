# Lab 2 – Inspect and Scale AKS Node Pools

## Problem statement

An organization needs to review the compute capacity of an existing Azure Kubernetes Service (AKS) cluster. Inspect its system and user node pools, review their VM sizes and node counts, and perform a controlled scale-out of a manually managed user node pool.

## Objectives

- Identify system and user node pools.
- Inspect VM size, node count and autoscaling configuration.
- View the Kubernetes nodes associated with the AKS cluster.
- Scale a user node pool out by one node and restore its original size.

## Key concepts

| Concept | Explanation |
|---|---|
| **Node pool** | A group of AKS worker nodes that share configuration such as VM size and operating system. |
| **System node pool** | Primarily runs critical Kubernetes system components. Do not use it for this lab's scaling exercise. |
| **User node pool** | Intended primarily for application workloads and is the target for this lab. |
| **VM size** | Defines the compute capacity of each node, including its CPU and memory resources. |
| **Manual scaling** | An operator explicitly sets the number of nodes in a node pool. |
| **Cluster Autoscaler** | Automatically adjusts node count within configured limits. Do not manually scale a pool while its autoscaler is enabled for this exercise. |

## Prerequisites

- An existing, running AKS cluster with at least one user node pool.
- Permission to view and scale the AKS node pool.
- Kubernetes permission to list nodes.
- A training or non-production cluster. Do not perform this exercise on production without approval.
- The selected user node pool must have **Cluster Autoscaler disabled**.

## Lab steps

### 1. Open Azure Cloud Shell

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Open **Cloud Shell** from the portal toolbar.
3. Select **Bash** as the shell environment.

Cloud Shell is already authenticated to Azure through the portal session. No separate `az login` is required.

### 2. Verify the active subscription

```bash
az account show --output table
```

Displays the currently active Azure account and subscription.

If required, select the subscription that contains the AKS cluster:

```bash
az account set --subscription "<SUBSCRIPTION-ID-OR-NAME>"
```

Sets the active subscription for subsequent Azure CLI commands.

### 3. Set the AKS details

Replace the sample values with the details provided by the instructor:

```bash
RESOURCE_GROUP="<RESOURCE-GROUP-NAME>"
AKS_CLUSTER="<AKS-CLUSTER-NAME>"
```

Stores the resource group and cluster name in Bash variables for reuse.

Verify the cluster:

```bash
az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER" --query "{Name:name,Location:location,State:provisioningState,KubernetesVersion:kubernetesVersion}" --output table
```

Displays basic AKS information. Confirm that the cluster is the intended one and its provisioning state is `Succeeded`.

### 4. List and compare the node pools

```bash
az aks nodepool list --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --query "[].{Name:name,Mode:mode,VMSize:vmSize,NodeCount:count,Autoscaling:enableAutoScaling,MinCount:minCount,MaxCount:maxCount,State:provisioningState}" --output table
```

Lists the node pools and displays their mode, VM size, node count, autoscaling settings and provisioning state.

**Record your observations:**

| Item | Observation |
|---|---|
| System node pool name | |
| User node pool name | |
| User pool VM size | |
| User pool node count | |
| User pool autoscaling enabled? | |

### 5. Select and inspect a user node pool

Set the name of an existing **user** node pool from the previous output:

```bash
USER_POOL="<USER-NODE-POOL-NAME>"
```

Displays the selected node pool's configuration:

```bash
az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query "{Name:name,Mode:mode,VMSize:vmSize,NodeCount:count,Autoscaling:enableAutoScaling,MinCount:minCount,MaxCount:maxCount,State:provisioningState}" --output table
```

Confirm all of the following before continuing:

- `Mode` is `User`.
- `State` is `Succeeded`.
- `Autoscaling` is `false`.

**Stop here** if the selected pool is a system pool, autoscaling is enabled, or the provisioning state is not `Succeeded`. Ask the instructor for a suitable user pool. Do not disable autoscaling on a shared cluster just to complete this lab.

### 6. Connect kubectl and inspect Kubernetes nodes

Retrieve the AKS credentials:

```bash
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER"
```

Adds the AKS cluster connection details to the Cloud Shell kubeconfig so that `kubectl` can communicate with the cluster.

List the Kubernetes nodes:

```bash
kubectl get nodes -o wide
```

Displays the nodes registered with Kubernetes, including their status, Kubernetes version and network information.

Display each node's AKS node pool label:

```bash
kubectl get nodes -L kubernetes.azure.com/agentpool
```

Shows the node pool name associated with each Kubernetes node.

### 7. Capture the original count and scale out by one node

Capture the current node count and autoscaling setting:

```bash
ORIGINAL_COUNT=$(az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query count --output tsv)
AUTOSCALING=$(az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query enableAutoScaling --output tsv)
POOL_MODE=$(az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query mode --output tsv)
POOL_STATE=$(az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query provisioningState --output tsv)
```

Stores the current count and retrieves the pool mode and autoscaling setting for the safety check and restore operation.

Display the values and validate them before scaling:

```bash
printf "Pool mode: %s\nAutoscaling enabled: %s\nProvisioning state: %s\n" "$POOL_MODE" "$AUTOSCALING" "$POOL_STATE"
```

Continue only when the mode is `User`, autoscaling is `false`, and the provisioning state is `Succeeded`. Otherwise, stop and ask the instructor for a suitable pool.

Set the target count to one more than the original count:

```bash
TARGET_COUNT=$((ORIGINAL_COUNT + 1))
echo "Original count: $ORIGINAL_COUNT | Target count: $TARGET_COUNT"
```

Calculates and displays the temporary node count for this exercise.

Scale the user node pool:

```bash
az aks nodepool scale --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --node-count "$TARGET_COUNT"
```

Changes the selected user node pool to the target node count. The command waits for the operation unless an asynchronous option is specified.

### 8. Verify the scale operation

```bash
az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query "{Name:name,NodeCount:count,State:provisioningState}" --output table
```

Checks the configured node count and Azure provisioning state after scaling.

```bash
kubectl get nodes -L kubernetes.azure.com/agentpool
```

Verifies that Kubernetes has registered the nodes and shows the pool to which each node belongs. Newly provisioned nodes should reach `Ready` status.

### 9. Restore the original node count

```bash
az aks nodepool scale --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --node-count "$ORIGINAL_COUNT"
```

Restores the user node pool to the count recorded before the exercise.

Verify the final state:

```bash
az aks nodepool show --resource-group "$RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" --name "$USER_POOL" --query "{Name:name,NodeCount:count,State:provisioningState}" --output table
```

Confirm that the original node count has been restored and the provisioning state is `Succeeded`.

## Expected outcome

Participants can identify system and user node pools, inspect their VM sizes and scaling configuration, view the Kubernetes nodes, and safely scale a manually managed user node pool up by one node and back to its original size.

## Quick knowledge check

1. What is the primary difference between a system node pool and a user node pool?
2. Which property identifies the VM size used by a node pool?
3. What does the `enableAutoScaling` property indicate?
4. Why should a pool with Cluster Autoscaler enabled not be manually scaled in this exercise?
5. Which command displays the Kubernetes nodes registered in the cluster?

## References

- [Manage node pools in AKS – Azure CLI reference](https://learn.microsoft.com/en-us/cli/azure/aks/nodepool)
- [Scale node pools in AKS](https://learn.microsoft.com/en-us/azure/aks/scale-node-pools)
- [Resize AKS clusters](https://learn.microsoft.com/en-us/azure/aks/resize-cluster?tabs=azure-cli)
