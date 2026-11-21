# Lab 4 – Deploy NGINX with ConfigMap Environment Variables on AKS


## Lab objective

Deploy an NGINX application on AKS, create a ConfigMap containing application settings, and inject the ConfigMap values into the NGINX container as environment variables.

## Concepts

- **ConfigMap:** A Kubernetes object that stores non-confidential configuration as key-value pairs, separately from the container image.
- **Environment variable injection:** Kubernetes can populate container environment variables from ConfigMap keys when a Pod starts.
- **Deployment:** A Kubernetes controller that maintains the desired number of application Pod replicas.

> **Security note:** ConfigMaps are for non-sensitive values only. Do not store passwords, tokens, or other secrets in a ConfigMap. Use Kubernetes Secrets or an appropriate secret-management integration for sensitive data.

## Prerequisites

- An existing AKS cluster is available and running.
- You can access the Azure Portal and open **Cloud Shell** in **Bash** mode.
- Your Azure identity can retrieve AKS user credentials and create Deployments and ConfigMaps in the target namespace.

## Lab steps

### Step 1 – Open Cloud Shell and connect to AKS

In the Azure Portal, open **Cloud Shell**, select **Bash**, and run the commands below. Replace the sample values with the resource group and AKS cluster names provided by your trainer.

```bash
RESOURCE_GROUP="<your-resource-group>"
AKS_NAME="<your-aks-cluster-name>"

az account show --output table
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_NAME"
kubectl config current-context
kubectl get nodes
```

- `az account show` displays the currently selected Azure subscription and signed-in identity.
- `az aks get-credentials` retrieves cluster credentials and merges the context into the Cloud Shell kubeconfig file.
- `kubectl config current-context` displays the active context; `kubectl get nodes` verifies cluster connectivity.

If the required subscription is not selected, set it before retrieving credentials:

```bash
az account set --subscription "<subscription-id-or-name>"
```

### Step 2 – Create a ConfigMap

Create a ConfigMap with three non-sensitive application settings:

```bash
kubectl create configmap nginx-app-config \
  --from-literal=APP_ENV=training \
  --from-literal=LOG_LEVEL=info \
  --from-literal=APP_MESSAGE="Welcome to the AKS ConfigMap lab" \
  --namespace default
```

- `kubectl create configmap` creates a ConfigMap resource in Kubernetes.
- Each `--from-literal` option adds one key-value pair to the ConfigMap.
- The ConfigMap and the Pod that uses it must be in the same namespace.

Verify the ConfigMap:

```bash
kubectl get configmap nginx-app-config --namespace default
kubectl describe configmap nginx-app-config --namespace default
```

`kubectl describe` displays the ConfigMap metadata and its keys and values.

### Step 3 – Create an NGINX Deployment that uses the ConfigMap

Create a Deployment manifest in Cloud Shell:

```bash
cat <<'EOF' > nginx-configmap-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-configmap-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-configmap-demo
  template:
    metadata:
      labels:
        app: nginx-configmap-demo
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: nginx-app-config
EOF
```

**Key configuration:**

- `replicas: 1` requests one NGINX Pod.
- `envFrom.configMapRef` imports all valid key-value pairs from `nginx-app-config` as environment variables in the container.
- The ConfigMap must exist before the Pod starts; it was created in Step 2.

Apply the Deployment:

```bash
kubectl apply -f nginx-configmap-deployment.yaml
```

`kubectl apply -f` creates or updates the Kubernetes resources described in the YAML file.

### Step 4 – Verify the Deployment and Pod

```bash
kubectl rollout status deployment/nginx-configmap-demo --namespace default
kubectl get deployment nginx-configmap-demo --namespace default
kubectl get pods --namespace default -l app=nginx-configmap-demo
```

- `kubectl rollout status` waits for the Deployment to become available.
- `kubectl get pods` lists the Pod created by the Deployment. Confirm that its status is `Running` and it shows `1/1` ready.

### Step 5 – Verify ConfigMap values inside the NGINX container

Run:

```bash
kubectl exec deployment/nginx-configmap-demo -- printenv | grep -E '^(APP_ENV|LOG_LEVEL|APP_MESSAGE)='
```

Expected output (the order may vary):

```text
APP_ENV=training
LOG_LEVEL=info
APP_MESSAGE=Welcome to the AKS ConfigMap lab
```

- `kubectl exec` runs a command inside a container in the selected Pod.
- `printenv` displays the container's environment variables; `grep` filters the three values supplied by the ConfigMap.

If the command does not return the expected values, inspect the Pod and recent namespace events:

```bash
kubectl describe pod -l app=nginx-configmap-demo --namespace default
kubectl get events --namespace default --sort-by=.metadata.creationTimestamp
```

## Validation checklist

- [ ] Connected to the intended AKS cluster from Cloud Shell.
- [ ] Created the `nginx-app-config` ConfigMap.
- [ ] Deployed the `nginx-configmap-demo` Deployment.
- [ ] Confirmed the NGINX Pod is `Running` and ready.
- [ ] Verified `APP_ENV`, `LOG_LEVEL` and `APP_MESSAGE` inside the container.

## Important observations

- ConfigMaps separate environment-specific settings from the container image, allowing the same image to be used in different environments.
- Environment variables sourced from a ConfigMap are read when the container starts. Updating the ConfigMap does **not** automatically update environment variables in an already-running container; restart the affected Pods to load changed values.
- This lab does not expose NGINX outside the cluster. It focuses on creating the workload and verifying environment variable injection.
- The custom variables demonstrate configuration injection. The standard NGINX image does not automatically change its behaviour based on these custom variables.

## Cleanup

Delete the Deployment and ConfigMap after completing the lab:

```bash
kubectl delete deployment nginx-configmap-demo --namespace default
kubectl delete configmap nginx-app-config --namespace default
rm -f nginx-configmap-deployment.yaml
```

## Knowledge check

1. What problem does a ConfigMap solve in a Kubernetes application?
2. What does `envFrom.configMapRef` do in a container specification?
3. Why should passwords and other sensitive values not be stored in a ConfigMap?
4. Does changing a ConfigMap automatically change environment variables in an already-running container?

## Expected outcome

Participants can create a ConfigMap, reference it from an NGINX Deployment, and verify that Kubernetes injects the ConfigMap values into the running container as environment variables.

## References

- [Kubernetes – ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes – Configure a Pod to Use a ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
