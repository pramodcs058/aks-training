# Lab 3 – Provision an AKS Cluster and Configure kubectl Access

## Prerequisites

- An Azure subscription with permission to create resource groups and AKS clusters.
- Permission to retrieve AKS user credentials (`listClusterUserCredential`) and Kubernetes permissions to query cluster resources.
- Access to the [Azure portal](https://portal.azure.com/).
- Cloud Shell storage configured when prompted.

> **Cost and region note:** AKS provisioning creates Azure resources that may incur charges. Use a training subscription and a region where the required VM capacity and quota are available. Delete the lab resource group after the exercise.

## Step 1 – Open Azure Cloud Shell

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Select the **Cloud Shell** icon (`>_`) in the portal toolbar.
3. Select **Bash** as the shell environment.
4. Complete the Cloud Shell storage setup if prompted.

Cloud Shell includes Azure CLI and `kubectl`, so no local installation is required.

## Step 2 – Verify Azure CLI Sign-in and Subscription

```bash
az account show --output table
```

Displays the active Azure account and subscription associated with the Cloud Shell session.

If the required subscription is not selected, list the available subscriptions:

```bash
az account list --output table
```

Select the subscription to use:

```bash
az account set --subscription "<SUBSCRIPTION_ID>"
```

Replace `<SUBSCRIPTION_ID>` with the required subscription ID. Confirm the selection:

```bash
az account show --query "{name:name,id:id,tenantId:tenantId}" --output table
```

## Step 3 – Define Lab Variables

```bash
SUFFIX=$RANDOM
RESOURCE_GROUP="rg-aks-cli-lab-${SUFFIX}"
CLUSTER_NAME="aks-cli-lab-${SUFFIX}"
LOCATION="centralindia"
```

These variables define unique names for the lab resource group and AKS cluster. Change `LOCATION` if another Azure region is required or has available quota.

Display the values before creating resources:

```bash
printf "Resource Group: %s\nCluster Name: %s\nLocation: %s\n" "$RESOURCE_GROUP" "$CLUSTER_NAME" "$LOCATION"
```

## Step 4 – Create the Resource Group

```bash
az group create \
  --name "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --output table
```

Creates the resource group that will contain the AKS cluster resource. AKS will also create a separate, Azure-managed node resource group for the supporting infrastructure.

## Step 5 – Provision the AKS Cluster

```bash
az aks create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --location "$LOCATION" \
  --node-count 1 \
  --generate-ssh-keys \
  --output table
```

Creates a basic AKS cluster with one system node. The single-node configuration is intended to reduce lab resource consumption; it is not a production high-availability configuration. Cluster provisioning can take several minutes.

> **Note:** `--generate-ssh-keys` generates SSH keys when required for the cluster's Linux nodes. Do not share private key material.

## Step 6 – Verify AKS Provisioning

```bash
az aks show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --query "{name:name,provisioningState:provisioningState,kubernetesVersion:kubernetesVersion,location:location,nodeResourceGroup:nodeResourceGroup}" \
  --output table
```

Displays the cluster's provisioning state, Kubernetes version, region and managed node resource group. Confirm that `provisioningState` is `Succeeded` before proceeding.

## Step 7 – Retrieve AKS Credentials

```bash
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --context "$CLUSTER_NAME" \
  --overwrite-existing
```

Retrieves the cluster's user credentials and adds or updates a context in the Cloud Shell kubeconfig file. The `--context` option gives the context a predictable name for the remaining steps.

> **Authentication and authorization:** Retrieving credentials configures the client connection; it does not bypass Kubernetes authorization. The signed-in identity must still have permission to perform the requested Kubernetes operations. This lab uses the default user-credential flow and does not request administrator credentials with `--admin`.

## Step 8 – View and Select the kubectl Context

List the available contexts:

```bash
kubectl config get-contexts
```

Displays the clusters and contexts currently configured in kubeconfig. The asterisk (`*`) identifies the active context.

Select the lab cluster context:

```bash
kubectl config use-context "$CLUSTER_NAME"
```

Sets the lab cluster as the active context so subsequent `kubectl` commands target this cluster.

Confirm the active context:

```bash
kubectl config current-context
```

The output should match the value of `$CLUSTER_NAME`.

## Step 9 – Verify Connectivity to the Kubernetes Control Plane

```bash
kubectl cluster-info
```

Displays the Kubernetes control plane endpoint and core service information. A successful response confirms that `kubectl` can communicate with the cluster API endpoint.

List the cluster nodes:

```bash
kubectl get nodes -o wide
```

Displays the worker nodes registered with the cluster, including their status, role, Kubernetes version and IP information. The node created for this lab should eventually show `Ready`.

List the namespaces:

```bash
kubectl get namespaces
```

Displays the logical namespaces available in the cluster. Namespaces help organize and isolate Kubernetes resources.

List the system Pods:

```bash
kubectl get pods --namespace kube-system
```

Displays the Pods running in the `kube-system` namespace. These Pods provide cluster-level services such as networking and DNS; the exact list varies by cluster configuration.

## Expected Results

- The resource group is created successfully.
- The AKS cluster reports `provisioningState: Succeeded`.
- The lab cluster context appears in `kubectl config get-contexts`.
- `kubectl config current-context` displays the lab cluster context.
- `kubectl cluster-info` returns the control plane information.
- `kubectl get nodes` lists the node and its status becomes `Ready`.
- Namespaces and system Pods are listed successfully.

## Troubleshooting

| Issue | What to check |
|---|---|
| `az aks create` fails due to quota or capacity | Confirm the selected region, VM quota and available VM capacity. Try an approved region with available capacity. |
| `az aks show` does not report `Succeeded` | Check the deployment state and error details in the Azure portal or with `az aks show`. Wait for provisioning to finish before retrieving credentials. |
| `az aks get-credentials` returns an authorization error | Confirm the active subscription and that the identity has permission to retrieve AKS user credentials. |
| `kubectl` reports `Unauthorized` or `Forbidden` | Credential retrieval and Kubernetes authorization are separate. Confirm the identity's configured Kubernetes/Azure RBAC permissions. |
| `kubectl` cannot reach the API server | Confirm the active context, cluster state and API server network accessibility. Private clusters and authorized IP ranges require an allowed network path. |
| Node status is `NotReady` immediately after creation | Allow node initialization to complete, then recheck with `kubectl get nodes` and `kubectl describe node <NODE_NAME>`. |

## Cleanup – Delete Lab Resources

> Run cleanup only when the resource group was created for this lab and contains no resources you need to keep. The command deletes the AKS cluster and all other resources in that resource group.

```bash
az group delete \
  --name "$RESOURCE_GROUP" \
  --yes
```

Deletes the lab resource group. The operation also removes the AKS cluster and its managed resources; allow the deletion operation to complete.

## Lab Completion Checklist

- [ ] Opened Azure Cloud Shell using Bash.
- [ ] Verified the active Azure subscription.
- [ ] Created a resource group.
- [ ] Provisioned an AKS cluster using Azure CLI.
- [ ] Verified the AKS provisioning state.
- [ ] Retrieved user credentials using `az aks get-credentials`.
- [ ] Viewed and selected the kubectl context.
- [ ] Verified control plane connectivity using `kubectl cluster-info`.
- [ ] Listed nodes, namespaces and system Pods.
- [ ] Deleted the lab resource group when finished.

## Key Takeaways

- `az aks create` provisions a managed Kubernetes cluster on Azure.
- `az aks get-credentials` configures kubeconfig so `kubectl` can connect to the AKS cluster.
- A kubeconfig context determines which cluster `kubectl` targets.
- `kubectl cluster-info` and `kubectl get nodes` are useful first checks after connecting to a cluster.
- Azure authentication and Kubernetes authorization are related but separate checks.

## References

- [Quickstart: Deploy an AKS cluster using Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli)
- [Azure CLI – `az aks`](https://learn.microsoft.com/en-us/cli/azure/aks)
- [Connect to an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal#connect-to-the-cluster)
- [kubectl command reference](https://kubernetes.io/docs/reference/kubectl/)
