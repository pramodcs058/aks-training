# Lab 20 – Analyze AKS Logs Using KQL in Log Analytics

## Lab overview

| Item | Details |
|---|---|
| **Platform** | Azure Kubernetes Service (AKS) |
| **Tools** | Azure Portal, Azure Cloud Shell (Bash), Log Analytics, Kusto Query Language (KQL) |
| **Estimated duration** | 20–30 minutes, excluding monitoring setup and data-ingestion delay |
| **Namespace used** | `kql-lab` |

### Problem statement

Use an existing Log Analytics workspace that receives AKS Container insights data. Run KQL queries to retrieve Pod inventory, application logs, Kubernetes events and container lifecycle information.

### Objectives

- Understand how AKS monitoring data is stored in Log Analytics.
- Use KQL to filter, project, summarize and sort Kubernetes data.
- Investigate Pod status, container logs, restarts and Kubernetes events.

### Expected outcome

Participants will be able to run basic KQL queries and use the results to investigate workload behaviour in an AKS cluster.

---

## Prerequisites

- An existing AKS cluster and access to its Kubernetes API.
- An existing Log Analytics workspace connected to that AKS cluster.
- **Container insights** enabled, with data flowing to the workspace.
- Permission to view the Log Analytics workspace and run queries. A common minimum is the **Log Analytics Reader** role on the workspace.
- Azure Portal access and permission to use Cloud Shell.

> **Important:** This lab uses the `ContainerLogV2`, `KubePodInventory`, `KubeEvents` and `ContainerInventory` tables. The tables available depend on the cluster's Data Collection Rule (DCR) and Container insights configuration. The `ContainerLogV2` table is used rather than the legacy `ContainerLog` table.

## Lab architecture

```text
AKS Cluster
   |
   | Container insights agent / Azure Monitor data collection
   v
Log Analytics Workspace
   |
   +-- KubePodInventory   -> Pod and container status, restarts, node
   +-- ContainerLogV2    -> Container stdout/stderr application logs
   +-- KubeEvents         -> Kubernetes events and reasons
   +-- ContainerInventory -> Container image and lifecycle information
   |
   v
Log Analytics – Logs
   |
   v
KQL queries and results
```

---

## Step 1 – Open Azure Cloud Shell and connect to AKS

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Select **Cloud Shell** and choose **Bash**.
3. Connect to the existing AKS cluster. Replace the values with your cluster details:

```bash
RESOURCE_GROUP="<aks-resource-group>"
AKS_NAME="<aks-cluster-name>"
az account show --output table
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME"
kubectl config current-context
kubectl get nodes
```

**Explanation:** `az aks get-credentials` retrieves the cluster credentials and updates the kubeconfig used by `kubectl`. `kubectl get nodes` confirms that the current context can communicate with the cluster.

---

## Step 2 – Verify that Container insights data is available

1. In the Azure Portal, open the AKS cluster.
2. Open **Monitoring** or **Insights** and confirm that Container insights is configured.
3. Open the Log Analytics workspace connected to the cluster and select **Logs**. You can also open **Logs** from the AKS cluster to start with a cluster-scoped query context.
4. Run this query to check whether Pod inventory is arriving:

```kusto
KubePodInventory
| where TimeGenerated > ago(1h)
| summarize Records = count(), LatestRecord = max(TimeGenerated) by ClusterName
| order by LatestRecord desc
```

5. Check whether container logs are arriving:

```kusto
ContainerLogV2
| where TimeGenerated > ago(1h)
| summarize Records = count(), LatestRecord = max(TimeGenerated) by PodNamespace
| order by LatestRecord desc
```

**Explanation:** These queries provide a quick ingestion check. If they return no rows, confirm that Container insights is enabled, that the cluster's DCR collects the relevant tables, and that the workspace selected is the one associated with the cluster.

---

## Step 3 – Create a sample workload to generate application logs

This step creates a small NGINX Deployment and a client Pod that sends periodic HTTP requests. The requests generate NGINX access-log entries that can be queried in Log Analytics.

1. Create a dedicated namespace:

```bash
kubectl create namespace kql-lab
```

2. Deploy two NGINX replicas:

```bash
kubectl create deployment kql-nginx --image=nginx:stable-alpine --replicas=2 --namespace=kql-lab
```

3. Expose the Deployment internally through a ClusterIP Service:

```bash
kubectl expose deployment kql-nginx --name=kql-nginx --port=80 --target-port=80 --type=ClusterIP --namespace=kql-lab
```

4. Create a BusyBox client Pod that periodically sends HTTP requests to the NGINX Service:

```bash
kubectl run kql-client --image=busybox:1.36 --restart=Never --namespace=kql-lab --command -- sh -c 'while true; do wget -q -O /dev/null http://kql-nginx.kql-lab.svc.cluster.local; sleep 15; done'
```

5. Wait for the NGINX Deployment and client Pod to become ready:

```bash
kubectl rollout status deployment/kql-nginx --namespace=kql-lab
kubectl wait --for=condition=Ready pod/kql-client --namespace=kql-lab --timeout=120s
kubectl get pods,services --namespace=kql-lab -o wide
```

6. Confirm that NGINX is producing access logs:

```bash
kubectl logs deployment/kql-nginx --namespace=kql-lab --tail=10
```

**Explanation:** The client reaches NGINX through Kubernetes Service DNS. Each request is written to the NGINX container's standard output, which Container insights collects when container-log collection is enabled.

> **Note:** Log ingestion is not instantaneous. Generate traffic, then allow several minutes for the records to appear in Log Analytics. Keep the client Pod running while completing the queries.

---

## Step 4 – Query Pod inventory using KQL

Open **Log Analytics workspace → Logs**. Set the time range to **Last 1 hour** and run the following queries.

### Query 4.1 – List Pods in the lab namespace

```kusto
KubePodInventory
| where TimeGenerated > ago(1h)
| where Namespace == "kql-lab"
| summarize arg_max(TimeGenerated, PodStatus, PodIp, Computer, ContainerStatus, ContainerStatusReason, ContainerRestartCount) by ClusterName, Namespace, Name
| project TimeGenerated, ClusterName, Namespace, PodName = Name, PodStatus, ContainerStatus, ContainerStatusReason, ContainerRestartCount, PodIp, Node = Computer
| order by PodName asc
```

**What to observe:** The two NGINX Pods and the client Pod should appear. Review Pod status, container status, node placement and restart count.

### Query 4.2 – Find Pods that are not running

```kusto
KubePodInventory
| where TimeGenerated > ago(1h)
| where Namespace == "kql-lab"
| where PodStatus != "Running"
| project TimeGenerated, Name, PodStatus, ContainerStatus, ContainerStatusReason, Computer
| order by TimeGenerated desc
```

**What to observe:** A healthy lab should normally show no current non-running Pods. Historical samples can remain in the selected time window, so check the timestamp before treating an older record as a current failure.

### Query 4.3 – Review container restarts

```kusto
KubePodInventory
| where TimeGenerated > ago(1h)
| where Namespace == "kql-lab"
| summarize arg_max(TimeGenerated, ContainerStatus, ContainerStatusReason, ContainerRestartCount) by ClusterName, Namespace, Name, ContainerName
| project Name, ContainerName, ContainerStatus, ContainerStatusReason, ContainerRestartCount, TimeGenerated
| order by ContainerRestartCount desc
```

**What to observe:** A rising restart count can indicate repeated container failures. Use the time and status fields to decide whether further investigation is required.

---

## Step 5 – Retrieve application logs from ContainerLogV2

### Query 5.1 – View NGINX application logs

```kusto
ContainerLogV2
| where TimeGenerated > ago(1h)
| where PodNamespace == "kql-lab"
| where PodName startswith "kql-nginx-"
| project TimeGenerated, PodName, ContainerName, LogSource, LogMessage
| order by TimeGenerated desc
```

**What to observe:** NGINX access-log messages should include HTTP requests generated by the `kql-client` Pod. `LogSource` identifies whether a record came from stdout or stderr.

