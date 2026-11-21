# Lab 1 – Connect to an Existing AKS Cluster Using Azure CLI and kubectl

**Lab type:** Basic  
**Suggested duration:** 15 minutes  
**Module:** Module 1 – AKS Architecture & Cluster Provisioning  
**Difficulty:** Beginner to Intermediate  

---

## 1. Lab Scenario

Your team already has an Azure Kubernetes Service (AKS) cluster provisioned for an application. As a developer or cloud engineer, you need to connect to the cluster and perform an initial health and inventory check before deploying or troubleshooting workloads.

In this lab, you will authenticate to Azure, download the AKS user credentials into your local Kubernetes configuration, select the correct kubectl context, and inspect the cluster's nodes, namespaces, and system Pods.

> **Important:** This lab uses an **existing AKS cluster**. You will not create, scale, modify, or delete Azure resources.

## 2. Objectives

By the end of this lab, participants will be able to:

1. Sign in to Azure using Azure CLI and select the correct subscription.
2. Retrieve user credentials for an existing AKS cluster using `az aks get-credentials`.
3. Understand and inspect Kubernetes contexts using `kubectl config`.
4. Verify cluster connectivity and list AKS worker nodes.
5. List Kubernetes namespaces and inspect system Pods in the `kube-system` namespace.

## 3. Expected Outcome

Participants will be able to connect to an existing AKS cluster, switch the active kubectl context when multiple contexts are configured, and identify the cluster's worker nodes, namespaces, and key system Pods.

## 4. Time Plan

| Activity | Suggested time |
|---|---:|
| Check prerequisites and sign in to Azure | 2 minutes |
| Select subscription and identify the AKS cluster | 2 minutes |
| Download AKS credentials | 2 minutes |
| Inspect and select the kubectl context | 2 minutes |
| List nodes, namespaces, and system Pods | 6 minutes |
| Review the results | 1 minute |
| **Total** | **15 minutes** |

## 5. Prerequisites

Before starting, make sure the following are available:

- An Azure subscription containing an already provisioned AKS cluster.
- The AKS cluster name and its Azure resource group name.
- An Azure user account with permission to sign in and retrieve the AKS **user** credentials. The user commonly needs the **Azure Kubernetes Service Cluster User Role** or equivalent permission at the appropriate scope.
- Kubernetes authorization that permits the required read operations, such as listing nodes, namespaces, and Pods in `kube-system`. Depending on the cluster's authorization configuration, this is provided through Azure RBAC for Kubernetes or Kubernetes RBAC.
- Azure CLI and `kubectl` installed on the machine, or access to **Azure Cloud Shell**, where these tools are available.
- Network connectivity to the AKS API server. For a private AKS cluster, use a machine or Cloud Shell environment that has the required network path to the private API endpoint. Authorized IP ranges must also permit the connection, where configured.

### Check whether the tools are available

Run the following commands in the terminal.

```bash
az version
kubectl version --client
```

- `az version` displays the installed Azure CLI version and confirms that the `az` command is available.
- `kubectl version --client` displays the installed Kubernetes command-line client version without requiring a cluster connection.

**If `kubectl` is not installed locally:** after installing Azure CLI, you can use the following command to install the Kubernetes CLI:

```bash
az aks install-cli
```

This downloads and installs `kubectl` for local use. Azure Cloud Shell normally has Azure CLI and `kubectl` available already.

## 6. Important Concepts

| Concept | Explanation |
|---|---|
| **AKS cluster** | A managed Kubernetes service in Azure. Azure manages the Kubernetes control plane, while you manage the worker node pools and the workloads running on them. |
| **Control plane** | The Kubernetes management layer that accepts API requests and makes scheduling and cluster-management decisions. In AKS, the control plane is managed by Azure and its managed components are not shown as ordinary worker nodes by `kubectl get nodes`. |
| **Worker node** | A virtual machine in an AKS node pool that provides compute capacity for application and system Pods. Nodes normally appear in the output of `kubectl get nodes`. |
| **Azure CLI** | The command-line tool used to authenticate to Azure and manage Azure resources, including retrieving AKS access credentials. |
| **kubectl** | The Kubernetes command-line client used to communicate with the Kubernetes API server and inspect or manage cluster resources. |
| **kubeconfig** | A configuration file, usually stored at `~/.kube/config`, that contains cluster connection information, user authentication configuration, and contexts used by kubectl. Treat this file as sensitive. |
| **Context** | A named combination of a Kubernetes cluster, user, and optionally a default namespace. kubectl sends commands to the cluster selected by the current context. |
| **Namespace** | A logical boundary used to organize Kubernetes resources within a cluster. For example, application teams can keep their resources separate from system resources. |
| **Pod** | The smallest deployable Kubernetes unit. A Pod contains one or more containers that share networking and can share storage. |
| **System namespace (`kube-system`)** | A namespace that contains Kubernetes and cluster add-on components required for cluster operation, such as DNS and networking-related Pods. The exact components depend on the cluster configuration. |

