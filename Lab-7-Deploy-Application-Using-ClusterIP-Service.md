# Lab 7 – Deploy an Application and Access It Using a ClusterIP Service on AKS

## Problem Statement

Deploy an NGINX application on AKS and expose it internally using a Kubernetes `ClusterIP` Service. From a separate Pod running in the same cluster, access the application using the Service name and verify that Kubernetes DNS resolves the Service correctly.

## Objectives

- Understand how a Kubernetes Service provides a stable network endpoint for Pods.
- Create a `ClusterIP` Service for an application Deployment.
- Use Kubernetes Service discovery to communicate between Pods by Service name.
- Verify Service endpoints and successful HTTP communication from inside the cluster.

## Key Concepts

| Concept | Explanation |
|---|---|
| **Pod** | The smallest deployable Kubernetes unit. It runs one or more containers and receives a cluster-internal IP address. |
| **Deployment** | Manages application Pods and helps maintain the desired number of replicas. |
| **Service** | Provides a stable network endpoint and forwards traffic to matching, ready Pods. |
| **ClusterIP** | The default Service type. It exposes an application on an IP address reachable only from within the Kubernetes cluster. |
| **Service discovery** | Kubernetes DNS lets Pods find Services by name instead of depending on changing Pod IP addresses. |
| **Service selector** | A label selector used by a Service to identify the Pods that should receive traffic. |

## Prerequisites

- An existing AKS cluster in a running state.
- Access to the AKS cluster with permission to create and view namespaces, Deployments, Services and Pods.
- Azure Portal access and Cloud Shell with **Bash** selected.
- The cluster must be able to pull the public NGINX and curl client images, or equivalent approved images must be available in your registry.

> **Important:** A `ClusterIP` Service is reachable from Pods inside the cluster. Do not try to open its ClusterIP directly from your laptop or Cloud Shell browser session. This lab uses a temporary client Pod to test access from inside AKS.

---

## Lab Steps

### Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Open **Cloud Shell** and select **Bash**.
3. Set the AKS resource group and cluster name. Replace the sample values with your environment details.

```bash
RESOURCE_GROUP="myResourceGroup"
AKS_CLUSTER="myAKSCluster"
NAMESPACE="clusterip-lab"
```

4. Retrieve the AKS credentials and configure `kubectl`.

```bash
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER"
```

This command downloads the cluster credentials and adds the AKS context to the current kubeconfig file.

5. Verify that `kubectl` can reach the cluster.

```bash
kubectl get nodes
kubectl config current-context
```

The nodes should show `Ready`, and the current context should refer to the intended AKS cluster.

### Step 2 – Create a namespace

```bash
kubectl create namespace "$NAMESPACE"
```

A namespace provides a logical boundary for the resources created in this lab.

Verify the namespace:

```bash
kubectl get namespace "$NAMESPACE"
```

### Step 3 – Deploy the NGINX application

Create a Deployment with two NGINX replicas:

```bash
kubectl create deployment nginx-web \
  --image=nginx:stable-alpine \
  --replicas=2 \
  --namespace="$NAMESPACE"
```

The Deployment creates and manages two NGINX Pods. The Pods are selected for the Service in the next step using the Deployment's `app=nginx-web` label.

Wait for the Pods to become ready:

```bash
kubectl rollout status deployment/nginx-web \
  --namespace="$NAMESPACE" \
  --timeout=120s
```

This command waits until the Deployment has successfully rolled out or reports a timeout.

Check the Pods and their labels:

```bash
kubectl get pods --namespace="$NAMESPACE" --show-labels -o wide
```

Confirm that both Pods are in the `Running` state and have a `Ready` status of `1/1`.

### Step 4 – Create a ClusterIP Service

Expose the Deployment internally on TCP port 80:

```bash
kubectl expose deployment nginx-web \
  --name=nginx-service \
  --type=ClusterIP \
  --port=80 \
  --target-port=80 \
  --namespace="$NAMESPACE"
```

The Service selects the Deployment's Pods and forwards traffic received on Service port `80` to container port `80`. `ClusterIP` ensures the Service is internal to the cluster.

Verify the Service:

```bash
kubectl get service nginx-service --namespace="$NAMESPACE" -o wide
```

Expected observation:

- `TYPE` is `ClusterIP`.
- `CLUSTER-IP` contains an internal IP address.
- `PORT(S)` shows `80/TCP`.
- `EXTERNAL-IP` is not assigned.

Inspect the Service configuration and selector:

```bash
kubectl describe service nginx-service --namespace="$NAMESPACE"
```

The selector should match the application Pods (typically `app=nginx-web`).

### Step 5 – Verify the Service endpoints

```bash
kubectl get endpointslices \
  --namespace="$NAMESPACE" \
  --selector="kubernetes.io/service-name=nginx-service" \
  -o wide
```

EndpointSlices show the Pod IP addresses and ports registered as backends for the Service. Confirm that the ready NGINX Pod endpoints are listed.