### Query 5.2 – Count HTTP GET requests by Pod and time interval

```kusto
ContainerLogV2
| where TimeGenerated > ago(1h)
| where PodNamespace == "kql-lab"
| where PodName startswith "kql-nginx-"
| where tostring(LogMessage) contains "GET /"
| summarize RequestCount = count() by PodName, bin(TimeGenerated, 5m)
| order by TimeGenerated asc
| render timechart
```

**What to observe:** The time chart shows the volume of matching requests processed by each NGINX Pod in five-minute intervals.

### Query 5.3 – Search for error messages

```kusto
ContainerLogV2
| where TimeGenerated > ago(1h)
| where PodNamespace == "kql-lab"
| where PodName startswith "kql-nginx-"
| where (LogSource == "stderr") or (tostring(LogMessage) has_any ("error", "failed", "exception"))
| project TimeGenerated, PodName, ContainerName, LogSource, LogMessage
| order by TimeGenerated desc
```

**What to observe:** This query searches stderr records and messages containing common error terms. An empty result is normal when the sample application has not generated errors.

---

## Step 6 – Examine Kubernetes events

Run the following query to retrieve events for the lab namespace:

```kusto
KubeEvents
| where TimeGenerated > ago(1h)
| where Namespace == "kql-lab"
| project TimeGenerated, Namespace, ObjectKind, Name, KubeEventType, Reason, SourceComponent, Message, Count
| order by TimeGenerated desc
```

**What to observe:** Review the event reason and message for information about scheduling, image pulling, container creation and other Kubernetes activity.

> **Important:** Container insights does not collect `Normal` Kubernetes event types by default. The `KubeEvents` table might therefore show only warning or other collected events. An empty result does not necessarily mean that no Kubernetes events occurred.

---

## Step 7 – Review container lifecycle information

Run this query to inspect container image and lifecycle information:

```kusto
ContainerInventory
| where TimeGenerated > ago(1h)
| where ContainerHostname startswith "kql-nginx-"
| project TimeGenerated, Computer, ContainerHostname, Name, Image, ImageTag, ContainerState, CreatedTime, StartedTime, FinishedTime
| order by TimeGenerated desc
```

**What to observe:** Identify the node, container image, image tag, current observed state and lifecycle timestamps for the NGINX containers.

> **Note:** `ContainerInventory` must be included in the workspace's data collection configuration. If this table is not available or has no records, use the `KubePodInventory` queries in Step 4 to inspect Pod and container status instead.

---

## Step 8 – Correlate Pod inventory with container logs

This query joins Pod inventory to container logs using the container ID. It is useful when you want to associate a log message with the Pod name recorded in inventory.

```kusto
let startTimestamp = ago(1h);
let LabContainers =
    KubePodInventory
    | where TimeGenerated > startTimestamp
    | where Namespace == "kql-lab"
    | where Name startswith "kql-nginx-"
    | where isnotempty(ContainerID)
    | project ContainerID, PodName = Name
    | distinct ContainerID, PodName;
LabContainers
| join kind=inner
    (
        ContainerLogV2
        | where TimeGenerated > startTimestamp
        | where PodNamespace == "kql-lab"
    )
    on $left.ContainerID == $right.ContainerId
| project TimeGenerated, PodName, ContainerName, LogSource, LogMessage
| order by TimeGenerated desc
```

**What to observe:** Each returned log entry is associated with the matching NGINX Pod through its container ID.

---

## Step 9 – Optional: Filter results to one AKS cluster

If the Log Analytics workspace receives data from multiple clusters, add a cluster or resource filter so that results are not mixed.

For `KubePodInventory` and `KubeEvents`, add a filter using the cluster name shown in the query results:

```kusto
| where ClusterName == "<aks-cluster-name>"
```

For `ContainerLogV2` and `ContainerInventory`, use the AKS resource ID when required:

```kusto
| where _ResourceId =~ "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.ContainerService/managedClusters/<aks-cluster-name>"
```

Replace the placeholder with the exact AKS resource ID. Do not use the placeholder text as-is.

