# Lab 17 – Apply Default-Deny and Allow NetworkPolicies on AKS

## Lab scenario

You are deploying a backend application in an AKS cluster. By default, Pods in a Kubernetes cluster can communicate with other Pods unless a network policy restricts that traffic. Your task is to apply a default-deny ingress policy to a dedicated namespace and then allow only the selected frontend Pods to connect to the backend Pods on TCP port 80.

**Execution environment:** Azure Portal → Cloud Shell → Bash  
**Estimated duration:** 20–25 minutes  
**Prerequisites:** An existing AKS cluster, `kubectl` access, and an AKS network-policy engine that enforces Kubernetes `NetworkPolicy` resources.

## Objectives

- Understand how Kubernetes NetworkPolicies select Pods using labels.
- Apply a default-deny ingress policy to all Pods in a namespace.
- Add a specific allow rule for frontend-to-backend communication.
- Verify allowed and denied traffic from different Pods.

## Concepts

| Concept | Description |
|---|---|
| NetworkPolicy | A Kubernetes resource that controls ingress and/or egress traffic for selected Pods. |
| Default-deny ingress | A policy selecting all Pods in a namespace and allowing no incoming connections unless another applicable policy allows them. |
| Pod selector | A label-based selector used to identify the Pods to which a policy or rule applies. |
| Ingress rule | Defines which sources can connect to selected destination Pods, including optional protocol and port restrictions. |
| Policy behaviour | NetworkPolicies are additive. A connection is allowed when at least one policy selecting the destination Pod allows it, and no other applicable policy denies it through the combined policy model. |

> **Important:** Creating a NetworkPolicy object does not itself enforce traffic restrictions. The AKS cluster must have a supported network-policy engine enabled, such as Azure CNI Powered by Cilium, Azure Network Policy Manager (NPM), or Calico. Check the AKS resource in Azure Portal under **Networking → Network policy engine** before starting. If no supported engine is configured, stop here: the manifests can be created, but traffic might not be filtered. Do not change the network-policy engine on a shared training or production cluster as part of this lab; enabling or changing the engine can require node-pool reimaging and cause disruption.

## Architecture
```
                    AKS namespace: np-lab

       Frontend Deployment                 Debug Pod
        label: app=frontend               label: app=debug
                 |                              |
                 | HTTP TCP/80                  | HTTP TCP/80
                 v                              v
          Backend Service  <--------------------+
          backend:80                 (blocked after policies)
                 |
                 v
        Backend Deployment
         label: app=backend
         2 NGINX Pods
```

```
NetworkPolicies:
1. default-deny-ingress: selects every Pod in np-lab; denies all ingress.
2. allow-frontend-to-backend: selects app=backend; allows TCP/80
   only from Pods labelled app=frontend in the same namespace.
```

## Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Open **Cloud Shell** and select **Bash**.
3. Confirm that `kubectl` is connected to the intended AKS cluster.

```bash
kubectl config current-context
kubectl get nodes
```

These commands show the active Kubernetes context and verify that the cluster is reachable.

Set the lab namespace variable:

```bash
export NAMESPACE=np-lab
```

## Step 2 – Verify NetworkPolicy support

In the Azure portal, open the AKS resource and select **Networking**. Under **Network profile**, confirm that a network-policy engine is configured. Supported engines include Cilium, Azure Network Policy Manager, and Calico. Cilium is the recommended option for new Linux-based AKS deployments.

You can also inspect the network configuration reported by Azure CLI (replace the values with your cluster details):

```bash
export RESOURCE_GROUP=<your-resource-group>
export AKS_CLUSTER=<your-aks-cluster-name>
az aks show --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER" --query "networkProfile.{networkPlugin:networkPlugin,networkPolicy:networkPolicy,networkDataplane:networkDataplane}" --output yaml
```

This displays the cluster networking properties. Use the Azure portal's **Network policy engine** field and the current Microsoft documentation to confirm the effective engine for your cluster; property values vary by AKS networking mode.

**Do not continue unless a supported NetworkPolicy engine is enabled.**

## Step 3 – Create a dedicated namespace

```bash
kubectl create namespace "$NAMESPACE"
kubectl get namespace "$NAMESPACE"
```

A namespace provides a logical boundary for the lab resources and ensures that the default-deny policy does not affect workloads in other namespaces.

## Step 4 – Deploy the backend application and Service

Create a two-replica NGINX Deployment and an internal ClusterIP Service. The Service provides a stable DNS name (`backend`) for clients in the same namespace.