### Step 6 – Create a client Pod inside the cluster

Create a temporary Pod containing the `curl` utility:

```bash
kubectl run curl-client \
  --namespace="$NAMESPACE" \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --command -- sleep 3600
```

The client Pod is separate from the NGINX application Pods. It will send an HTTP request to the Service from inside the AKS cluster.

Wait until the client Pod is ready:

```bash
kubectl wait \
  --namespace="$NAMESPACE" \
  --for=condition=Ready pod/curl-client \
  --timeout=120s
```

Verify that it is running:

```bash
kubectl get pod curl-client --namespace="$NAMESPACE" -o wide
```

### Step 7 – Access NGINX using the Service name

Run an HTTP request from the client Pod using the short Service name:

```bash
kubectl exec \
  --namespace="$NAMESPACE" \
  curl-client -- \
  curl --include --max-time 10 http://nginx-service
```

The request uses `nginx-service` rather than a Pod IP or the Service's ClusterIP. Kubernetes DNS resolves the Service name, and the Service forwards the request to one of its ready backend Pods.

Expected result: an HTTP response similar to the following, followed by the NGINX welcome page HTML.

```text
HTTP/1.1 200 OK
Server: nginx
Content-Type: text/html
```

You can also test the Service using its fully qualified DNS name:

```bash
kubectl exec \
  --namespace="$NAMESPACE" \
  curl-client -- \
  curl --include --max-time 10 \
  "http://nginx-service.${NAMESPACE}.svc.cluster.local"
```

The standard Kubernetes Service DNS format is:

```text
<service-name>.<namespace>.svc.cluster.local
```

The short name `nginx-service` works here because the client Pod and Service are in the same namespace. The fully qualified name identifies the Service and its namespace explicitly.

### Step 8 – Review the resources

```bash
kubectl get deployment,pods,service --namespace="$NAMESPACE" -o wide
```

Confirm the following:

- The NGINX Deployment has two available replicas.
- The NGINX Pods are running and ready.
- The `nginx-service` Service has type `ClusterIP`.
- The Service has ready backend endpoints.
- The client Pod successfully received an HTTP `200 OK` response from NGINX.

---

## Troubleshooting

| Issue | What to check | Suggested command |
|---|---|---|
| NGINX Pod is `Pending` | Node capacity, quotas and scheduling events. | `kubectl describe pod <pod-name> -n "$NAMESPACE"` |
| NGINX Pod shows `ImagePullBackOff` | Image name, registry access and cluster outbound connectivity. | `kubectl describe pod <pod-name> -n "$NAMESPACE"` |
| Service has no endpoints | Service selector must match the labels on ready Pods. | `kubectl describe svc nginx-service -n "$NAMESPACE"` and `kubectl get pods -n "$NAMESPACE" --show-labels` |
| `curl` cannot resolve `nginx-service` | Confirm the client Pod and Service are in the same namespace and cluster DNS is healthy. | `kubectl get pods -n kube-system` |
| Connection times out or is refused | Check Service port, target port, Pod readiness and any NetworkPolicies. | `kubectl describe svc nginx-service -n "$NAMESPACE"` |

## Cleanup

Delete the lab namespace and all resources created inside it:

```bash
kubectl delete namespace "$NAMESPACE"
```

This removes the NGINX Deployment, ClusterIP Service, client Pod and other resources in the lab namespace. It does **not** delete the AKS cluster.

## Completion Checklist

- [ ] Connected to the AKS cluster from Azure Portal Cloud Shell (Bash).
- [ ] Created the `clusterip-lab` namespace.
- [ ] Deployed NGINX with two ready replicas.
- [ ] Created a `ClusterIP` Service on port 80.
- [ ] Verified the Service selector and EndpointSlices.
- [ ] Created a client Pod inside the cluster.
- [ ] Accessed NGINX using the Kubernetes Service name.
- [ ] Received an HTTP `200 OK` response.
- [ ] Deleted the lab namespace.

## Knowledge Check

1. What problem does a Kubernetes Service solve when Pods can be recreated and receive new IP addresses?
2. Why is a `ClusterIP` Service not directly accessible from a user's laptop or Cloud Shell?
3. What is the purpose of the Service selector?
4. Why can the client Pod use `nginx-service` instead of the Service's IP address?
5. What does an empty EndpointSlice for this Service indicate?

## Key Takeaways

- A `ClusterIP` Service provides an internal, stable endpoint for an application.
- A Service routes traffic to ready Pods that match its selector.
- Kubernetes DNS enables Service-to-Service communication by name.
- A Pod inside the cluster can access a same-namespace Service using its short name.
- `kubectl get service` and EndpointSlices help validate the Service configuration and its backends.

## References

- [Kubernetes – Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes – DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes – kubectl expose](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/)
- [Microsoft Learn – Connect to an AKS cluster](https://learn.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal)
