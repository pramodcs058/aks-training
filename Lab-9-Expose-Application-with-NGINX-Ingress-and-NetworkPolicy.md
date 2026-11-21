# Lab 9 – Expose an Application Through NGINX Ingress and Apply a NetworkPolicy on AKS

## Lab objective

Deploy a frontend and backend application on AKS, expose the frontend through the managed NGINX Ingress controller, and use a Kubernetes `NetworkPolicy` to restrict access to the backend.

By completing this lab, participants will be able to:

- Explain how an Ingress resource routes HTTP traffic to a Kubernetes Service.
- Configure an Ingress resource to use the AKS Application Routing add-on's NGINX Ingress controller.
- Understand how NetworkPolicy selects Pods and controls incoming Pod-to-Pod traffic.
- Verify that frontend-to-backend traffic is allowed while traffic from an unrelated Pod is blocked.

## Environment

- Azure Portal **Cloud Shell – Bash**
- An existing AKS cluster and permission to deploy workloads
- Azure CLI and `kubectl` (both are preinstalled in Cloud Shell)
- An AKS Application Routing add-on with its managed NGINX Ingress controller
- A supported NetworkPolicy engine enabled on the AKS cluster

> **Important – cluster prerequisites:** A Kubernetes `NetworkPolicy` resource does not enforce traffic restrictions by itself. The AKS cluster must use a supported policy engine, such as Azure CNI powered by Cilium, Calico, or Azure Network Policy Manager (NPM). Cilium is Microsoft's recommended choice for Linux node pools. Confirm the cluster configuration in Step 2 before proceeding with the policy test. Do not assume that a cluster enforces policies just because `kubectl apply` succeeds.
>
> **NGINX lifecycle notice (21 November 2026):** This lab follows the NGINX Ingress requirement in the training TOC and uses the AKS Application Routing add-on. The upstream Ingress NGINX project maintenance ended in March 2026, and Microsoft's published critical-security-patch support for the managed NGINX Ingress add-on ends in November 2026. Treat this as a TOC-aligned learning exercise, not a recommendation for a new long-term production deployment. For new designs, evaluate the AKS Application Routing Gateway API implementation or Application Gateway for Containers.

---

## Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Open **Cloud Shell** and select **Bash**.
3. Set the AKS resource group and cluster name. Replace the sample values with your cluster details.

```bash
export RESOURCE_GROUP="my-aks-resource-group"
export CLUSTER_NAME="my-aks-cluster"
export NAMESPACE="aks-ingress-networkpolicy-lab"
```

4. Confirm the active subscription and sign in context.

```bash
az account show --output table
```

5. Retrieve AKS credentials and configure `kubectl`.

```bash
az aks get-credentials \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --overwrite-existing
```

6. Confirm cluster access.

```bash
kubectl get nodes
```

**Explanation:** `az aks get-credentials` merges the cluster credentials into the Cloud Shell kubeconfig. `kubectl get nodes` confirms that the current context can communicate with the Kubernetes API server.

## Step 2 – Check Application Routing and NetworkPolicy support

1. Inspect the AKS networking configuration.

```bash
az aks show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --query "networkProfile.{networkPlugin:networkPlugin,networkPolicy:networkPolicy,networkDataplane:networkDataplane}" \
  --output yaml
```

2. Confirm that one of the following is configured:
   - `networkDataplane: cilium`, or
   - `networkPolicy: calico`, or
   - `networkPolicy: azure` (Azure NPM).

   If none is shown, stop and ask the trainer to provide a cluster with a supported NetworkPolicy engine. Do not continue to the policy enforcement test on a cluster without an enforcement engine.

3. Check whether the AKS Application Routing add-on is enabled.

```bash
az aks show \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --query "ingressProfile.webAppRouting.enabled" \
  --output tsv
```

4. If the command returns `false`, enable Application Routing with an external default NGINX controller.

```bash
az aks approuting enable \
  --resource-group "$RESOURCE_GROUP" \
  --name "$CLUSTER_NAME" \
  --nginx External
```

5. Verify the managed Ingress class and controller Service.

```bash
kubectl get ingressclass
kubectl get pods -n app-routing-system
kubectl get service -n app-routing-system
```