```bash
cat <<'BACKEND_YAML' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: np-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: np-lab
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
BACKEND_YAML
```

The Deployment creates two NGINX Pods labelled `app=backend`. The Service forwards TCP port 80 to those Pods.

Wait for the backend Pods to become ready and inspect the Service endpoints:

```bash
kubectl rollout status deployment/backend -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -l app=backend -o wide
kubectl get service backend -n "$NAMESPACE"
kubectl get endpointslices -n "$NAMESPACE" -l kubernetes.io/service-name=backend
```

Expected result: two backend Pods should be `Running` and `Ready`, and the EndpointSlice should list their Pod IP addresses on port 80.

## Step 5 – Deploy the frontend client and an unauthorized test Pod

The frontend Pod is labelled `app=frontend`; the debug Pod is labelled `app=debug`. Both run a lightweight curl container and remain idle so that you can execute connectivity tests from inside them.

```bash
cat <<'CLIENTS_YAML' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: np-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: curl
          image: curlimages/curl:8.12.1
          command: ["/bin/sh", "-c"]
          args: ["sleep 36000"]
          resources:
            requests:
              cpu: 25m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: debug
  namespace: np-lab
  labels:
    app: debug
spec:
  containers:
    - name: curl
      image: curlimages/curl:8.12.1
      command: ["/bin/sh", "-c"]
      args: ["sleep 36000"]
      resources:
        requests:
          cpu: 25m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 64Mi
CLIENTS_YAML
```

Wait for both test clients to become ready:

```bash
kubectl rollout status deployment/frontend -n "$NAMESPACE"
kubectl wait --for=condition=Ready pod/debug -n "$NAMESPACE" --timeout=120s
kubectl get pods -n "$NAMESPACE" --show-labels
```

## Step 6 – Verify connectivity before applying NetworkPolicies

Test the backend Service from both the frontend and debug Pods:

```bash
kubectl exec deployment/frontend -n "$NAMESPACE" -- curl -sS --max-time 5 http://backend
kubectl exec pod/debug -n "$NAMESPACE" -- curl -sS --max-time 5 http://backend
```

Expected result: both commands return the default NGINX welcome page. This confirms that the two clients can reach the backend before traffic restrictions are applied.

## Step 7 – Apply a default-deny ingress NetworkPolicy

The policy below selects **all Pods in the `np-lab` namespace** and defines ingress as the only policy type, with no ingress allow rules. Incoming connections to these Pods are therefore denied unless another applicable NetworkPolicy explicitly allows them. Egress traffic is not restricted by this policy.

```bash
cat <<'DENY_YAML' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: np-lab
spec:
  podSelector: {}
  policyTypes:
    - Ingress
DENY_YAML
```

Confirm that the policy exists:

```bash
kubectl get networkpolicy -n "$NAMESPACE"
kubectl describe networkpolicy default-deny-ingress -n "$NAMESPACE"
```

Test from the frontend Pod. At this point, the request should fail because no policy allows ingress to the backend Pods:

```bash
if kubectl exec deployment/frontend -n "$NAMESPACE" -- curl -sS --connect-timeout 2 --max-time 4 http://backend; then
  echo "Unexpected result: frontend reached backend before an allow policy was added."
else
  echo "Expected result: frontend-to-backend traffic is blocked by default-deny ingress."
fi
```

The connection usually times out because the policy engine drops the disallowed traffic. A timeout is expected here; it is not an application failure.

## Step 8 – Allow only frontend Pods to access backend Pods

Create a second NetworkPolicy. It selects only backend Pods and allows ingress on TCP port 80 from Pods labelled `app=frontend` in the same namespace.

```bash
cat <<'ALLOW_YAML' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: np-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
ALLOW_YAML
```

The `podSelector` under `spec` selects the destination Pods (`app=backend`). The `podSelector` under `from` selects the source Pods (`app=frontend`). Because the rule has no `namespaceSelector`, the source selector matches frontend Pods in the same namespace as this policy.

## Step 9 – Verify allowed and denied communication

### Test 1: Frontend to backend – should be allowed

```bash
kubectl exec deployment/frontend -n "$NAMESPACE" -- curl -sS --max-time 5 -o /dev/null -w 'HTTP status: %{http_code}\n' http://backend
```

Expected result:

```text
HTTP status: 200
```

The frontend Pods match the allowed source selector, so the request reaches the backend Service and one of its NGINX Pods.

### Test 2: Debug Pod to backend – should be denied

