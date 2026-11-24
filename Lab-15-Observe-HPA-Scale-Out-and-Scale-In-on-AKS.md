# Lab 15 – Configure HPA and Observe Automatic Pod Scaling on AKS

## Lab overview

| Item | Details |
|---|---|
| **Platform** | Azure Kubernetes Service (AKS) |
| **Execution environment** | Azure Portal → Cloud Shell → Bash |
| **Estimated duration** | 20–25 minutes |
| **Difficulty** | Intermediate |
| **Workload** | PHP-Apache sample application |
| **Autoscaling metric** | Average CPU utilization |

### Problem statement

Configure a Horizontal Pod Autoscaler (HPA) for an application, generate workload traffic, and observe how Kubernetes automatically increases or decreases the number of Pod replicas as application demand changes.

### Objectives

- Understand how HPA uses CPU metrics to determine the desired number of Pod replicas.
- Configure minimum and maximum replica counts and a target CPU utilization.
- Generate HTTP traffic from an in-cluster load-generator Pod.
- Observe automatic scale-out when CPU utilization increases and scale-in after traffic stops.

### Expected outcome

Participants will be able to configure and validate HPA-based scaling, interpret current CPU utilization and desired replica counts, and explain the scale-out and scale-in behaviour of a Kubernetes Deployment.

---

## 1. Background concepts

**Horizontal Pod Autoscaler (HPA):** A Kubernetes controller that adjusts the replica count of a scalable workload, such as a Deployment, according to observed metrics and the configured target.

**CPU utilization target:** HPA calculates CPU utilization as a percentage of the CPU request configured for the workload's containers. A CPU request is required when using a percentage-based CPU target.

**Scale-out and scale-in:** Scale-out adds Pod replicas when measured demand is above the target. Scale-in removes excess replicas when demand falls, subject to HPA stabilization and scaling rules.

**Metrics Server:** Provides CPU and memory resource metrics through the Kubernetes Metrics API. HPA needs these metrics to make CPU-based scaling decisions.

## 2. Prerequisites

- An existing AKS cluster is running and accessible.
- The participant has permission to create Deployments, Services, Pods and HPAs in the cluster.
- Azure Portal Cloud Shell is opened in **Bash** mode.
- The AKS cluster has available CPU capacity for up to five application replicas and one small load-generator Pod.
- The Kubernetes Metrics API is available.

> **Important:** This lab creates Kubernetes resources in the selected cluster. Use a training or non-production cluster. The exercise scales Pods, not AKS nodes. If the cluster has insufficient node capacity, some new Pods may remain `Pending` until capacity becomes available.

---

## 3. Connect to the AKS cluster

In Azure Portal, open **Cloud Shell**, select **Bash**, and run the following commands.

### 3.1 Select the subscription and AKS cluster

```bash
az account show --output table
```

Displays the currently selected Azure subscription and confirms the Cloud Shell Azure CLI session.

```bash
az aks get-credentials --resource-group <RESOURCE_GROUP> --name <AKS_CLUSTER_NAME> --overwrite-existing
```

Downloads the AKS user credentials and configures the current `kubectl` context. Replace the placeholders with the actual resource group and cluster names.

```bash
kubectl get nodes
```

Confirms that `kubectl` can communicate with the AKS API server and that cluster nodes are visible.

### 3.2 Set the lab namespace

```bash
NAMESPACE=aks-hpa-lab
```

Stores the namespace name in a Bash variable so the commands in this lab consistently use the same namespace.

```bash
kubectl create namespace "$NAMESPACE"
```

Creates an isolated namespace for the application, HPA and load-generator resources.

## 4. Verify that CPU metrics are available

```bash
kubectl get apiservice v1beta1.metrics.k8s.io
```

Checks whether the Kubernetes Metrics API is registered and available. The `AVAILABLE` column should show `True`.

```bash
kubectl top nodes
```

Displays current node CPU and memory usage, confirming that resource metrics can be queried.

> **Expected result:** The Metrics APIService is available and `kubectl top nodes` returns usage values. If metrics are temporarily unavailable, wait briefly and retry. Do not continue with the HPA exercise until CPU metrics are available; HPA cannot calculate CPU utilization without them.

## 5. Deploy the sample application

The sample application uses the `registry.k8s.io/hpa-example` image. Its container requests `200m` CPU, which allows HPA to calculate CPU utilization as a percentage of the requested CPU.

### 5.1 Create the Deployment and Service manifest

Create a file: **vi php-apache.yaml**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: aks-hpa-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 200m
            limits:
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: aks-hpa-lab
spec:
  type: ClusterIP
  selector:
    app: php-apache
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Creates a YAML file containing a one-replica Deployment and an internal ClusterIP Service. The Service gives the load-generator Pod a stable DNS name to call.