Confirm that the Ingress class `webapprouting.kubernetes.azure.com` is available and the NGINX controller is running. If Application Routing is already enabled but this Ingress class is missing, ask the trainer to verify the add-on's NGINX controller configuration before proceeding.

**Explanation:** The Application Routing add-on manages the NGINX Ingress controller. The `ingressClassName` in an Ingress resource tells Kubernetes which controller should process that resource.

## Step 3 – Create a lab namespace

```bash
kubectl create namespace "$NAMESPACE"
```

Confirm the namespace:

```bash
kubectl get namespace "$NAMESPACE"
```

**Explanation:** A namespace keeps the lab Deployments, Services, Ingress and NetworkPolicy separate from other workloads in the cluster.

## Step 4 – Deploy the backend application

The backend uses a lightweight HTTP echo container that returns a fixed response. It listens on TCP port `5678`.

```bash
kubectl apply -n "$NAMESPACE" -f - <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:5678"
            - "-text=Hello from AKS backend"
          ports:
            - name: http
              containerPort: 5678
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 5678
      targetPort: 5678
      protocol: TCP
YAML
```

Wait for the backend Pod to become ready and check the Service:

```bash
kubectl rollout status deployment/backend -n "$NAMESPACE" --timeout=120s
kubectl get pods,service -n "$NAMESPACE" -l app=backend
```

**Explanation:** The Deployment manages the backend Pod. The ClusterIP Service named `backend` provides a stable DNS name (`backend`) and virtual IP for other Pods in the same namespace.

## Step 5 – Create the frontend NGINX configuration

The frontend serves a simple home page and forwards requests under `/api/` to the backend Service.

```bash
kubectl create configmap frontend-nginx-config \
  --namespace "$NAMESPACE" \
  --from-literal=default.conf='server {
    listen 8080;
    server_name _;

    location = /health {
      default_type text/plain;
      return 200 "frontend-ok\n";
    }

    location /api/ {
      proxy_pass http://backend:5678/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
      default_type text/plain;
      return 200 "Hello from AKS frontend\n";
    }
  }'
```

Verify the ConfigMap:

```bash
kubectl get configmap frontend-nginx-config -n "$NAMESPACE"
```

**Explanation:** The ConfigMap stores NGINX configuration separately from the container image. The frontend uses the Kubernetes Service DNS name `backend` to reach the backend Pod.

## Step 6 – Deploy the frontend and its Service

```bash
kubectl apply -n "$NAMESPACE" -f - <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
              readOnly: true
      volumes:
        - name: nginx-config
          configMap:
            name: frontend-nginx-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
YAML
```

Wait for the frontend rollout and inspect the Pods and Services:

```bash
kubectl rollout status deployment/frontend -n "$NAMESPACE" --timeout=120s
kubectl get pods,services -n "$NAMESPACE" -o wide
```

Confirm that two frontend Pods and one backend Pod are running.

## Step 7 – Verify frontend-to-backend communication before applying the policy

Run a request from a frontend Pod to the backend Service:

```bash
kubectl exec -n "$NAMESPACE" deployment/frontend -- \
  wget -qO- http://backend:5678
```

Expected response:

```text
Hello from AKS backend
```

**Explanation:** This confirms that the frontend can resolve the backend Service name through cluster DNS and reach the backend over the Pod network.

## Step 8 – Create an Ingress resource

Create an Ingress resource that routes HTTP requests to the frontend Service.

```bash
kubectl apply -n "$NAMESPACE" -f - <<'YAML'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
YAML
```

Check the Ingress status:

```bash
kubectl get ingress -n "$NAMESPACE" -o wide
```

Retrieve the external IP of the managed NGINX controller Service:

```bash
kubectl get service nginx -n app-routing-system -o wide
```

Wait until the Service has an external IP address. Then set it in a variable:

```bash
export INGRESS_IP=$(kubectl get service nginx -n app-routing-system \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$INGRESS_IP"
```

If `INGRESS_IP` is empty, the Azure Load Balancer address is not ready yet. Recheck the Service and its events before continuing.

Test the frontend through the Ingress endpoint:

```bash
curl -i "http://${INGRESS_IP}/"
```

Expected response body:

```text
Hello from AKS frontend
```

Test the frontend's API proxy route:

```bash
curl -i "http://${INGRESS_IP}/api/"
```

Expected response body:

```text
Hello from AKS backend
```

**Explanation:** The Ingress controller receives the HTTP request and routes it to the frontend Service. NGINX in the frontend then forwards `/api/` requests to the backend Service.

## Step 9 – Create a test client Pod

Create a separate Pod with the label `app=debug-client`. It will be used to test whether an unrelated Pod can reach the backend.

```bash
kubectl run debug-client \
  --namespace "$NAMESPACE" \
  --image=curlimages/curl:8.10.1 \
  --labels=app=debug-client \
  --restart=Never \
  --command -- sleep 3600
```

Wait until the Pod is ready:

```bash
kubectl wait \
  --namespace "$NAMESPACE" \
  --for=condition=Ready pod/debug-client \
  --timeout=90s
```

Before applying the NetworkPolicy, test access to the backend:

```bash
kubectl exec -n "$NAMESPACE" debug-client -- \
  curl -sS --max-time 5 http://backend:5678
```

Expected response before the policy is applied:

```text
Hello from AKS backend
```

**Explanation:** By default, Kubernetes allows Pod-to-Pod communication unless a NetworkPolicy selects and isolates the destination or source Pods.

## Step 10 – Apply a NetworkPolicy to the backend

The policy below selects Pods labelled `app=backend` and allows incoming TCP traffic on port `5678` only from Pods labelled `app=frontend` in the same namespace.

```bash
kubectl apply -n "$NAMESPACE" -f - <<'YAML'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
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
          port: 5678
YAML
```

Verify the policy:

```bash
kubectl get networkpolicy -n "$NAMESPACE"
kubectl describe networkpolicy allow-backend-from-frontend -n "$NAMESPACE"
```

**Explanation:** The NetworkPolicy selects backend Pods and isolates them for ingress traffic. Only Pods matching `app=frontend` are allowed to connect to the backend on TCP port `5678`. The policy does not restrict other traffic directions or other Pods.

## Step 11 – Test allowed and blocked communication

### Test A – Frontend to backend (allowed)

```bash
kubectl exec -n "$NAMESPACE" deployment/frontend -- \
  wget -qO- http://backend:5678
```

Expected response:

```text
Hello from AKS backend
```

### Test B – Debug client to backend (blocked)

```bash
if kubectl exec -n "$NAMESPACE" debug-client -- \
  curl -sS --connect-timeout 3 --max-time 5 http://backend:5678; then
  echo "UNEXPECTED: debug-client can still reach the backend. Check NetworkPolicy engine and labels."
else
  echo "EXPECTED: debug-client is blocked from reaching the backend."
fi
```

The request should time out or fail to connect. The frontend request should continue to succeed because its Pods match the NetworkPolicy's allowed selector.

### Test C – Ingress route to frontend and backend (allowed path)

```bash
curl -i "http://${INGRESS_IP}/"
curl -i "http://${INGRESS_IP}/api/"
```

Both requests should succeed. The second request travels through the frontend Pod, which is allowed to connect to the backend.

> **Validation:** If the debug client can still reach the backend, first confirm that a supported NetworkPolicy engine is enabled on the AKS cluster. Then check the policy, Pod labels, namespace and backend port. Applying a NetworkPolicy object without an enforcing engine does not block traffic.

## Step 12 – Inspect the resources

```bash
kubectl get deployments,pods,services,ingress,networkpolicy -n "$NAMESPACE" -o wide
kubectl get endpointslices -n "$NAMESPACE"
```

Review the following:

- The Ingress uses class `webapprouting.kubernetes.azure.com`.
- The frontend Service selects frontend Pods.
- The backend Service selects backend Pods.
- The backend NetworkPolicy selects only `app=backend` Pods.
- The NetworkPolicy allows ingress from `app=frontend` Pods on TCP port `5678`.

## Step 13 – Clean up lab resources

Delete the lab namespace and all resources created inside it:

```bash
kubectl delete namespace "$NAMESPACE"
```

Confirm that the namespace has been removed:

```bash
kubectl get namespace "$NAMESPACE"
```

