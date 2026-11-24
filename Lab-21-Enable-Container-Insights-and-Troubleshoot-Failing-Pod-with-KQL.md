# Lab 21 – Enable Azure Monitor Container Insights and Troubleshoot a Failing Pod Using KQL

## Lab overview

**Scenario:** Enable Azure Monitor Container Insights for an existing AKS cluster, deploy a deliberately failing sample Pod, and investigate its status, container logs, restart behaviour and Kubernetes warning events using Log Analytics and Kusto Query Language (KQL).

| Item | Details |
|---|---|
| Environment | Azure Portal – Cloud Shell – Bash |
| Duration | 30–40 minutes (monitoring data ingestion may add wait time) |
| AKS resource | Existing AKS cluster |
| Monitoring destination | Existing Log Analytics workspace |
| Sample namespace | `ci-debug-lab` |
| Sample Pod | `failing-demo-pod` |
| Main tables | `KubePodInventory`, `ContainerLogV2`, `KubeEvents` |

## Objectives

- Enable Azure Monitor Container Insights for an existing AKS cluster.
- Connect the monitoring add-on to a Log Analytics workspace.
- Deploy a sample Pod that repeatedly exits with an application error.
- Inspect Pod state and restart details in Kubernetes.
- Use KQL to examine Pod inventory, application logs and Kubernetes events.
- Identify the root cause and verify a corrected workload.

## Concepts

- **Azure Monitor Container Insights:** Collects container logs and Kubernetes inventory/events from AKS and stores the data in a Log Analytics workspace for analysis.
- **Log Analytics workspace:** Azure Monitor Logs data store where the collected Container Insights tables can be queried.
- **KubePodInventory:** Contains Pod and container inventory, including Pod status, container status and restart counts.
- **ContainerLogV2:** Stores container standard output and standard error log records in the current V2 schema.
- **KubeEvents:** Stores Kubernetes events such as container back-off and scheduling warnings.
- **KQL:** Query language used in Azure Monitor Logs to filter, project, summarize and correlate monitoring records.

> **Important:** Container Insights sends data to Log Analytics asynchronously. After enabling monitoring or creating the failing Pod, allow several minutes for records to appear. The exact delay depends on the cluster and workspace.

---

## Step 1 – Open Azure Cloud Shell using Bash

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Open **Cloud Shell** from the top navigation bar.
3. Select **Bash** as the shell environment.
4. Confirm that the correct Azure subscription is selected.

```bash
az account show --output table
```

This displays the active subscription and signed-in account context.

If required, select the subscription that contains the AKS cluster and Log Analytics workspace:

```bash
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
```

This changes the active subscription for subsequent Azure CLI commands.

## Step 2 – Set the AKS and Log Analytics workspace details

Replace the sample values below with the resource names in your environment.

```bash
AKS_RESOURCE_GROUP="<AKS_RESOURCE_GROUP>"
AKS_NAME="<AKS_CLUSTER_NAME>"
WORKSPACE_RESOURCE_GROUP="<WORKSPACE_RESOURCE_GROUP>"
WORKSPACE_NAME="<LOG_ANALYTICS_WORKSPACE_NAME>"
NAMESPACE="ci-debug-lab"
POD_NAME="failing-demo-pod"
```

Retrieve the Log Analytics workspace resource ID:

```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show --resource-group "$WORKSPACE_RESOURCE_GROUP" --workspace-name "$WORKSPACE_NAME" --query id --output tsv)
```

The workspace resource ID identifies the Log Analytics workspace that will receive Container Insights data.

Display the workspace details and confirm the ID is not empty:

```bash
echo "$WORKSPACE_ID"
```

If this command fails, verify the workspace name, resource group, subscription and your Azure permissions before continuing.

## Step 3 – Check the existing AKS cluster and monitoring add-on

Display the AKS cluster status and current monitoring add-on state:

```bash
az aks show --resource-group "$AKS_RESOURCE_GROUP" --name "$AKS_NAME" --query "{Name:name,Location:location,ProvisioningState:provisioningState,MonitoringEnabled:addonProfiles.omsagent.enabled}" --output table
```

The `MonitoringEnabled` value indicates whether the Container Insights monitoring add-on is already enabled.

Enable Container Insights with the existing Log Analytics workspace if it is not already enabled:

```bash
az aks enable-addons --addon monitoring --resource-group "$AKS_RESOURCE_GROUP" --name "$AKS_NAME" --workspace-resource-id "$WORKSPACE_ID"
```

This enables the AKS monitoring add-on and associates it with the specified Log Analytics workspace. If monitoring is already enabled and connected to the correct workspace, do not run the enable command again.

Verify the monitoring add-on configuration:

```bash
az aks show --resource-group "$AKS_RESOURCE_GROUP" --name "$AKS_NAME" --query addonProfiles.omsagent --output json
```

The output should show that the monitoring add-on is enabled and include its configuration. Enabling the add-on does not make historical logs appear; new telemetry is collected after monitoring is configured.

> **Permissions:** You need permission to read the AKS cluster and workspace and to update the AKS monitoring configuration. If the enable operation fails, ask the subscription or resource administrator to verify your RBAC permissions.

## Step 4 – Connect kubectl to the AKS cluster

```bash
az aks get-credentials --resource-group "$AKS_RESOURCE_GROUP" --name "$AKS_NAME" --overwrite-existing
```

This retrieves the cluster credentials and updates the kubeconfig context used by `kubectl`.

Verify cluster connectivity:

```bash
kubectl get nodes
```

The command should list the AKS nodes and their status. Continue only when the cluster is reachable.

## Step 5 – Create a namespace for the troubleshooting exercise

```bash
kubectl create namespace "$NAMESPACE"
```

A namespace isolates the sample workload and makes it easier to filter Kubernetes objects and monitoring records for this lab.

Verify the namespace:

```bash
kubectl get namespace "$NAMESPACE"
```

## Step 6 – Deploy a deliberately failing Pod

The Pod below prints a startup message and a simulated configuration error to standard error, waits briefly, and exits with a non-zero status. Kubernetes restarts the container, allowing you to observe restart behaviour and collect repeated log records.

```bash
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: failing-demo-pod
  namespace: ${NAMESPACE}
  labels:
    app: failing-demo
spec:
  containers:
    - name: demo-container
      image: busybox:1.36.1
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Starting sample application"
          echo "ERROR: required configuration file /app/config.yaml was not found" >&2
          sleep 10
          exit 1
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
        limits:
          cpu: 100m
          memory: 64Mi
YAML
```

The Pod is intentionally configured to fail after writing an error message. The `sleep` interval gives the monitoring agent time to collect the container output before each restart.

## Step 7 – Inspect the failing Pod using kubectl

Check the Pod status:

```bash
kubectl get pod "$POD_NAME" --namespace "$NAMESPACE" --watch
```

Wait until the status changes to `CrashLoopBackOff`, then press **Ctrl+C** to stop watching. The exact restart count and timing can vary.

Display the current Pod status and restart count:

```bash
kubectl get pod "$POD_NAME" --namespace "$NAMESPACE" --output wide
```

Inspect the Pod's conditions, container state and recent events:

```bash
kubectl describe pod "$POD_NAME" --namespace "$NAMESPACE"
```

Retrieve the logs from the current container instance:

```bash
kubectl logs "$POD_NAME" --namespace "$NAMESPACE" --container demo-container
```

Retrieve the logs from the previous container instance after a restart:

```bash
kubectl logs "$POD_NAME" --namespace "$NAMESPACE" --container demo-container --previous
```

These commands provide an immediate cluster-side view. The next steps use the centralized monitoring data in Log Analytics.

## Step 8 – Open the Log Analytics workspace

1. In the Azure portal, search for **Log Analytics workspaces**.
2. Open the workspace specified in Step 2.
3. Select **Logs** from the workspace menu.
4. If prompted, choose the workspace-scoped query experience.
5. Set the query time range to **Last 1 hour** or a range that includes the time when the Pod was deployed.

Run each KQL query below separately. Replace `ci-debug-lab` or `failing-demo-pod` if you changed the values in this lab.

> **Data availability check:** Before troubleshooting, run `ContainerLogV2 | where TimeGenerated > ago(1h) | take 10`. If the table is not found or returns no records, see the troubleshooting section at the end of this lab. Ensure the workspace is the one linked to this AKS cluster.

## Step 9 – Query Pod status and restart information

Run the following query in Log Analytics:

```kusto
KubePodInventory
| where TimeGenerated > ago(2h)
| where Namespace == "ci-debug-lab"
| where Name == "failing-demo-pod"
| summarize arg_max(TimeGenerated, *) by PodUid, ContainerID
| project TimeGenerated,
          ClusterName,
          Namespace,
          Name,
          PodStatus,
          ContainerName,
          ContainerStatus,
          ContainerStatusReason,
          ContainerRestartCount,
          PodRestartCount,
          Computer
| order by TimeGenerated desc
```

**What to inspect:**

- `PodStatus` – the last observed Kubernetes Pod phase.
- `ContainerStatus` – the container's last observed state, such as `waiting` or `terminated`.
- `ContainerStatusReason` – the reason associated with the container state, where available.
- `ContainerRestartCount` and `PodRestartCount` – restart counts recorded by Container Insights.
- `Computer` – the node on which the Pod is running.

