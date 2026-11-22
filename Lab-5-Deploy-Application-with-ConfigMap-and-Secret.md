# Lab 5 – Deploy an Application Using ConfigMap and Secret on AKS

## Lab objective
Deploy a sample application that reads non-sensitive settings from a ConfigMap mounted as a file and sensitive values from a Kubernetes Secret injected as environment variables.

## Learning outcomes
- Explain the difference between a ConfigMap and a Secret.
- Create a ConfigMap and mount its contents as files inside a container.
- Create a Secret and inject selected Secret keys as environment variables.
- Verify that the application can read both configuration sources without displaying secret values.

## Key concepts
| Concept | Explanation |
|---|---|
| ConfigMap | Stores non-confidential configuration separately from the container image, such as application name, environment and log level. |
| Secret | Stores sensitive values such as usernames, passwords, tokens or keys. Base64 encoding is not encryption. |
| Volume mount | Makes ConfigMap data available to a container as files at a specified mount path. |
| `secretKeyRef` | Maps one key from a Kubernetes Secret to an environment variable in a container. |
| Namespace | Provides a logical boundary for Kubernetes resources. The Deployment, ConfigMap and Secret must use the same namespace in this lab. |

## Prerequisites
- Access to an existing AKS cluster.
- Azure Portal Cloud Shell opened in **Bash** mode, with `kubectl` configured for the cluster.
- Permission to create namespaces, ConfigMaps, Secrets and Deployments.

> **Important:** Use dummy training credentials only. Do not use production passwords, API keys or other real credentials in this lab.

---

## Step 1 – Verify AKS connectivity and create a lab namespace

```bash
kubectl cluster-info
kubectl get nodes
```

These commands confirm that `kubectl` can reach the cluster and that nodes are available.

```bash
NAMESPACE=aks-config-secret-lab
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
```

`NAMESPACE` stores the namespace name for reuse. The command creates the namespace if it does not exist and safely applies the same definition if it already exists.

## Step 2 – Create a ConfigMap

The ConfigMap stores non-sensitive settings in a JSON file. The application will read this file from a mounted directory.

Create a file: vi app-configmap.yaml

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-settings
  namespace: aks-config-secret-lab
data:
  app-settings.json: |
    {
      "appName": "orders-demo",
      "environment": "training",
      "logLevel": "info"
    }

```

```
kubectl apply -f app-configmap.yaml
kubectl describe configmap app-settings -n "$NAMESPACE"
```

`kubectl apply` creates the ConfigMap. `kubectl describe` displays its metadata and keys.

## Step 3 – Create a Secret

Enter a dummy username and password. The password is hidden while typing and is not written literally in the `kubectl` command.

```bash
read -r -p "Enter demo database username: " DB_USERNAME
read -r -s -p "Enter demo database password: " DB_PASSWORD
printf '\n'

kubectl create secret generic app-credentials \
  --namespace "$NAMESPACE" \
  --from-literal=DB_USERNAME="$DB_USERNAME" \
  --from-literal=DB_PASSWORD="$DB_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -

unset DB_USERNAME DB_PASSWORD
kubectl describe secret app-credentials -n "$NAMESPACE"
```

`read -s` hides the password as it is entered. `kubectl create secret generic` creates an Opaque Secret with two keys, and `unset` removes the temporary shell variables. `kubectl describe secret` shows the key names and sizes without displaying the values.

> **Security note:** Kubernetes Secret values are Base64-encoded in their API representation; Base64 is not encryption. Production clusters should use appropriate RBAC restrictions and encryption at rest. Consider integrating AKS with Azure Key Vault through the Secrets Store CSI Driver for production secrets.

## Step 4 – Create the application Deployment

The sample Python application reads `app-settings.json` from the mounted ConfigMap volume and reads `DB_USERNAME` and `DB_PASSWORD` from environment variables. Its HTTP response reports only whether the credentials are configured; it never returns their values.

```bash
cat > app-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-secret-demo
  namespace: aks-config-secret-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-secret-demo
  template:
    metadata:
      labels:
        app: config-secret-demo
    spec:
      containers:
        - name: demo-app
          image: python:3.12-alpine
          imagePullPolicy: IfNotPresent
          command: ["python", "-u", "-c"]
          args:
            - |
              import json
              import os
              from http.server import BaseHTTPRequestHandler, HTTPServer
              from pathlib import Path

              settings = json.loads(Path("/etc/app-config/app-settings.json").read_text())
              username_configured = bool(os.getenv("DB_USERNAME"))
              password_configured = bool(os.getenv("DB_PASSWORD"))

              print("Application settings loaded:", settings)
              print("DB_USERNAME configured:", username_configured)
              print("DB_PASSWORD configured:", password_configured)

              class Handler(BaseHTTPRequestHandler):
                  def do_GET(self):
                      response = {
                          "appName": settings["appName"],
                          "environment": settings["environment"],
                          "logLevel": settings["logLevel"],
                          "databaseUsernameConfigured": username_configured,
                          "databasePasswordConfigured": password_configured
                      }
                      payload = (json.dumps(response, indent=2) + "\n").encode("utf-8")
                      self.send_response(200)
                      self.send_header("Content-Type", "application/json")
                      self.send_header("Content-Length", str(len(payload)))
                      self.end_headers()
                      self.wfile.write(payload)

              HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-credentials
                  key: DB_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-credentials
                  key: DB_PASSWORD
          volumeMounts:
            - name: app-config-volume
              mountPath: /etc/app-config
              readOnly: true
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 2
            periodSeconds: 5
      volumes:
        - name: app-config-volume
          configMap:
            name: app-settings