## 7. Lab Instructions

### Step 1 – Sign in to Azure

Open **Azure Cloud Shell** in the Azure portal, or open a terminal on your local machine.

Run:

```bash
az login
```

**Command explanation:** `az login` authenticates your user with Azure and establishes an Azure CLI session. Follow the browser sign-in process if prompted; in Cloud Shell, you may already be signed in.

If the command reports that you are already logged in, continue to the next step.

To view the signed-in account and current subscription, run:

```bash
az account show --output table
```

**Command explanation:** `az account show` displays the active Azure account and subscription. Confirm that the displayed subscription is the one that contains the training AKS cluster.

### Step 2 – Select the correct Azure subscription

List the subscriptions available to your account:

```bash
az account list --output table
```

**Command explanation:** `az account list` displays the subscriptions your signed-in account can access. Identify the subscription that contains the AKS cluster.

Set the correct subscription using its name or ID:

```bash
az account set --subscription "<SUBSCRIPTION_NAME_OR_ID>"
```

**Command explanation:** `az account set` changes the active Azure subscription used by subsequent Azure CLI commands. Replace the placeholder with the actual subscription name or subscription ID; do not type the angle brackets.

Verify the selection:

```bash
az account show --query "{subscription:name, subscriptionId:id}" --output table
```

**Command explanation:** This query displays the active subscription name and ID so you can confirm that Azure CLI is targeting the expected subscription.

### Step 3 – Specify and verify the AKS cluster details

For the remaining commands, define the resource group and cluster name. Replace the sample values with the details supplied by your trainer or administrator.

**Bash / Azure Cloud Shell (Bash):**

```bash
RESOURCE_GROUP="rg-aks-training"
AKS_CLUSTER="aks-training-01"
```

**Windows PowerShell:**

```powershell
$RESOURCE_GROUP = "rg-aks-training"
$AKS_CLUSTER = "aks-training-01"
```

The values above are examples only. Use the actual resource group and AKS cluster name provided for the lab.

Check that the cluster exists and inspect a few properties:

```bash
az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER" --query "{name:name, location:location, kubernetesVersion:kubernetesVersion, provisioningState:provisioningState}" --output table
```

**Command explanation:** `az aks show` retrieves the configuration of the specified AKS resource. Confirm the cluster name and that its provisioning state is `Succeeded` before proceeding.

### Step 4 – Retrieve AKS credentials

Run the following command:

```bash
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER"
```

**Command explanation:** `az aks get-credentials` retrieves the AKS **user** access credentials and merges the cluster connection information into the kubeconfig file. kubectl uses this configuration to connect to the Kubernetes API server.

What happens when this command runs?

1. Azure CLI contacts Azure Resource Manager to retrieve the cluster's user credential information.
2. The cluster connection and authentication configuration are added to the kubeconfig file, normally `~/.kube/config`.
3. A context for the AKS cluster is added or updated, and it is normally selected as the current context.
4. kubectl can now use that context to communicate with the AKS API server, provided the user has the necessary Kubernetes authorization and network access.

> **Security note:** Use the default user credentials for this lab. Do not add `--admin` as a shortcut. Admin credentials are highly privileged and should only be retrieved by authorized administrators when there is a specific operational requirement.

### Step 5 – Inspect and select the kubectl context

List the contexts available in your kubeconfig:

```bash
kubectl config get-contexts
```

**Command explanation:** `kubectl config get-contexts` lists the configured Kubernetes contexts. The asterisk (`*`) identifies the context currently used by kubectl.

Display the active context:

```bash
kubectl config current-context
```

**Command explanation:** `kubectl config current-context` prints the name of the context that kubectl will use when no other context is specified on a command.

#### Practise switching contexts

If more than one context is listed, copy the exact name of the AKS context from the `kubectl config get-contexts` output and select it:

```bash
kubectl config use-context <AKS_CONTEXT_NAME>
```

**Command explanation:** `kubectl config use-context` changes the current context in kubeconfig. Replace the placeholder with the exact AKS context name shown in your output.

Verify the selection:

```bash
kubectl config current-context
```

If this is the only context in your kubeconfig, you may not have another cluster to switch to. In that case, verify the current context and continue. Do not create or switch to an unrelated context just for this exercise.