The AKS Application Routing add-on is cluster-scoped and may be shared by other workloads. Do not disable the add-on as part of this lab cleanup unless the trainer specifically instructs you to do so.

---

## Troubleshooting

| Issue | What to check |
|---|---|
| Ingress has no external address | Check `kubectl get service nginx -n app-routing-system` and wait for the Azure Load Balancer provisioning to complete. |
| Ingress class not found | Confirm that the Application Routing add-on and its managed NGINX controller are enabled. Run `kubectl get ingressclass`. |
| Frontend or backend Pod is not ready | Run `kubectl describe pod` and `kubectl logs` for the affected Pod. Check image pull errors and container startup messages. |
| Frontend cannot resolve `backend` | Confirm the backend Service exists in the same namespace and its selector matches the backend Pod labels. |
| `/api/` returns an error | Verify the backend Pod and Service, then check the NGINX ConfigMap and frontend Pod logs. |
| Debug client can still access backend after policy | Confirm that AKS has Cilium, Calico or Azure NPM enabled; then verify the policy selectors, Pod labels, namespace and TCP port. |
| Frontend cannot access backend after policy | Confirm frontend Pods have label `app=frontend`, backend Pods have label `app=backend`, and the policy allows TCP port `5678`. |

Useful commands:

```bash
kubectl describe pod <pod-name> -n "$NAMESPACE"
kubectl logs <pod-name> -n "$NAMESPACE"
kubectl describe service backend -n "$NAMESPACE"
kubectl describe ingress frontend-ingress -n "$NAMESPACE"
kubectl describe networkpolicy allow-backend-from-frontend -n "$NAMESPACE"
kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp
```

## Lab completion checklist

- [ ] Connected to the AKS cluster from Azure Portal Cloud Shell (Bash).
- [ ] Confirmed that a supported NetworkPolicy engine is enabled.
- [ ] Verified the managed NGINX Ingress controller and Ingress class.
- [ ] Deployed frontend and backend workloads with ClusterIP Services.
- [ ] Created an Ingress resource and accessed the frontend through its external IP.
- [ ] Verified that the frontend can communicate with the backend.
- [ ] Applied a NetworkPolicy to the backend Pods.
- [ ] Verified that frontend-to-backend traffic is allowed.
- [ ] Verified that debug-client-to-backend traffic is blocked.
- [ ] Deleted the lab namespace.

## Key concepts

| Concept | Explanation |
|---|---|
| Ingress | A Kubernetes API resource that defines HTTP or HTTPS routing rules to Services. It requires an Ingress controller to implement those rules. |
| Ingress controller | A component that watches Ingress resources and configures a data plane, such as NGINX, to route incoming traffic. |
| IngressClass | Identifies which Ingress controller should process an Ingress resource. |
| ClusterIP Service | Exposes a workload through a stable virtual IP and DNS name inside the cluster. |
| NetworkPolicy | Defines allowed network traffic for selected Pods. Enforcement requires a compatible network-policy engine. |
| Pod selector | Matches Pods using labels such as `app=frontend` or `app=backend`. |
| Default behaviour | Without a policy selecting a Pod for ingress isolation, inbound traffic is allowed by default. NetworkPolicy rules are additive. |

## Knowledge check

1. What is the difference between a Kubernetes Service and an Ingress resource?
2. Why does an Ingress resource require an Ingress controller?
3. What is the purpose of `ingressClassName`?
4. Why does the backend Service use `ClusterIP` in this lab?
5. Which Pods are selected by the NetworkPolicy?
6. Why can the frontend reach the backend while the debug client cannot?
7. Why must an AKS NetworkPolicy engine be enabled before applying a policy?
8. Does this NetworkPolicy restrict outbound traffic from the backend Pods?

## Official references

- [AKS managed NGINX Ingress with the Application Routing add-on](https://learn.microsoft.com/en-us/azure/aks/app-routing)
- [AKS Ingress networking concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-network-ingress)
- [Secure Pod traffic with Network Policies in AKS](https://learn.microsoft.com/en-us/azure/aks/use-network-policies)
- [Kubernetes NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [AKS Application Routing Gateway API migration guidance](https://learn.microsoft.com/en-us/azure/aks/app-routing-migration)