```bash
if kubectl exec pod/debug -n "$NAMESPACE" -- curl -sS --connect-timeout 2 --max-time 4 http://backend; then
  echo "Unexpected result: debug Pod reached backend. Check the NetworkPolicy engine and Pod labels."
else
  echo "Expected result: debug-to-backend traffic is blocked."
fi
```

The debug Pod has label `app=debug`, so it does not match the allowed source selector `app=frontend`. The default-deny ingress policy continues to block it.

### Test 3: Review labels and policies

```bash
kubectl get pods -n "$NAMESPACE" --show-labels
kubectl get networkpolicy -n "$NAMESPACE"
kubectl describe networkpolicy allow-frontend-to-backend -n "$NAMESPACE"
```

Confirm that:

- Backend Pods have label `app=backend`.
- The frontend Pod has label `app=frontend`.
- The debug Pod has label `app=debug`.
- Both NetworkPolicies exist in the `np-lab` namespace.

## Step 10 – Understand the final traffic behaviour

| Source | Destination | Protocol / Port | Expected result | Reason |
|---|---|---|---|---|
| Frontend Pod (`app=frontend`) | Backend Pods (`app=backend`) | TCP/80 | Allowed | Matches the explicit allow rule. |
| Debug Pod (`app=debug`) | Backend Pods (`app=backend`) | TCP/80 | Denied | Does not match the allowed source selector. |
| Any other Pod in `np-lab` | Any Pod in `np-lab` | Any ingress port | Denied unless separately allowed | Selected by the default-deny ingress policy. |
| Pods in other namespaces | Pods in `np-lab` | Any ingress port | Denied unless separately allowed | The default-deny policy selects all destination Pods in `np-lab`. |

**Note:** NetworkPolicies are additive. If you add another policy that selects the same backend Pods and allows additional sources, those sources are also allowed. Review all policies that select a workload when evaluating effective access.

## Step 11 – Clean up lab resources

Delete the lab namespace and all resources created inside it:

```bash
kubectl delete namespace "$NAMESPACE"
```

Verify that the namespace has been removed:

```bash
kubectl get namespace "$NAMESPACE"
```

A `NotFound` response confirms cleanup is complete.

## Troubleshooting

| Issue | Checks / action |
|---|---|
| Frontend cannot reach backend before policies are applied | Check that all Pods are Ready, the backend Service selects `app=backend`, and its EndpointSlice contains backend Pod IPs. |
| NetworkPolicy objects exist but traffic is not blocked | Confirm that a supported network-policy engine is enabled on the AKS cluster. Kubernetes accepts NetworkPolicy objects even when no engine is enforcing them. |
| Frontend remains blocked after the allow policy is applied | Check source and destination Pod labels, policy namespace, Service endpoints, TCP port 80, and the policy engine status. |
| Debug Pod unexpectedly reaches backend | Confirm the debug Pod is labelled `app=debug` and not `app=frontend`; inspect all NetworkPolicies because policies are additive. |
| `kubectl exec` reports that a Pod is not found or not ready | Run `kubectl get pods -n "$NAMESPACE" -o wide` and wait for the frontend and debug Pods to become Ready. |
| Service DNS name cannot be resolved | Check that the cluster DNS Pods are healthy and that the command uses `http://backend` from within the same namespace. |

## Completion checklist

- [ ] Confirmed that the AKS cluster has a supported NetworkPolicy engine.
- [ ] Created the dedicated `np-lab` namespace.
- [ ] Deployed the backend Deployment and ClusterIP Service.
- [ ] Deployed frontend and debug test clients with distinct labels.
- [ ] Verified that both clients could reach the backend before policies were applied.
- [ ] Applied the default-deny ingress NetworkPolicy.
- [ ] Applied the frontend-to-backend allow NetworkPolicy.
- [ ] Verified that frontend-to-backend traffic succeeds with HTTP 200.
- [ ] Verified that debug-to-backend traffic is blocked.
- [ ] Deleted the lab namespace.

## Knowledge check

1. What is the difference between `spec.podSelector` and the `podSelector` inside an ingress rule's `from` section?
2. Why does the default-deny policy use `podSelector: {}`?
3. Why can the frontend reach the backend after the allow policy is added, while the debug Pod cannot?
4. Does this lab restrict egress traffic? Explain your answer.
5. Why must an AKS network-policy engine be enabled for the policies to have an effect?

## References

- [Microsoft Learn – Secure traffic between Pods with Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/use-network-policies)
- [Microsoft Learn – Network policy best practices for AKS](https://learn.microsoft.com/en-us/azure/aks/network-policy-best-practices)
- [Kubernetes – Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

