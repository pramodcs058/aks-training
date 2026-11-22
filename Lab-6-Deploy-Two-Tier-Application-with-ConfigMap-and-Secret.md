# Lab 6 – Deploy a Two-Tier Application on AKS Using ConfigMap and Secret

## Lab objective
Deploy a two-tier application on AKS consisting of an NGINX frontend and a Python backend API. Use ConfigMaps for non-sensitive application configuration and a Kubernetes Secret for sensitive backend credentials.

## Learning outcomes
- Deploy frontend and backend components as separate Kubernetes Deployments.
- Use Kubernetes Services for internal communication between application tiers.
- Externalize application settings using ConfigMaps.
- Inject sensitive configuration into the backend using Secret environment variables.
- Test the frontend-to-backend request flow without exposing Secret values.

## Application architecture

```text
                       Browser / curl
                             |
                      Cloud Shell port-forward
                             |
                      frontend Service
                         (port 80)
                             |
                      NGINX frontend Pod
                             |
                /api/ reverse-proxy request
                             |
                      backend Service
                        (port 8080)
                             |
                      Python backend Pod
                       /             \
             backend-config     backend-credentials
                ConfigMap             Secret
```

- **Frontend:** NGINX serves a simple web page and forwards requests under `/api/` to the backend Service.
- **Backend:** A small Python HTTP API reads application settings from a ConfigMap and checks that Secret-backed credentials are configured.
- **ConfigMaps:** Store non-sensitive settings such as application name, environment and frontend NGINX configuration.
- **Secret:** Stores dummy backend credentials. The application reports only whether the values are configured; it never returns the values.
- **Services:** Provide stable DNS names so the frontend can reach the backend using `backend` as its hostname.

> **Scope note:** This training application demonstrates the frontend-to-backend pattern. It does not connect to a real database; the Secret contains dummy database-style credentials to demonstrate secure configuration injection. Do not use production credentials in this lab.

## Prerequisites
- Access to an existing AKS cluster.
- Azure Portal Cloud Shell opened in **Bash** mode, with `kubectl` configured for the AKS cluster.
- Permission to create namespaces, ConfigMaps, Secrets, Deployments and Services.
- Cluster nodes must be able to pull the public `nginx:alpine` and `python:3.12-alpine` images.

---

## Step 1 – Verify cluster access and create a namespace

```bash
kubectl cluster-info
kubectl get nodes

NAMESPACE=aks-two-tier-lab
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
```

The first two commands verify cluster connectivity and node availability. The namespace keeps the lab resources together and makes cleanup easier.

## Step 2 – Create the frontend ConfigMap

The frontend ConfigMap contains the NGINX reverse-proxy configuration and the HTML page served to users.

Create a file: vi frontend-config.yaml

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
  namespace: aks-two-tier-lab
data:
  default.conf: |
    server {
      listen 80;
      server_name _;

      location / {
        root /usr/share/nginx/html;
        index index.html;
      }

      location /api/ {
        proxy_pass http://backend:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
      }
    }
  index.html: |
    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width, initial-scale=1">
      <title>AKS Two-Tier Demo</title>
    </head>
    <body>
      <h1>AKS Two-Tier Application</h1>
      <p>Frontend: NGINX</p>
      <h2>Backend API response</h2>
      <pre id="result">Loading backend response...</pre>
      <script>
        fetch('/api/')
          .then(response => {
            if (!response.ok) throw new Error(`HTTP ${response.status}`);
            return response.json();
          })
          .then(data => {
            document.getElementById('result').textContent =
              JSON.stringify(data, null, 2);
          })
          .catch(error => {
            document.getElementById('result').textContent =
              `Backend request failed: ${error}`;
          });
      </script>
    </body>
    </html>
```

```
kubectl apply -f frontend-config.yaml
kubectl describe configmap frontend-config -n "$NAMESPACE"
```

`default.conf` tells NGINX to serve the web page and forward `/api/` requests to the internal `backend` Service. `index.html` calls that API and displays its JSON response.

## Step 3 – Create the backend ConfigMap

The backend ConfigMap stores settings that are not confidential. These values are supplied to the backend container as environment variables.

Create a file: vi backend-config.yaml

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: aks-two-tier-lab
data:
  APP_NAME: "orders-api"
  APP_ENV: "training"
  APP_MESSAGE: "Response returned by the AKS backend API"
```

```
kubectl apply -f backend-config.yaml
kubectl describe configmap backend-config -n "$NAMESPACE"
```

## Step 4 – Create a Secret for backend credentials

Enter dummy training credentials. The password is hidden while typing and is not included literally in the command or YAML file.

```bash
read -r -p "Enter dummy database username: " DB_USERNAME
read -r -s -p "Enter dummy database password: " DB_PASSWORD
printf '\n'

if [[ -z "$DB_USERNAME" || -z "$DB_PASSWORD" ]]; then
  echo "Username and password must not be empty. Run this step again."
  unset DB_USERNAME DB_PASSWORD
  exit 1
fi

kubectl create secret generic backend-credentials \
  --namespace "$NAMESPACE" \
  --from-literal=DB_USERNAME="$DB_USERNAME" \
  --from-literal=DB_PASSWORD="$DB_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -

unset DB_USERNAME DB_PASSWORD
kubectl describe secret backend-credentials -n "$NAMESPACE"
```

