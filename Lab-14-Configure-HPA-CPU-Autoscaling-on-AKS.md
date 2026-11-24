# Lab 14 – Configure Horizontal Pod Autoscaler (HPA) Using CPU Utilization on AKS

## Lab overview

**Objective:** Configure a Horizontal Pod Autoscaler (HPA) for a Kubernetes Deployment using CPU utilization. Inspect the minimum and maximum replica settings, generate application load, and observe the current and desired replica counts.

**Environment:** Azure Portal → Cloud Shell → Bash  
**Estimated duration:** 20–25 minutes (autoscaling and scale-down observations can take a few minutes)  
**Workload:** `php-apache` sample application  
**HPA configuration:** Minimum 1 replica, maximum 5 replicas, target average CPU utilization 50%

## Concepts

- **Horizontal Pod Autoscaler (HPA):** A Kubernetes controller that increases or decreases the number of Pod replicas based on observed metrics.
- **CPU utilization target:** HPA compares Pod CPU usage with the CPU request defined in the Pod specification. For example, 50% of a `200m` CPU request is `100m` of CPU usage per Pod on average.
- **Metrics Server:** Provides CPU and memory metrics through the Kubernetes Metrics API. HPA needs this metrics data to make CPU-based scaling decisions.
- **Minimum and maximum replicas:** `minReplicas` and `maxReplicas` define the lower and upper limits within which HPA can scale the target workload.

## Prerequisites

- An existing, running AKS cluster and permission to create Deployments, Services, Pods and HorizontalPodAutoscalers in a namespace.
- Azure Portal access and Azure Cloud Shell opened in **Bash** mode.
- Internet access from the AKS nodes to pull the sample container images.
- Sufficient free CPU and node capacity to schedule up to five application Pods and a load-generator Pod.

> **Important:** Run this exercise in a training or non-production cluster. The load-generator creates repeated requests to increase CPU usage. Do not run it against a production workload.

---

## Step 1 – Connect to AKS from Azure Cloud Shell

Open **Azure Portal → Cloud Shell**, select **Bash**, and connect to the required AKS cluster if it is not already configured.

```bash
az account show --output table
# Run az login only if Cloud Shell is not authenticated.
# az login
az aks get-credentials --resource-group <RESOURCE_GROUP_NAME> --name <AKS_CLUSTER_NAME>
kubectl config current-context
kubectl get nodes
```

Replace the resource group and cluster placeholders with the values for your AKS environment.

**Command explanation:** `az aks get-credentials` downloads and merges the AKS access credentials into the Cloud Shell kubeconfig. `kubectl get nodes` confirms that the current context can communicate with the cluster.

## Step 2 – Create a dedicated namespace

```bash
kubectl create namespace aks-hpa-lab
```

**Explanation:** A namespace isolates the Deployment, Service, HPA and load-generator Pod created for this exercise.

Set the namespace as a Bash variable to keep subsequent commands short:

```bash
NAMESPACE=aks-hpa-lab
```

## Step 3 – Verify that CPU metrics are available

```bash
kubectl top nodes
kubectl get apiservice v1beta1.metrics.k8s.io
```

**Explanation:** `kubectl top` reads current resource usage from the Kubernetes Metrics API. The APIService check confirms that the resource metrics API is registered.

**Expected result:** `kubectl top nodes` displays CPU and memory usage for the AKS nodes, and the Metrics APIService should show `True` under `AVAILABLE`.

> **Note:** AKS provides Metrics Server integration for resource metrics. If these checks fail, do not install a second Metrics Server blindly. Check the cluster's Metrics Server/APIService health or contact the cluster administrator before continuing.

## Step 4 – Create the sample application Deployment and Service

The sample application performs CPU-intensive work when it receives HTTP requests, making it useful for demonstrating CPU-based autoscaling.

Create the manifest: **vi php-apache.yaml**

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: aks-hpa-lab
spec:
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
          image: registry.k8s.io/hpa-example:latest
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
  selector:
    app: php-apache
  ports:
    - name: http
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply the manifest and wait for the Deployment to become available:

```bash
kubectl apply -f php-apache.yaml
kubectl rollout status deployment/php-apache -n "$NAMESPACE" --timeout=180s
kubectl get deployment,pods -n "$NAMESPACE"
```

**Command explanation:** `kubectl apply` creates the Deployment and its internal ClusterIP Service. `kubectl rollout status` waits for the application Pod to become ready.

> **Why define a CPU request?** CPU utilization for a resource-based HPA is calculated relative to the CPU request. Without a CPU request for the application container, HPA cannot calculate CPU utilization correctly.

## Step 5 – Create the HPA

Create an HPA manifest using the `autoscaling/v2` API:

Create a file: **vi php-apache-hpa.yaml**

```bash
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
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
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
```

Apply the HPA:

```bash
kubectl apply -f php-apache-hpa.yaml
```

**Explanation:** This HPA targets the `php-apache` Deployment. Kubernetes attempts to keep average CPU utilization near 50% of the Pods' CPU requests, while keeping the replica count between one and five. The 60-second scale-down stabilization window is shortened for this training demonstration; production workloads should use a value appropriate to their traffic pattern.

## Step 6 – Inspect HPA configuration and current status

```bash
kubectl get hpa -n "$NAMESPACE"
kubectl describe hpa php-apache-hpa -n "$NAMESPACE"
kubectl get hpa php-apache-hpa -n "$NAMESPACE" -o yaml
```

Check these fields:

| Field | What to observe |
|---|---|
| `REFERENCE` | Deployment targeted by the HPA (`php-apache`) |
| `TARGETS` | Current average CPU utilization compared with the 50% target |
| `MINPODS` | Minimum replica count: `1` |
| `MAXPODS` | Maximum replica count: `5` |
| `REPLICAS` | Current number of replicas reported by the HPA |
| `desiredReplicas` | Number of replicas HPA currently wants the Deployment to run |

The CPU target may initially show `<unknown>` while metrics are being collected. Wait briefly and check again; proceed to load generation only after metrics are available.

## Step 7 – Generate CPU load

Create a temporary BusyBox Pod that continuously sends HTTP requests to the application through its ClusterIP Service:

```bash
kubectl run load-generator \
  --namespace="$NAMESPACE" \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- /bin/sh -c 'while true; do wget -q -O- http://php-apache >/dev/null; done'
```

**Explanation:** The load-generator sends repeated requests to the `php-apache` Service. The sample application consumes CPU while processing requests, allowing HPA to observe CPU utilization above the configured target.

Verify that the load-generator Pod is running:

```bash
kubectl get pods -n "$NAMESPACE"
```

## Step 8 – Observe automatic scaling

Watch the HPA status:

```bash
kubectl get hpa php-apache-hpa -n "$NAMESPACE" --watch
```

Press **Ctrl+C** to stop watching after you have observed the replica count increase.

In another Cloud Shell command, inspect the Deployment and Pods:

```bash
kubectl get deployment php-apache -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
kubectl top pods -n "$NAMESPACE"
kubectl describe hpa php-apache-hpa -n "$NAMESPACE"
```

**Expected observation:** CPU utilization should rise above the 50% target and HPA should increase the Deployment's replica count. The exact replica count and time required depend on cluster capacity, node performance and the metrics collected; the HPA will not necessarily scale to five replicas.

> **Important:** HPA does not create additional AKS nodes. If there is insufficient node capacity, additional Pods may remain `Pending` until capacity becomes available. Node-level scaling is handled separately by Cluster Autoscaler or another node-provisioning mechanism.

## Step 9 – Stop the load and observe scale-down

Delete the load-generator Pod:

```bash
kubectl delete pod load-generator -n "$NAMESPACE"
```

Watch the HPA and Deployment while CPU usage decreases:

```bash
kubectl get hpa php-apache-hpa -n "$NAMESPACE" --watch
```

In another command, verify the Deployment replica count:

```bash
kubectl get deployment php-apache -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE"
```

**Expected observation:** After CPU utilization falls and the scale-down stabilization period passes, HPA should reduce the replica count toward the minimum of one. This can take a few minutes because HPA evaluates metrics periodically.

## Step 10 – Clean up lab resources

```bash
kubectl delete namespace "$NAMESPACE"
rm -f php-apache.yaml php-apache-hpa.yaml
```

**Explanation:** Deleting the namespace removes the sample Deployment, Service, HPA and load-generator Pod. The local manifest files are also removed from the Cloud Shell session.

---

## Troubleshooting

| Issue | Possible reason | Action |
|---|---|---|
| `kubectl top nodes` returns `Metrics API not available` | Metrics API/metrics-server is unavailable or unhealthy | Check `kubectl get apiservice v1beta1.metrics.k8s.io` and the cluster's Metrics Server health; ask the AKS administrator to investigate. |
| HPA `TARGETS` shows `<unknown>` | Metrics are not available yet, or CPU requests are missing | Confirm Metrics API availability and verify that the target container defines `resources.requests.cpu`. |
| HPA does not scale up | Load-generator is not running, CPU is below target, or metrics have not updated | Check the load-generator Pod, run `kubectl top pods`, and inspect `kubectl describe hpa`. |
| Pods remain `Pending` after HPA scales out | Insufficient schedulable node capacity or resource quota | Run `kubectl describe pod` and inspect scheduling events; provide more node capacity or adjust the lab replica limit. |
| Image pull fails | Node cannot reach the image registry or image pull is temporarily unavailable | Run `kubectl describe pod` and inspect events; verify registry connectivity and retry when available. |
| Scale-down takes longer than expected | HPA stabilization and metric collection intervals | Stop the load, wait a few minutes, and inspect HPA conditions and events. |

Useful diagnostic commands:

```bash
kubectl describe hpa php-apache-hpa -n "$NAMESPACE"
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp'
kubectl top pods -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
```

## Completion checklist

- [ ] Connected to the AKS cluster from Azure Cloud Shell (Bash).
- [ ] Verified that the Metrics API is available.
- [ ] Deployed the sample application with a CPU request.
- [ ] Created an HPA with minimum one and maximum five replicas.
- [ ] Inspected the HPA target CPU utilization and replica configuration.
- [ ] Generated application load and observed HPA scale-out.
- [ ] Stopped the load and observed HPA scale-down toward the minimum.
- [ ] Deleted the lab namespace and temporary files.

## Knowledge check

1. What is the role of the Horizontal Pod Autoscaler in Kubernetes?
2. Why must the application container define a CPU request for CPU-utilization-based HPA?
3. In this lab, what do `minReplicas: 1` and `maxReplicas: 5` control?
4. Does a CPU target of 50% mean that a Pod is limited to 50% of one CPU core?
5. What is the difference between HPA scaling Pods and Cluster Autoscaler scaling AKS nodes?
6. Why might HPA show a desired replica count greater than the number of ready Pods?

## References

- [Microsoft Learn – Horizontal Pod Autoscaling (HPA) on AKS](https://learn.microsoft.com/en-us/azure/aks/horizontal-pod-autoscaler)
- [Kubernetes – HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Kubernetes – kubectl autoscale reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_autoscale/)
- [Microsoft Learn – Optimize AKS usage and costs](https://learn.microsoft.com/en-us/azure/aks/optimize-aks-costs)
