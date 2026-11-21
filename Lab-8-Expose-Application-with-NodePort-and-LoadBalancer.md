# Lab 8 – Expose an Application Using NodePort and LoadBalancer Services on AKS

**Level:** Intermediate  
**Suggested duration:** 20 minutes  
**Environment:** Azure Portal → Cloud Shell → Bash  
**Application:** NGINX web server (two replicas)

## 1. Objective

Deploy an NGINX application on AKS and expose it through two Kubernetes Service types: `NodePort` and `LoadBalancer`. Compare how clients reach the application and inspect the backend endpoints selected by each Service.

## 2. Learning outcomes

By the end of this lab, participants will be able to:

- Explain the difference between `NodePort` and `LoadBalancer` Services.
- Deploy an application and expose it through both Service types.
- Access a NodePort Service from a Pod inside the cluster.
- Access a LoadBalancer Service using its external IP address.
- Inspect Service configuration and EndpointSlices.

## 3. Concepts

| Concept | Explanation |
|---|---|
| **NodePort** | Exposes a Service on a static port on each eligible node. Kubernetes allocates a port automatically when one is not specified; the default range is `30000–32767`. |
| **LoadBalancer** | Requests an external load balancer from the cloud provider. In AKS, a standard `LoadBalancer` Service normally provisions an Azure Load Balancer frontend and an external IP address. |
| **Service selector** | Matches Pods using labels. Traffic is forwarded to ready Pods that match the selector. |
| **EndpointSlice** | Records the network endpoints (usually Pod IP addresses and ports) associated with a Service. It helps show which Pods can receive Service traffic. |

## 4. Prerequisites

- An existing, running AKS cluster.
- Azure Portal access and an active Azure subscription.
- Azure Cloud Shell opened in **Bash** mode.
- `kubectl` configured to connect to the target AKS cluster.
- Permission for the AKS-managed identity to manage the required Azure Load Balancer and public IP resources. Azure Policy or public IP quota restrictions can prevent LoadBalancer provisioning.

> **Important:** Creating a public `LoadBalancer` Service can create billable Azure networking resources. Use a training cluster and remove the lab resources at the end.

## 5. Connect to the AKS cluster

Open **Azure Portal → Cloud Shell → Bash**.

If the Cloud Shell is not already connected to the target cluster, retrieve its credentials:

```bash
RESOURCE_GROUP="<your-resource-group>"
AKS_NAME="<your-aks-cluster-name>"

az account show --output table
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME"
kubectl config current-context
kubectl get nodes
```

- `az account show` displays the active Azure subscription and account context.
- `az aks get-credentials` downloads the AKS access credentials and merges the cluster context into the kubeconfig file used by `kubectl`.
- `kubectl get nodes` confirms that the Kubernetes API is reachable and shows the cluster nodes.

## 6. Create a namespace

```bash
NAMESPACE="nodeport-lb-lab"
kubectl create namespace "$NAMESPACE"
```

A namespace isolates the Deployment, Services and test Pod created for this exercise from other workloads in the cluster.

## 7. Deploy the NGINX application and create both Services

Create a manifest file in Cloud Shell:

```bash
cat > service-types.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-demo
  namespace: nodeport-lb-lab
  labels:
    app: web-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 3
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: nodeport-lb-lab
spec:
  type: NodePort
  selector:
    app: web-demo
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  namespace: nodeport-lb-lab
spec:
  type: LoadBalancer
  selector:
    app: web-demo
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
EOF

kubectl apply -f service-types.yaml
kubectl rollout status deployment/web-demo --namespace "$NAMESPACE" --timeout=180s
kubectl get pods --namespace "$NAMESPACE" -o wide
kubectl get services --namespace "$NAMESPACE" -o wide
```

- The Deployment starts two NGINX replicas and labels them `app: web-demo`.
- Both Services use the same selector, so they route traffic to the same ready NGINX Pods.
- The NodePort is deliberately omitted; Kubernetes assigns an available port automatically.
- The `LoadBalancer` Service asks AKS to provision an external Azure load-balancing endpoint.