### 5.2 Apply the manifest

```bash
kubectl apply -f php-apache.yaml
```

Creates the application Deployment and its ClusterIP Service in the lab namespace.

```bash
kubectl rollout status deployment/php-apache -n "$NAMESPACE"
```

Waits until the initial application Deployment is successfully rolled out.

```bash
kubectl get deployment,pods,svc -n "$NAMESPACE"
```

Displays the Deployment, its Pod and the Service. Confirm that the initial Deployment has one available replica and the Pod is `Running`.

### 5.3 Confirm CPU metrics for the application Pod

```bash
kubectl top pods -n "$NAMESPACE"
```

Displays the current CPU and memory usage of the application Pod. The initial CPU usage should normally be low before load is generated.

## 6. Configure the Horizontal Pod Autoscaler

Create an HPA that maintains between **1 and 5 replicas**, targeting **50% average CPU utilization**.

### 6.1 Create the HPA manifest

Create a file: **vi php-apache-hpa.yaml**

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: aks-hpa-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

Defines an HPA that targets the `php-apache` Deployment. It uses CPU utilization and keeps the replica count between one and five.

### 6.2 Apply and inspect the HPA

```bash
kubectl apply -f php-apache-hpa.yaml
```

Creates the HorizontalPodAutoscaler resource.

```bash
kubectl get hpa -n "$NAMESPACE"
```

Displays the target CPU utilization, minimum and maximum replicas, and the current replica count.

```bash
kubectl describe hpa php-apache -n "$NAMESPACE"
```

Shows the HPA target, current metrics, conditions and recent scaling events. Check that the HPA can read CPU metrics and has no metric-related errors.

> **Understand the target:** Each application Pod requests `200m` CPU. A target of `50%` therefore represents an average CPU usage target of approximately `100m` per Pod. This is a utilization target, not a CPU limit.

## 7. Generate application traffic

Create a load-generator Pod in the same namespace. It continuously sends HTTP requests to the `php-apache` Service, increasing CPU demand on the application Pods.

### 7.1 Create the load-generator manifest

Create a file: **vi load-generator.yaml**

```bash
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
  namespace: aks-hpa-lab
  labels:
    app: load-generator
spec:
  restartPolicy: Always
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - |
          while true; do
            wget -q -O- http://php-apache >/dev/null
            sleep 0.01
          done
```

Defines a BusyBox Pod that repeatedly sends requests to the application Service. The short delay keeps requests continuous while avoiding an unnecessary tight loop.

### 7.2 Start the load generator

```bash
kubectl apply -f load-generator.yaml
```

Creates the load-generator Pod, which starts sending requests to the PHP-Apache application.

```bash
kubectl get pod load-generator -n "$NAMESPACE"
```

Confirms that the load-generator Pod is running.

## 8. Observe automatic scale-out

Run the following command to watch HPA status while the load-generator Pod is active.

```bash
kubectl get hpa php-apache -n "$NAMESPACE" --watch
```

Continuously displays the current CPU utilization, target CPU utilization and replica count. Allow time for metrics collection and the HPA control loop to react. Press **Ctrl+C** to stop watching; this does not stop the load-generator Pod.

In another command after stopping the watch, inspect the Deployment and Pods:

```bash
kubectl get deployment,pods -n "$NAMESPACE" -o wide
```

Shows how many application replicas have been created and whether the Pods are running and scheduled on nodes.

```bash
kubectl top pods -n "$NAMESPACE"
```

Displays CPU usage for individual Pods while traffic is being generated.

```bash
kubectl describe hpa php-apache -n "$NAMESPACE"
```

Shows the HPA's current metrics, desired replica count, conditions and scaling events.

> **Expected observation:** CPU utilization rises above the 50% target and HPA increases the desired replica count. The exact CPU percentage and number of replicas vary with cluster capacity, metric timing and request load. The HPA will not necessarily scale all the way to five replicas.

## 9. Stop the load and observe automatic scale-in

### 9.1 Stop generating traffic

```bash
kubectl delete pod load-generator -n "$NAMESPACE"
```

Deletes the load-generator Pod and stops the continuous HTTP requests to the application.

### 9.2 Watch the HPA scale down

```bash
kubectl get hpa php-apache -n "$NAMESPACE" --watch
```

Monitors CPU utilization and replica count after the load stops. CPU utilization should fall, and HPA should gradually reduce replicas toward the configured minimum of one.

> **Important:** Scale-in is not immediate. Kubernetes applies downscale stabilization to avoid removing Pods too quickly when metrics fluctuate. It can take several minutes for the Deployment to return to one replica.