`read -s` hides the password as it is entered. `secretKeyRef` will later map the Secret keys to environment variables in the backend container. `kubectl describe secret` shows key names without displaying their values.

> **Security note:** Kubernetes Secret data is Base64-encoded in its API representation; Base64 is not encryption. Use appropriate RBAC and encryption-at-rest protections in real environments. For production workloads on AKS, consider retrieving secrets from Azure Key Vault using an approved integration such as the Secrets Store CSI Driver.

## Step 5 – Deploy the backend API and its Service

The backend runs a small Python HTTP API on port `8080`. It reads application settings from the backend ConfigMap and credentials from the Secret. Its response includes only a Boolean status for the credentials, never their values.

Create a file: vi backend.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: aks-two-tier-lab
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
          image: python:3.12-alpine
          imagePullPolicy: IfNotPresent
          command: ["python", "-u", "-c"]
          args:
            - |
              import json
              import os
              from http.server import BaseHTTPRequestHandler, HTTPServer

              app_name = os.getenv("APP_NAME", "unknown")
              app_env = os.getenv("APP_ENV", "unknown")
              app_message = os.getenv("APP_MESSAGE", "")
              credentials_configured = bool(os.getenv("DB_USERNAME")) and bool(os.getenv("DB_PASSWORD"))

              class Handler(BaseHTTPRequestHandler):
                  def do_GET(self):
                      response = {
                          "application": app_name,
                          "environment": app_env,
                          "message": app_message,
                          "backend": "python-api",
                          "databaseCredentialsConfigured": credentials_configured
                      }
                      payload = (json.dumps(response, indent=2) + "\n").encode("utf-8")
                      self.send_response(200)
                      self.send_header("Content-Type", "application/json")
                      self.send_header("Content-Length", str(len(payload)))
                      self.end_headers()
                      self.wfile.write(payload)

              print("Backend API listening on port 8080")
              print("Database credentials configured:", credentials_configured)
              HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: APP_NAME
              valueFrom:
                configMapKeyRef:
                  name: backend-config
                  key: APP_NAME
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: backend-config
                  key: APP_ENV
            - name: APP_MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: backend-config
                  key: APP_MESSAGE
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: backend-credentials
                  key: DB_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: backend-credentials
                  key: DB_PASSWORD
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: aks-two-tier-lab
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 8080
      targetPort: http
```


```
kubectl apply -f backend.yaml
kubectl rollout status deployment/backend -n "$NAMESPACE"
kubectl get pods -l app=backend -n "$NAMESPACE"
kubectl get service backend -n "$NAMESPACE"
```

The Deployment manages the backend Pod. The `backend` ClusterIP Service provides a stable in-cluster DNS name and forwards port `8080` to the backend container.

## Step 6 – Deploy the NGINX frontend and its Service

The frontend Deployment mounts the ConfigMap files into the NGINX container. NGINX serves the page and proxies API requests to `http://backend:8080/` using Kubernetes Service discovery.

Create a file: vi frontend.yaml

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: aks-two-tier-lab
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
        - name: frontend
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: frontend-config-volume
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
              readOnly: true
            - name: frontend-config-volume
              mountPath: /usr/share/nginx/html/index.html
              subPath: index.html
              readOnly: true
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 2
            periodSeconds: 5
      volumes:
        - name: frontend-config-volume
          configMap:
            name: frontend-config
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: aks-two-tier-lab
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - name: http
      port: 80
      targetPort: http
```

```
kubectl apply -f frontend.yaml
kubectl rollout status deployment/frontend -n "$NAMESPACE"
kubectl get pods,services -n "$NAMESPACE" -o wide
```

The frontend Service exposes NGINX inside the cluster on port `80`. It is a ClusterIP Service, so it is not directly exposed to the public internet.

## Step 7 – Verify ConfigMap and Secret injection

Verify the backend received its non-sensitive ConfigMap values:

```bash
kubectl exec deployment/backend -n "$NAMESPACE" -- \
  python -c 'import os; print("APP_NAME:", os.getenv("APP_NAME")); print("APP_ENV:", os.getenv("APP_ENV")); print("APP_MESSAGE:", os.getenv("APP_MESSAGE"))'
```

Verify the Secret-backed variables are present without printing their values:

```bash
kubectl exec deployment/backend -n "$NAMESPACE" -- sh -c \
  'if [ -n "$DB_USERNAME" ] && [ -n "$DB_PASSWORD" ]; then echo "Both Secret environment variables are configured"; else echo "A required Secret environment variable is missing"; exit 1; fi'
```

Check the backend startup logs:

```bash
kubectl logs deployment/backend -n "$NAMESPACE"
```

Expected log pattern:

```text
Backend API listening on port 8080
Database credentials configured: True
```

## Step 8 – Test frontend-to-backend communication

Forward a Cloud Shell local port to the frontend Service. The frontend will then proxy `/api/` requests to the backend using the internal Service DNS name.

```bash
kubectl port-forward service/frontend 8080:80 -n "$NAMESPACE" \
  >/tmp/aks-two-tier-port-forward.log 2>&1 &