EOF
```

Apply the Deployment and wait for the Pod to become ready:

```bash
kubectl apply -f app-deployment.yaml
kubectl rollout status deployment/config-secret-demo -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
```

`volumeMounts` makes the ConfigMap file available at `/etc/app-config`. Each `secretKeyRef` maps one Secret key to an environment variable inside the container.

## Step 5 – Verify the mounted ConfigMap file

```bash
kubectl exec deployment/config-secret-demo -n "$NAMESPACE" -- \
  cat /etc/app-config/app-settings.json
```

Expected output:

```json
{
  "appName": "orders-demo",
  "environment": "training",
  "logLevel": "info"
}
```

This confirms that the ConfigMap key `app-settings.json` is mounted as a file inside the application container.

## Step 6 – Verify Secret environment variables without revealing values

```bash
kubectl exec deployment/config-secret-demo -n "$NAMESPACE" -- \
  sh -c 'if [ -n "$DB_USERNAME" ] && [ -n "$DB_PASSWORD" ]; then echo "Both Secret environment variables are configured"; else echo "A required Secret environment variable is missing"; exit 1; fi'
```

The command checks that both environment variables are non-empty. It deliberately does not print their values.

Inspect the application startup messages:

```bash
kubectl logs deployment/config-secret-demo -n "$NAMESPACE"
```

Expected log pattern:

```text
Application settings loaded: {'appName': 'orders-demo', 'environment': 'training', 'logLevel': 'info'}
DB_USERNAME configured: True
DB_PASSWORD configured: True
```

## Step 7 – Test the application response

Use `kubectl port-forward` to access the application locally through Cloud Shell. The commands start port forwarding temporarily, call the application endpoint and then stop the port-forward process.

```bash
kubectl port-forward deployment/config-secret-demo 8080:8080 -n "$NAMESPACE" \
  >/tmp/config-secret-port-forward.log 2>&1 &
PF_PID=$!
sleep 3
curl --fail --silent --show-error http://127.0.0.1:8080/
CURL_STATUS=$?
kill "$PF_PID" 2>/dev/null || true
wait "$PF_PID" 2>/dev/null || true
test "$CURL_STATUS" -eq 0
```

Expected response:

```json
{
  "appName": "orders-demo",
  "environment": "training",
  "logLevel": "info",
  "databaseUsernameConfigured": true,
  "databasePasswordConfigured": true
}
```

The response confirms that the application read the mounted configuration file and detected both Secret-backed environment variables. Secret values are not included in the response.

## Step 8 – Clean up lab resources

```bash
kubectl delete namespace "$NAMESPACE"
rm -f app-configmap.yaml app-deployment.yaml
```

Deleting the namespace removes the lab Deployment, ConfigMap, Secret and other namespaced resources. The second command removes the local YAML files from Cloud Shell.

## Completion checklist

- [ ] Connected to the AKS cluster through Cloud Shell.
- [ ] Created the `aks-config-secret-lab` namespace.
- [ ] Created the `app-settings` ConfigMap.
- [ ] Created the `app-credentials` Secret without displaying the password.
- [ ] Deployed the sample application.
- [ ] Verified the ConfigMap file inside the container.
- [ ] Verified Secret-backed environment variables without printing their values.
- [ ] Tested the application response.
- [ ] Deleted the lab namespace and local manifest files.

## Key takeaways

- Use **ConfigMaps** for non-sensitive application configuration and **Secrets** for credentials or other sensitive values.
- A ConfigMap volume exposes its keys as files inside the container; `secretKeyRef` injects selected Secret keys as environment variables.
- ConfigMaps and Secrets must be in the same namespace as the Pod that consumes them.
- Environment variables sourced from a Secret are set when the container starts. Restart the Pods after changing the Secret to make the new values available to the application.
- Secret objects require appropriate access controls and encryption-at-rest protections; Base64 encoding alone does not secure sensitive data.
- This demo application reads its configuration file during startup. Restart it after changing the ConfigMap so the application reloads the settings.

## Official references

- [Kubernetes – ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes – Distribute Credentials Securely Using Secrets](https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Microsoft Learn – Storage options in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