### 9.3 Verify the final replica count

```bash
kubectl get deployment php-apache -n "$NAMESPACE"
```

Confirms that the Deployment has returned to one desired and available replica after CPU demand has subsided.

```bash
kubectl get hpa php-apache -n "$NAMESPACE"
```

Confirms that the HPA remains configured with a minimum of one and maximum of five replicas, and shows the latest CPU metric and current replica count.

---

## 10. Understand the HPA output

| HPA field | Meaning |
|---|---|
| **TARGETS** | Current average CPU utilization compared with the configured target (50%). |
| **MINPODS** | Minimum number of replicas HPA is allowed to maintain (1). |
| **MAXPODS** | Maximum number of replicas HPA is allowed to create (5). |
| **REPLICAS** | Current replica count reported for the target workload. |
| **DESIRED REPLICAS** | In `kubectl describe hpa`, the number of replicas HPA currently wants the Deployment to run. |

### Expected behaviour

| Stage | Load-generator status | Expected HPA behaviour |
|---|---|---|
| Initial | Not running | Deployment remains at one replica; CPU utilization is low. |
| Scale-out | Running | CPU utilization increases; HPA raises the desired replica count, up to five. |
| Scale-in | Deleted | CPU utilization falls; after the stabilization period, HPA reduces replicas toward one. |

The exact readings and timing will vary. Use the observed metrics and events from your cluster rather than expecting fixed sample numbers.

## 11. Troubleshooting

| Issue | Possible cause | Action |
|---|---|---|
| HPA shows `<unknown>` for CPU | Metrics API is unavailable or metrics have not been collected yet. | Run `kubectl get apiservice v1beta1.metrics.k8s.io` and `kubectl top pods -n "$NAMESPACE"`; wait briefly and retry. |
| HPA does not scale out | CPU has not exceeded the target, traffic is not reaching the Service, or metrics are unavailable. | Check `kubectl get pod -n "$NAMESPACE"`, `kubectl logs load-generator -n "$NAMESPACE"`, `kubectl top pods -n "$NAMESPACE"` and `kubectl describe hpa php-apache -n "$NAMESPACE"`. |
| Application Pod is `Pending` | Insufficient cluster CPU or memory capacity, or scheduling constraints. | Run `kubectl describe pod -n "$NAMESPACE" <POD_NAME>` and inspect the Events section. |
| Load-generator Pod is not running | Image pull issue or container startup failure. | Run `kubectl describe pod load-generator -n "$NAMESPACE"` and `kubectl logs load-generator -n "$NAMESPACE"`. |
| HPA does not scale down immediately | Downscale stabilization and metric collection intervals delay scale-in. | Confirm the load-generator is deleted and continue watching HPA status for several minutes. |
| `kubectl top` reports metrics unavailable | Metrics API has not become ready or is unhealthy. | Verify the Metrics APIService and cluster metrics components before continuing. |

## 12. Cleanup

Delete the lab namespace and all resources created inside it.

```bash
kubectl delete namespace "$NAMESPACE"
```

Removes the Deployment, Service, HPA and any remaining Pods created for this lab.

```bash
rm -f php-apache.yaml php-apache-hpa.yaml load-generator.yaml
```

Removes the local YAML files created in the Cloud Shell session.

---

## 13. Completion checklist

- [ ] Connected to the AKS cluster through Azure Portal Cloud Shell (Bash).
- [ ] Verified that the Kubernetes Metrics API is available.
- [ ] Deployed the PHP-Apache application with a CPU request.
- [ ] Created an HPA with a 50% CPU target, minimum one replica and maximum five replicas.
- [ ] Started an in-cluster load generator.
- [ ] Observed HPA increase the desired replica count under load.
- [ ] Stopped the load generator.
- [ ] Observed HPA reduce the replica count toward one after CPU usage decreased.
- [ ] Deleted the lab resources.

## 14. Knowledge check

1. Why must a CPU request be defined on the application container when HPA uses CPU utilization as a percentage?
2. What is the difference between the HPA target CPU utilization and the container CPU limit?
3. Which component provides the CPU metrics used by HPA?
4. What happens when CPU utilization rises above the configured target?
5. Why might scale-in take several minutes after the load-generator Pod is deleted?
6. Does HPA add AKS worker nodes, or does it change the number of application Pod replicas?

## References

- [Kubernetes – Horizontal Pod Autoscaler walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Kubernetes – Horizontal Pod Autoscaling concepts](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Microsoft Learn – AKS scaling overview](https://learn.microsoft.com/en-us/azure/aks/concepts-scale)