PF_PID=$!
sleep 3

curl --fail --silent --show-error http://127.0.0.1:8080/
printf '\n--- Backend API through frontend proxy ---\n'
curl --fail --silent --show-error http://127.0.0.1:8080/api/
CURL_STATUS=$?

kill "$PF_PID" 2>/dev/null || true
wait "$PF_PID" 2>/dev/null || true
test "$CURL_STATUS" -eq 0
```

Expected API response (the values should match your ConfigMap settings):

```json
{
  "application": "orders-api",
  "environment": "training",
  "message": "Response returned by the AKS backend API",
  "backend": "python-api",
  "databaseCredentialsConfigured": true
}
```

The successful `/api/` response confirms that the request reached the NGINX frontend, was forwarded to the backend Service and was handled by the Python API. The response confirms that the backend received both Secret-backed variables without revealing their contents.

> **Note:** The frontend ConfigMap uses `subPath` mounts. Changes to ConfigMap keys mounted this way are not automatically reflected in the mounted files; restart the frontend Pods after changing the NGINX or HTML configuration. Environment variables sourced from ConfigMaps or Secrets are set when a container starts; restart the relevant Pods after changing those values.

## Step 9 – Inspect the deployed resources

```bash
kubectl get deployments,pods,services,configmaps,secrets -n "$NAMESPACE"
kubectl get endpoints backend frontend -n "$NAMESPACE"
```

These commands provide a final view of the lab resources and show whether the Services have ready backend endpoints.

## Step 10 – Clean up lab resources

```bash
kubectl delete namespace "$NAMESPACE"
rm -f frontend-config.yaml backend-config.yaml backend.yaml frontend.yaml
rm -f /tmp/aks-two-tier-port-forward.log
```

Deleting the namespace removes the two Deployments, two Services, ConfigMaps, Secret and Pods created for this lab. The remaining commands remove the local manifest and temporary log files from Cloud Shell.

## Troubleshooting

| Issue | What to check |
|---|---|
| Pod is in `ImagePullBackOff` | Run `kubectl describe pod <pod-name> -n "$NAMESPACE"`. Check image name, registry access and node outbound connectivity. |
| Pod is in `CrashLoopBackOff` | Check `kubectl logs deployment/backend -n "$NAMESPACE"` or `kubectl logs deployment/frontend -n "$NAMESPACE"`. Review the Deployment manifest and container startup output. |
| Backend Pod is not ready | Check that the ConfigMap and Secret exist in the same namespace and that all referenced keys are spelled correctly. |
| Frontend returns `502 Bad Gateway` | Check `kubectl get endpoints backend -n "$NAMESPACE"`, backend Pod readiness and the NGINX proxy target `http://backend:8080/`. |
| Frontend page loads but API call fails | Check backend Service labels and selector, backend Pod status, and `kubectl logs deployment/frontend -n "$NAMESPACE"`. |
| Port-forward cannot bind to port 8080 | Use another local port, for example `8181:80`, and update the `curl` URL to port `8181`. |

Useful inspection commands:

```bash
kubectl get pods -n "$NAMESPACE" -o wide
kubectl describe deployment frontend -n "$NAMESPACE"
kubectl describe deployment backend -n "$NAMESPACE"
kubectl logs deployment/frontend -n "$NAMESPACE"
kubectl logs deployment/backend -n "$NAMESPACE"
```

## Completion checklist

- [ ] Connected to the AKS cluster through Azure Portal Cloud Shell (Bash).
- [ ] Created the `aks-two-tier-lab` namespace.
- [ ] Created the frontend and backend ConfigMaps.
- [ ] Created the `backend-credentials` Secret using dummy values.
- [ ] Deployed the backend API and its ClusterIP Service.
- [ ] Deployed the NGINX frontend and its ClusterIP Service.
- [ ] Verified ConfigMap and Secret injection in the backend.
- [ ] Accessed the frontend through port-forwarding.
- [ ] Confirmed that the frontend successfully proxied a request to the backend.
- [ ] Deleted the lab namespace and local files.

## Key takeaways

- A Kubernetes Deployment manages the desired number of application Pods; a Service provides a stable network endpoint for those Pods.
- The frontend reaches the backend through the backend Service DNS name (`backend`) rather than relying on a Pod IP address.
- ConfigMaps hold non-sensitive settings and can be consumed as environment variables or mounted files.
- Secrets are intended for sensitive values. Restrict access with RBAC and use encryption at rest; Base64 encoding by itself is not protection.
- Keep sensitive values out of container images, source code, manifests committed to source control and application responses.
- The frontend and backend are separate Deployments, so each tier can be updated and scaled independently.

## Official references

- [Kubernetes – Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes – Services, Load Balancing and Networking](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes – DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes – ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes – Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Microsoft Learn – Deploy an application to Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/tutorial-kubernetes-deploy-application)