> **Trainer note:** A context is a local kubectl selection; it is not a separate AKS cluster. When multiple clusters are configured, switching contexts changes which cluster receives subsequent kubectl commands. Always check the current context before performing operations, especially commands that modify resources.

### Step 6 – Verify connectivity to the AKS cluster

Run:

```bash
kubectl cluster-info
```

**Command explanation:** `kubectl cluster-info` displays the addresses of the Kubernetes control plane and core services that kubectl can discover. A successful response is an initial indication that kubectl can communicate with the cluster API.

Now list the cluster nodes:

```bash
kubectl get nodes
```

**Command explanation:** `kubectl get nodes` lists the Kubernetes worker nodes registered with the cluster. Check the `STATUS` column; nodes that are ready to host workloads normally show `Ready`.

For additional node details, run:

```bash
kubectl get nodes --output wide
```

**Command explanation:** The `-o wide` option displays additional node information, such as the node's internal IP address, operating system image, and runtime version.

**What to observe:**

- The node names commonly include the AKS node pool name.
- The `STATUS` column indicates whether a node is ready.
- The `ROLES` column may show `agent` or may be blank, depending on the Kubernetes and AKS version/configuration.
- AKS's Azure-managed control plane is not listed as a normal worker node in this output.

### Step 7 – List Kubernetes namespaces

Run:

```bash
kubectl get namespaces
```

**Command explanation:** `kubectl get namespaces` lists the namespaces in the current cluster. Namespaces help organize and separate Kubernetes resources.

You can use the shorter form as well:

```bash
kubectl get ns
```

**Command explanation:** `ns` is the short name for `namespaces`; it returns the same type of information using fewer keystrokes.

Look for common namespaces such as:

| Namespace | Purpose |
|---|---|
| `default` | The default namespace used when a resource is created without an explicit namespace. |
| `kube-system` | Contains Kubernetes system and cluster add-on resources. |
| `kube-public` | A namespace intended for publicly readable cluster information; it is not commonly used for application workloads. |
| `kube-node-lease` | Contains Lease objects used by Kubernetes to help track node heartbeats. |

The exact list can vary because administrators and add-ons may create additional namespaces.

### Step 8 – Inspect system Pods in kube-system

Run:

```bash
kubectl get pods --namespace kube-system
```

**Command explanation:** This command lists Pods in the `kube-system` namespace. Review the `READY`, `STATUS`, `RESTARTS`, and `AGE` columns to understand whether the listed system Pods appear healthy.

For more details, including the node on which each Pod is running, use:

```bash
kubectl get pods --namespace kube-system --output wide
```

**Command explanation:** The wide output adds information such as the Pod IP and the node hosting the Pod. This helps relate system workloads to the AKS worker nodes.

Depending on the AKS configuration, you may see components with names related to:

| Example component | Purpose |
|---|---|
| CoreDNS | Provides DNS name resolution for services and Pods inside the Kubernetes cluster. |
| Metrics Server | Collects resource usage metrics such as CPU and memory for Kubernetes features that consume these metrics. |
| Azure networking or node agents | Support networking, node-level integration, or other AKS platform functions. The exact agents depend on the cluster's networking and configuration. |
| Konnectivity-related components | Support communication paths between the control plane and workloads or nodes in supported AKS configurations. |

The exact Pod names and set of components vary by AKS version, networking mode, add-ons, and cluster configuration. Do not expect every example in this table to appear in every cluster.

> **Important distinction:** Not every Kubernetes control-plane component appears as a Pod in `kube-system`. AKS control-plane components such as the API server, scheduler, and controller manager are managed by Azure and are not exposed as ordinary customer-managed worker-node Pods.

### Step 9 – Inspect one system Pod (optional, if time permits)

Choose one Pod name from the `kubectl get pods -n kube-system` output and run:

```bash
kubectl describe pod <POD_NAME> --namespace kube-system
```

**Command explanation:** `kubectl describe pod` displays detailed information about a Pod, including its labels, containers, node assignment, conditions, and recent events. Replace `<POD_NAME>` with an actual Pod name from your cluster.

Use the output to locate the `Node`, `Status`, `Containers`, and `Events` sections. This is a useful first step when investigating a Pod that is not healthy.

## 8. Validation Checklist

Before completing the lab, confirm each item:

- [ ] Azure CLI sign-in completed successfully.
- [ ] The correct Azure subscription is selected.
- [ ] `az aks show` returned the expected AKS cluster.
- [ ] `az aks get-credentials` completed without an authorization or connectivity error.
- [ ] The AKS context is visible in `kubectl config get-contexts`.
- [ ] The expected AKS context is selected as the current context.
- [ ] `kubectl cluster-info` returned cluster information.
- [ ] `kubectl get nodes` listed the worker nodes and their statuses.
- [ ] `kubectl get namespaces` listed the cluster namespaces.
- [ ] `kubectl get pods -n kube-system` listed the system Pods.

## 9. Common Issues and Troubleshooting

| Issue / message | Possible reason | What to do |
|---|---|---|
| `az: command not found` or `az is not recognized` | Azure CLI is not installed or is not available in the terminal's PATH. | Use Azure Cloud Shell or install Azure CLI, then open a new terminal. |
| `kubectl: command not found` or `kubectl is not recognized` | kubectl is not installed or is not available in PATH. | Run `az aks install-cli` from a terminal with Azure CLI installed, or use Azure Cloud Shell. |
| `Please run 'az login'` | Azure CLI has no active sign-in session. | Run `az login` and complete authentication. |
| AKS resource not found | The subscription, resource group, or cluster name is incorrect. | Run `az account show` and `az aks list --output table`; verify the resource group and cluster name with the trainer or administrator. |
| `AuthorizationFailed` while running `az aks get-credentials` | The signed-in user does not have permission to retrieve AKS user credentials at the required scope. | Ask the Azure administrator to grant the appropriate AKS Cluster User permission. Do not work around this by requesting admin credentials unless explicitly authorized. |
| `Forbidden` from a kubectl command | Authentication may have succeeded, but the user does not have Kubernetes authorization for the requested resource or namespace. | Ask the cluster administrator to grant the required read-only access through Azure RBAC for Kubernetes or Kubernetes RBAC. Listing nodes and namespaces requires suitable cluster-level read permissions; listing system Pods requires read permission in `kube-system`. |
| Connection timeout or unable to reach the API server | Network access to the API endpoint is unavailable, or the cluster is private / protected by authorized IP ranges. | Use an approved network path, VPN, jump host, or permitted Cloud Shell configuration. Ask the administrator to confirm private endpoint DNS/routing and authorized IP settings. |
| `context` not found | The context name supplied to `use-context` does not exactly match a configured context. | Run `kubectl config get-contexts` and copy the exact context name. |
| Credentials appear to be expired or authentication fails | The Azure sign-in session or token needs refreshing, or the cluster's identity configuration requires additional sign-in. | Run `az login` again and retry `az aks get-credentials`. Follow your organization's Microsoft Entra ID sign-in requirements. |

### Useful diagnostic commands

```bash
az account show --output table
az aks list --output table
kubectl config get-contexts
kubectl config current-context
kubectl cluster-info
```

These commands help verify the active Azure subscription, discover AKS clusters available in that subscription, inspect kubectl contexts, and test API connectivity.

## 10. Knowledge Check

1. What is the purpose of `az aks get-credentials`?
2. Where does kubectl normally store its cluster and context configuration?
3. What does the asterisk (`*`) indicate in `kubectl config get-contexts`?
4. What is the difference between a Kubernetes node and a Pod?
5. Why might a user be able to run `az aks get-credentials` but receive `Forbidden` when running `kubectl get nodes`?
6. Why are the AKS-managed control-plane components not listed as ordinary worker nodes?
7. What type of components would you generally expect to find in the `kube-system` namespace?

## 11. Key Takeaways

- Azure CLI is used to authenticate to Azure and retrieve AKS connection credentials; kubectl is used to interact with Kubernetes resources.
- `az aks get-credentials` normally merges the AKS user credentials and cluster information into the kubeconfig file.
- A kubectl context determines which configured cluster and user kubectl uses for commands.
- `kubectl get nodes` displays worker nodes, not the Azure-managed AKS control plane.
- Namespaces organize Kubernetes resources, while `kube-system` contains system and add-on workloads.
- Successful authentication does not guarantee authorization to read every Kubernetes resource. Access is also governed by the cluster's Kubernetes authorization configuration.

## 12. References

- [Microsoft Learn – Azure CLI `az aks` reference (`get-credentials`)](https://learn.microsoft.com/en-us/cli/azure/aks)
- [Microsoft Learn – Connect to an AKS cluster using kubectl](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-cluster)
- [Microsoft Learn – Create AKS infrastructure and retrieve cluster credentials](https://learn.microsoft.com/en-us/azure/aks/create-aks-infrastructure)
- [Kubernetes documentation – kubectl config](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/)
- [Kubernetes documentation – kubectl config current-context](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_current-context/)

---

**End of Lab 1 – Connect to an Existing AKS Cluster Using Azure CLI and kubectl**