**Expected observation:** Two NGINX Pods should reach `Running` and `Ready`. `web-nodeport` should show a port mapping similar to `80:30xxx/TCP`. `web-loadbalancer` may initially show `EXTERNAL-IP` as `<pending>` while Azure provisions the frontend.

## 8. Test the NodePort Service from inside the cluster

In many AKS clusters, worker nodes have private IP addresses and are not directly reachable from the public internet. Therefore, test NodePort from a temporary client Pod inside the cluster.

### 8.1 Create a temporary curl client

```bash
kubectl run curl-client \
  --namespace "$NAMESPACE" \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --command -- sleep 3600

kubectl wait \
  --namespace "$NAMESPACE" \
  --for=condition=Ready pod/curl-client \
  --timeout=120s
```

The temporary Pod provides a `curl` command from within the cluster network. It is used only to test connectivity and will be deleted during cleanup.

### 8.2 Retrieve the NodePort and a node's internal IP

```bash
NODE_PORT=$(kubectl get service web-nodeport \
  --namespace "$NAMESPACE" \
  -o jsonpath='{.spec.ports[0].nodePort}')

NODE_IP=$(kubectl get nodes \
  -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

echo "Node internal IP: $NODE_IP"
echo "Allocated NodePort: $NODE_PORT"
```

- The first command reads the port allocated to the NodePort Service.
- The second command reads the internal IP address of the first AKS node.

### 8.3 Send a request to the NodePort

```bash
kubectl exec --namespace "$NAMESPACE" curl-client -- \
  curl --silent --show-error --include \
  --max-time 10 "http://${NODE_IP}:${NODE_PORT}"
```

The request travels from the client Pod to the node's NodePort, then through the Service to one of the ready NGINX endpoints.

**Expected result:** The response should include `HTTP/1.1 200 OK` and the default NGINX welcome page HTML.

> **Access note:** A NodePort does not automatically assign a public IP to an AKS node. Accessing `http://<node-ip>:<node-port>` from a laptop or the public internet works only when the node address is reachable and the network security rules permit the traffic. Do not open node ports broadly for this exercise; use the in-cluster client above to validate NodePort.

## 9. Test the LoadBalancer Service

### 9.1 Wait for the external IP address

```bash
LB_IP=""

for attempt in {1..30}; do
  LB_IP=$(kubectl get service web-loadbalancer \
    --namespace "$NAMESPACE" \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

  if [[ -n "$LB_IP" ]]; then
    break
  fi

  echo "Waiting for the Azure Load Balancer external IP... ($attempt/30)"
  sleep 10
done

if [[ -z "$LB_IP" ]]; then
  echo "External IP is still pending. Check the Service events and Azure permissions."
else
  echo "LoadBalancer external IP: $LB_IP"
fi
```

The loop checks for up to five minutes while AKS and Azure provision the external frontend. Provisioning can take longer in some environments.

### 9.2 Inspect and access the LoadBalancer Service

```bash
kubectl get service web-loadbalancer --namespace "$NAMESPACE" -o wide
kubectl describe service web-loadbalancer --namespace "$NAMESPACE"
```

If `LB_IP` contains an address, send an HTTP request from Cloud Shell:

```bash
if [[ -n "$LB_IP" ]]; then
  curl --include --max-time 20 "http://${LB_IP}/"
fi
```

- `kubectl describe service` shows the Service configuration, allocated ports, events and endpoint information.
- `curl` sends a request to the external IP created for the LoadBalancer Service.

**Expected result:** The HTTP response should show `200 OK` and the NGINX welcome page. If the external IP remains pending, use the troubleshooting section before proceeding.

## 10. Inspect the Service endpoints

Both Services select the same NGINX Pods. Inspect their EndpointSlices:

```bash
kubectl get endpointslices \
  --namespace "$NAMESPACE" \
  --selector kubernetes.io/service-name=web-nodeport \
  -o wide

kubectl get endpointslices \
  --namespace "$NAMESPACE" \
  --selector kubernetes.io/service-name=web-loadbalancer \
  -o wide
```

For a detailed view of the selected Pod IP addresses and ports:

```bash
kubectl get endpointslices \
  --namespace "$NAMESPACE" \
  --selector kubernetes.io/service-name=web-nodeport \
  -o yaml
```

**Expected observation:** Each Service should have ready endpoints corresponding to the NGINX Pods. If no endpoints appear, check that the Deployment Pods are ready and that the Service selector matches their labels.

## 11. Compare NodePort and LoadBalancer

| Feature | NodePort | LoadBalancer |
|---|---|---|
| Exposure | Opens an allocated port on the cluster nodes | Provisions an external load-balancing endpoint through the cloud provider |
| Address used by client | Node IP + allocated NodePort | LoadBalancer external IP + Service port |
| Typical AKS use | Testing or access from a reachable private network / custom load-balancing setup | Direct external access to a Service when required |
| Public IP created automatically | No | Normally yes, for the default public AKS LoadBalancer Service |
| Port | Automatically allocated from the NodePort range unless specified | Uses the Service port; the implementation also uses node ports by default |
| Backend selection | Service selector and ready endpoints | Service selector and ready endpoints |

**Remember:** `NodePort` exposes a port on nodes; it does not create node public IPs. `LoadBalancer` asks AKS to configure Azure load-balancing resources and an external frontend. For production HTTP/HTTPS applications, consider an Ingress or Gateway solution when centralized routing and TLS termination are required.

## 12. Troubleshooting

| Issue | Checks and corrective action |
|---|---|
| NGINX Pods are not Ready | Run `kubectl get pods -n "$NAMESPACE"` and `kubectl describe pod <pod-name> -n "$NAMESPACE"`. Check image-pull events and readiness probe status. |
| NodePort request times out | Confirm the client Pod is Ready, the NodePort value is populated, and the node internal IP was retrieved. Check NetworkPolicies and cluster networking if applicable. |
| `EXTERNAL-IP` remains `<pending>` | Run `kubectl describe service web-loadbalancer -n "$NAMESPACE"` and inspect Events. Check Azure permissions for the AKS-managed identity, public IP quota, Azure Policy and provisioning errors. |
| LoadBalancer IP exists but HTTP request fails | Verify the Service has ready EndpointSlices, the Service port is 80, and the NGINX Pods are Ready. Check any configured network restrictions. |
| EndpointSlice has no addresses | Compare Service selector with Pod labels using `kubectl get pods -n "$NAMESPACE" --show-labels`. Correct the selector or Pod labels if they do not match. |

## 13. Cleanup

Delete the lab namespace and all resources created inside it, including the Deployment, Services and temporary client Pod:

```bash
kubectl delete namespace "$NAMESPACE"
rm -f service-types.yaml
```

Deleting the LoadBalancer Service releases its Service-managed frontend configuration and associated dynamically allocated IP resources when managed by AKS. The AKS cluster and its node pools are not deleted.

## 14. Completion checklist

- [ ] Connected to the AKS cluster from Azure Cloud Shell using Bash.
- [ ] Deployed two NGINX replicas and verified they are Ready.
- [ ] Created a NodePort Service and identified its allocated NodePort.
- [ ] Accessed NGINX through NodePort from a Pod inside the cluster.
- [ ] Created a LoadBalancer Service and retrieved its external IP.
- [ ] Accessed NGINX through the LoadBalancer external IP.
- [ ] Inspected EndpointSlices for both Services.
- [ ] Deleted the lab namespace and temporary resources.

## 15. Knowledge check

1. Does a NodePort Service create a public IP address for an AKS node?
2. What is the default Kubernetes NodePort range?
3. Which Azure resource is requested when a Service is configured with `type: LoadBalancer` in AKS?
4. Why can two different Services route traffic to the same set of Pods?
5. What does an empty EndpointSlice indicate when the Service selector is expected to match application Pods?

## References

- [Microsoft Learn – Kubernetes Services in AKS](https://learn.microsoft.com/azure/aks/concepts-network-services)
- [Kubernetes – Service types](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes – EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes – Ports and Protocols (including the default NodePort range)](https://kubernetes.io/docs/reference/networking/ports-and-protocols/)