---

## Step 10 – Clean up lab resources

After completing the KQL queries, return to Cloud Shell and delete the namespace:

```bash
kubectl delete namespace kql-lab
```

Verify that the namespace is removed:

```bash
kubectl get namespace kql-lab
```

The command should report that the namespace is not found. This cleanup removes the sample Deployment, Service and client Pod. It does not delete the AKS cluster or Log Analytics workspace.

---

## Troubleshooting

| Issue | Possible cause | What to check |
|---|---|---|
| `KubePodInventory` returns no rows | Wrong workspace, monitoring not enabled, table not collected or selected time range too narrow | Confirm the AKS cluster is connected to this workspace, Container insights is enabled, the DCR collects Pod inventory, and the time range is appropriate. |
| `ContainerLogV2` returns no rows | Container-log collection is disabled, the table is not in the DCR, or ingestion has not completed | Check the DCR's collected data settings and generate requests again. Allow time for ingestion. |
| NGINX logs do not show HTTP requests | Client Pod is not running, Service name is wrong or the client cannot reach the Service | Run `kubectl get pods,svc -n kql-lab`, `kubectl logs kql-client -n kql-lab` and `kubectl logs deployment/kql-nginx -n kql-lab --tail=20`. |
| `KubeEvents` has no rows | No events were collected for the selected namespace or Normal events are excluded | Widen the time range and remember that Normal events are not collected by default. |
| `ContainerInventory` has no rows | Container inventory is not enabled in the DCR or no matching records were ingested | Check DCR table selection and use `KubePodInventory` as the alternative. |
| KQL reports a table or column not found | Table schema differs from the configured collection profile or a legacy query was used | Check the table list and schema in the workspace. Prefer `ContainerLogV2` for container stdout/stderr logs. |
| Results include Pods from other clusters | The workspace is shared by multiple clusters | Filter by `ClusterName` or the AKS `_ResourceId`. |

---

## Completion checklist

- [ ] Connected to the existing AKS cluster from Cloud Shell.
- [ ] Confirmed that Container insights data is arriving in the intended workspace.
- [ ] Created the `kql-lab` namespace, NGINX Deployment, Service and client Pod.
- [ ] Queried Pod information using `KubePodInventory`.
- [ ] Retrieved NGINX application logs using `ContainerLogV2`.
- [ ] Queried Kubernetes events using `KubeEvents`.
- [ ] Reviewed container lifecycle information using `ContainerInventory` or Pod inventory.
- [ ] Correlated Pod inventory with container logs.
- [ ] Deleted the `kql-lab` namespace.

## Knowledge-check questions

1. What type of information is stored in `KubePodInventory`?
2. Which table stores container stdout and stderr logs when the ContainerLogV2 schema is enabled?
3. What is the purpose of `TimeGenerated > ago(1h)` in a KQL query?
4. Why is `PodNamespace` useful when querying `ContainerLogV2`?
5. Which field can be used to associate container logs with Pod inventory?
6. Why might `KubeEvents` contain no `Normal` events?
7. How can you filter results when one Log Analytics workspace receives data from multiple AKS clusters?

## Key takeaways

- Container insights sends AKS inventory, event and container-log data to Log Analytics according to the configured DCR and collection settings.
- `KubePodInventory` is useful for Pod status, container status, restart counts and node placement.
- `ContainerLogV2` is the preferred table for querying collected container stdout/stderr logs.
- `KubeEvents` provides Kubernetes event reasons and messages, subject to event collection settings.
- `ContainerInventory` provides container image and lifecycle information when that table is collected.
- KQL makes it possible to filter, summarize, visualize and correlate Kubernetes monitoring data across time.

## Official references

- [Query container logs in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-log-query)
- [Enable monitoring for AKS clusters](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable)
- [ContainerLogV2 schema](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-logs-schema)
- [KubePodInventory table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/kubepodinventory)
- [KubeEvents table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/kubeevents)
- [ContainerLogV2 table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/containerlogv2)
- [ContainerInventory table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/containerinventory)

---

**End of Lab 20**