The inventory is collected periodically, so its latest record may lag behind the live `kubectl` output. Also note that the Kubernetes **Pod phase** can remain `Running` while the container is in a waiting state with `ContainerStatusReason` set to `CrashLoopBackOff`; use the container status and reason fields to diagnose the restart loop.

## Step 10 – Query application logs from ContainerLogV2

```kusto
ContainerLogV2
| where TimeGenerated > ago(2h)
| where PodNamespace == "ci-debug-lab"
| where PodName == "failing-demo-pod"
| project TimeGenerated,
          PodNamespace,
          PodName,
          ContainerName,
          LogSource,
          LogLevel,
          LogMessage
| order by TimeGenerated desc
```

**What to inspect:**

- `LogMessage` – the message emitted by the application.
- `LogSource` – whether the message came from standard output (`stdout`) or standard error (`stderr`).
- `TimeGenerated` – when the log record was generated.

Filter the query to show only the simulated error:

```kusto
ContainerLogV2
| where TimeGenerated > ago(2h)
| where PodNamespace == "ci-debug-lab"
| where PodName == "failing-demo-pod"
| where tostring(LogMessage) contains "ERROR"
| project TimeGenerated, ContainerName, LogSource, LogMessage
| order by TimeGenerated desc
```

This query narrows the output to the application error message that explains why the container exits.

## Step 11 – Inspect Kubernetes warning events

```kusto
KubeEvents
| where TimeGenerated > ago(2h)
| where Namespace == "ci-debug-lab"
| where Name == "failing-demo-pod"
| project TimeGenerated,
          KubeEventType,
          ObjectKind,
          Name,
          Reason,
          Message,
          SourceComponent,
          Count
| order by TimeGenerated desc
```

**What to inspect:**

- `Reason` – the event reason, such as `BackOff`.
- `Message` – the event details describing the repeated restart/back-off behaviour.
- `KubeEventType` – the event type, commonly `Warning` for problematic conditions.
- `SourceComponent` – the Kubernetes component that generated the event.

> **Note:** Container Insights does not collect Kubernetes `Normal` event types by default in the standard configuration. A missing `Normal` event does not mean that no event occurred. Warning events associated with failures are the primary focus of this exercise.

## Step 12 – Analyse restart behaviour over time

```kusto
KubePodInventory
| where TimeGenerated > ago(2h)
| where Namespace == "ci-debug-lab"
| where Name == "failing-demo-pod"
| summarize MaxContainerRestarts = max(ContainerRestartCount),
            MaxPodRestarts = max(PodRestartCount)
    by bin(TimeGenerated, 1m)
| order by TimeGenerated asc
| render timechart
```

This query groups inventory snapshots into one-minute intervals and visualizes the observed restart counts. The values are monitoring snapshots, not a real-time counter.

## Step 13 – Correlate the evidence and identify the root cause

Use the following table to connect the observations from Kubernetes and Log Analytics.

| Evidence | Expected observation | Interpretation |
|---|---|---|
| `kubectl get pod` | `CrashLoopBackOff` | Kubernetes repeatedly restarts the container after it exits. |
| `kubectl describe pod` | Restart count increases; recent `BackOff` events may appear | The container repeatedly terminates and Kubernetes applies restart back-off. |
| `kubectl logs --previous` | `ERROR: required configuration file /app/config.yaml was not found` | The application reports the simulated configuration problem. |
| `KubePodInventory` | Container status/reason and restart count are recorded | Centralized inventory confirms the workload's unhealthy state and restart activity. |
| `ContainerLogV2` | Error message appears in the log records | Centralized application logs identify the reason the process exits. |
| `KubeEvents` | Warning event such as `BackOff` | Kubernetes reports repeated restart/back-off behaviour. |

**Root cause:** The sample application is configured to exit because its required configuration file is missing. Kubernetes detects the process termination and repeatedly restarts the container, resulting in `CrashLoopBackOff`.

## Step 14 – Correct the sample workload and verify recovery

The sample failure is intentional. Delete the failing Pod and recreate it with a command that remains running.

```bash
kubectl delete pod "$POD_NAME" --namespace "$NAMESPACE"
```

Apply the corrected Pod:

```bash
cat <<YAML | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: failing-demo-pod
  namespace: ${NAMESPACE}
  labels:
    app: failing-demo
spec:
  containers:
    - name: demo-container
      image: busybox:1.36.1
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "Application started successfully"
          sleep 3600
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
        limits:
          cpu: 100m
          memory: 64Mi
YAML
```

Verify that the new Pod is running:

```bash
kubectl get pod "$POD_NAME" --namespace "$NAMESPACE" --output wide
```

Retrieve its current application log:

```bash
kubectl logs "$POD_NAME" --namespace "$NAMESPACE" --container demo-container
```

Run the KQL queries again after telemetry has had time to arrive. The new Pod has a new Pod UID even though it uses the same Pod name; the earlier records and error logs can remain in Log Analytics within the configured retention period.

## Step 15 – Clean up lab resources

Delete the lab namespace and all resources created in it:

```bash
kubectl delete namespace "$NAMESPACE"
```

This removes the failing sample Pod and the corrected Pod. It does not disable Container Insights or delete the Log Analytics workspace.

---

## Troubleshooting

| Issue | Possible cause | Action |
|---|---|---|
| `az monitor log-analytics workspace show` fails | Incorrect workspace name/resource group, wrong subscription or insufficient permissions | Verify the values, select the correct subscription and confirm access to the workspace. |
| Monitoring add-on is already enabled | Container Insights is already configured | Confirm that its workspace is the intended workspace; do not enable it again unnecessarily. |
| Monitoring enable command fails | Missing permissions, unsupported cluster state or workspace configuration issue | Review the Azure CLI error and ask an administrator to verify AKS/workspace permissions and cluster health. |
| `kubectl get nodes` fails | Credentials or Kubernetes API connectivity issue | Run `az aks get-credentials` again and verify the selected subscription, network access and cluster state. |
| `ContainerLogV2` table is not found | ContainerLogV2 collection is not configured, the wrong workspace is open, or data has not arrived | Verify the monitoring add-on workspace and data collection configuration. Check whether the environment uses the legacy `ContainerLog` table and migrate/configure V2 as appropriate. |
| KQL query returns no records | Ingestion delay, time range too narrow, wrong namespace/Pod name, namespace filtering or wrong workspace | Widen the query time range, verify exact names with `kubectl`, and confirm the workspace and collection settings. |
| No `KubeEvents` result appears | Event has not been ingested, wrong time range, or only Normal events are being searched | Search for the Pod name and namespace over a wider time window. Normal events are not collected by default; this lab expects a Warning event such as `BackOff`. |
| `ContainerLogV2` contains no error message | Pod has not yet started, log collection is not enabled for that namespace, or the wrong Pod/container is queried | Check `kubectl logs --previous`, verify the Pod name and namespace, and confirm ContainerLogV2 is included in the data collection rule. |
| The Pod is not in `CrashLoopBackOff` yet | The first restart or back-off interval has not completed | Wait and watch with `kubectl get pod --watch`; restart timing varies. |

## Completion checklist

- [ ] Confirmed the active Azure subscription.
- [ ] Retrieved the existing Log Analytics workspace resource ID.
- [ ] Enabled or verified the AKS monitoring add-on.
- [ ] Connected `kubectl` to the AKS cluster.
- [ ] Created the `ci-debug-lab` namespace.
- [ ] Deployed the deliberately failing Pod.
- [ ] Observed `CrashLoopBackOff` and inspected Pod details.
- [ ] Queried `KubePodInventory` for Pod and restart information.
- [ ] Queried `ContainerLogV2` for the simulated application error.
- [ ] Queried `KubeEvents` for warning events.
- [ ] Identified the simulated configuration-file failure.
- [ ] Recreated the Pod with a corrected command and verified it was running.
- [ ] Deleted the lab namespace.

## Knowledge check

1. What is the purpose of Azure Monitor Container Insights in an AKS environment?
2. Which table should you query to view Pod status and restart counts?
3. Which table contains container stdout and stderr log records when the V2 schema is enabled?
4. Which KQL column contains the application log text in `ContainerLogV2`?
5. What does `CrashLoopBackOff` indicate?
6. Why might `KubeEvents` not show Normal events?
7. Why can `kubectl logs --previous` be useful when a container is restarting?
8. Why might a Pod created a few seconds ago not yet appear in Log Analytics?

## Key takeaways

- Container Insights centralizes AKS container logs, Pod inventory and Kubernetes event data in Azure Monitor Logs.
- `KubePodInventory` helps investigate Pod state and restart behaviour.
- `ContainerLogV2` helps identify application-level errors written to stdout and stderr.
- `KubeEvents` provides Kubernetes event context, including warning events such as container back-off.
- Combining `kubectl` inspection with KQL provides both immediate cluster details and centralized historical troubleshooting evidence.

## Official references

- [Enable monitoring for AKS clusters – Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable)
- [Query container logs in Azure Monitor – Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-log-query)
- [ContainerLogV2 table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/containerlogv2)
- [KubePodInventory table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/kubepodinventory)
- [KubeEvents table reference](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/kubeevents)
- [Use Kubernetes events for troubleshooting AKS](https://learn.microsoft.com/en-us/azure/aks/events)
