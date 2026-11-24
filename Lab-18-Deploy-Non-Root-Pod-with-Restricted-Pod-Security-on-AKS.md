# Lab 18 – Deploy a Non-Root Application Pod with Restricted Pod Security on AKS

## Lab objective

Apply the Kubernetes **Restricted Pod Security Standard** to a dedicated namespace and deploy a hardened NGINX application that runs as a non-root user.

## Learning outcomes

By completing this lab, participants will be able to:

- Enforce the `restricted` Pod Security Standard at namespace level.
- Configure Pod-level and container-level `securityContext` settings.
- Run an application with a non-root UID and GID.
- Disable privilege escalation and drop Linux capabilities.
- Apply the `RuntimeDefault` seccomp profile.
- Use a read-only root filesystem with a writable temporary volume.
- Verify the running container's identity and application response.

## Environment and prerequisites

- An existing AKS cluster with a Linux node pool.
- Access to the Azure subscription and AKS cluster.
- `kubectl` available in **Azure Portal → Cloud Shell → Bash**.
- Permission to create namespaces, Deployments and Services, and to label namespaces.
- Outbound image-pull access to Docker Hub (or an approved mirror containing `nginxinc/nginx-unprivileged`).

> **Execution environment:** Run all commands in Azure Portal Cloud Shell using Bash. Connect Cloud Shell to the intended AKS cluster before starting.

## Concepts

| Concept | Explanation |
|---|---|
| Pod Security Standards (PSS) | Kubernetes defines `privileged`, `baseline` and `restricted` profiles. `restricted` applies the strongest built-in Pod hardening requirements. |
| Pod Security Admission (PSA) | The Kubernetes admission controller evaluates Pod creation requests against the PSS level configured on the namespace. |
| `securityContext` | Kubernetes settings that control the user/group identity and security privileges of a Pod or container. |
| `runAsNonRoot` | Requires the container process to run with a non-zero user ID. |
| `allowPrivilegeEscalation: false` | Prevents a process from gaining more privileges than its parent process. |
| `capabilities.drop: ["ALL"]` | Removes all Linux capabilities from the container. |
| `seccompProfile: RuntimeDefault` | Applies the container runtime's default seccomp system-call filter. |
| `readOnlyRootFilesystem` | Makes the container's root filesystem read-only. This is additional hardening; it is not itself a Restricted PSS requirement. |

## Lab architecture

```text
Azure Portal Cloud Shell (Bash)
             |
             v
     AKS API Server / PSA
             |
             v
  Namespace: aks-restricted-lab
  Enforce: restricted
             |
             v
  Deployment: nginx-secure
             |
             v
  NGINX unprivileged Pod
  UID/GID: 101:101
  Port: 8080
  Root filesystem: read-only
  /tmp: writable emptyDir
             |
             v
  ClusterIP Service: nginx-secure
```

## Step 1 – Connect to the AKS cluster

In Azure Portal, open **Cloud Shell**, select **Bash**, and connect to the required AKS cluster if it is not already the current context.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl version
```

- `kubectl config current-context` displays the cluster context currently selected by kubectl.
- `kubectl get nodes -o wide` verifies cluster connectivity and displays node details, including operating system information.
- `kubectl version` displays the Kubernetes client and server versions.

Confirm that the cluster has at least one Linux node before deploying the Linux NGINX image.

## Step 2 – Create a dedicated namespace

```bash
kubectl create namespace aks-restricted-lab
```

A namespace provides an isolated scope for the lab resources and allows Pod Security Admission to enforce a policy only for this exercise.

## Step 3 – Enforce the Restricted Pod Security Standard

```bash
kubectl label --overwrite namespace aks-restricted-lab pod-security.kubernetes.io/enforce=restricted pod-security.kubernetes.io/enforce-version=latest pod-security.kubernetes.io/warn=restricted pod-security.kubernetes.io/warn-version=latest
```

- The `enforce=restricted` label causes the API server to reject Pods that do not satisfy the Restricted profile.
- The `warn=restricted` label also provides user-facing warnings for policy violations. The `latest` version tracks the policy shipped with the cluster's Kubernetes version.

Verify the namespace labels:

```bash
kubectl get namespace aks-restricted-lab --show-labels
```

Confirm that the output includes `pod-security.kubernetes.io/enforce=restricted`.

## Step 4 – Create a hardened NGINX Deployment

The `nginxinc/nginx-unprivileged` image is designed to run NGINX without root privileges and listens on port `8080` by default. The manifest explicitly sets UID/GID `101`, applies the Restricted profile settings, and makes the root filesystem read-only.

The `/tmp` directory is mounted as an `emptyDir` volume because NGINX needs a writable temporary location for its PID and temporary files.

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-secure
  namespace: aks-restricted-lab
  labels:
    app: nginx-secure
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-secure
  template:
    metadata:
      labels:
        app: nginx-secure
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        fsGroup: 101
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:stable-alpine
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
          volumeMounts:
            - name: nginx-tmp
              mountPath: /tmp
      volumes:
        - name: nginx-tmp
          emptyDir: {}
YAML
```

**Important settings in the manifest:**

- `runAsNonRoot: true` and `runAsUser: 101` make the process run as a non-root user.
- `allowPrivilegeEscalation: false` prevents privilege escalation.
- `capabilities.drop: ["ALL"]` removes all Linux capabilities.
- `seccompProfile.type: RuntimeDefault` applies the runtime's default syscall filter.
- `readOnlyRootFilesystem: true` protects the image filesystem from writes.
- The `/tmp` `emptyDir` volume provides temporary writable storage without making the container root filesystem writable.

## Step 5 – Verify the Deployment and Pod

```bash
kubectl rollout status deployment/nginx-secure -n aks-restricted-lab
kubectl get deployment,pods -n aks-restricted-lab -o wide
```

The Deployment rollout should complete and one NGINX Pod should show `Running` and `1/1` Ready.

If the Pod is not ready, inspect its events and logs:

```bash
kubectl describe pod -n aks-restricted-lab -l app=nginx-secure
kubectl logs -n aks-restricted-lab deployment/nginx-secure
```

## Step 6 – Verify the container runs as a non-root user

```bash
kubectl exec -n aks-restricted-lab deployment/nginx-secure -- id
```

The output should show a non-zero UID and GID, normally `uid=101` and `gid=101` for this image.

Verify that the temporary directory is writable:

```bash
kubectl exec -n aks-restricted-lab deployment/nginx-secure -- sh -c 'touch /tmp/psa-check && ls -l /tmp/psa-check'
```

The command should create the test file successfully in the mounted `emptyDir` volume.

## Step 7 – Expose the application internally

Create a ClusterIP Service for the NGINX Pod:

```bash
cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: nginx-secure
  namespace: aks-restricted-lab
spec:
  type: ClusterIP
  selector:
    app: nginx-secure
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
YAML
```

The Service exposes port `80` inside the cluster and forwards requests to the NGINX container on port `8080`.

Verify the Service and its endpoints:

```bash
kubectl get service nginx-secure -n aks-restricted-lab
kubectl get endpointslice -n aks-restricted-lab -l kubernetes.io/service-name=nginx-secure
```

## Step 8 – Test the NGINX application

Use `kubectl port-forward` to make the internal Service reachable from the Cloud Shell session:

```bash
kubectl port-forward -n aks-restricted-lab service/nginx-secure 8080:80 >/tmp/nginx-secure-port-forward.log 2>&1 &
PF_PID=$!
sleep 3
curl -I http://127.0.0.1:8080
kill "$PF_PID"
wait "$PF_PID" 2>/dev/null || true
```

The response should include `HTTP/1.1 200 OK`. If port `8080` is already in use, replace the local port in both the port-forward command and the `curl` URL with another available port, such as `8081`.

## Step 9 – Observe Restricted policy enforcement (negative test)

The following Pod deliberately requests root execution and privileged mode. The namespace's Restricted policy should reject it at admission time.

```bash
cat <<'YAML' | kubectl apply -f - || echo "Expected result: Pod rejected by Pod Security Admission."
apiVersion: v1
kind: Pod
metadata:
  name: restricted-violation
  namespace: aks-restricted-lab
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        privileged: true
        runAsUser: 0
YAML
```

Expected result: the API server returns a `Forbidden` or Pod Security admission error describing one or more Restricted policy violations. The invalid Pod should not be created.

Confirm that the violating Pod does not exist:

```bash
kubectl get pod restricted-violation -n aks-restricted-lab
```

`NotFound` is the expected result for this negative test.

## Step 10 – Review the applied security settings

```bash
kubectl get pod -n aks-restricted-lab -l app=nginx-secure -o yaml
kubectl describe namespace aks-restricted-lab
```

Review the Pod specification and confirm these values:

| Setting | Expected value |
|---|---|
| `runAsNonRoot` | `true` |
| `runAsUser` | `101` |
| `runAsGroup` | `101` |
| `seccompProfile.type` | `RuntimeDefault` |
| `allowPrivilegeEscalation` | `false` |
| `capabilities.drop` | `ALL` |
| `readOnlyRootFilesystem` | `true` |

## Expected outcome

- The namespace enforces the `restricted` Pod Security Standard.
- The hardened NGINX Pod is admitted and reaches the `Running` state.
- The NGINX process runs with a non-root UID/GID.
- The container has privilege escalation disabled, all Linux capabilities dropped, and the RuntimeDefault seccomp profile applied.
- The root filesystem is read-only while `/tmp` remains writable through an `emptyDir` volume.
- The application responds successfully through the ClusterIP Service.
- The deliberately privileged, root-running Pod is rejected by Pod Security Admission.

## Troubleshooting

| Issue | Possible cause | What to check |
|---|---|---|
| Pod rejected by Restricted policy | One or more required security settings are missing or invalid. | Review the admission error and compare the Pod securityContext with the manifest in Step 4. |
| `CreateContainerError` or container exits | The image or its startup process may need a writable path or compatible user permissions. | Inspect `kubectl describe pod` and `kubectl logs`; confirm `/tmp` is mounted and writable. |
| `ImagePullBackOff` | The cluster cannot pull the image or the registry is unavailable. | Check Pod events, registry access and image-pull permissions. |
| Pod remains `Pending` | No suitable Linux node has available resources, or scheduling constraints prevent placement. | Run `kubectl describe pod` and `kubectl get nodes`. |
| `curl` cannot connect | Port-forward did not start, the local port is busy, or the Service has no ready endpoints. | Review `/tmp/nginx-secure-port-forward.log`, Service details and EndpointSlices. |

## Cleanup

Delete the lab namespace and all resources created in it:

```bash
kubectl delete namespace aks-restricted-lab
```

> This deletes the namespace, Deployment, Pod, Service and temporary volume created for this lab. Do not run the cleanup command if you intend to reuse the resources for another exercise.

## Knowledge check

1. What is the difference between the `baseline` and `restricted` Pod Security Standards?
2. Why are both `runAsNonRoot: true` and a non-zero `runAsUser` configured?
3. What is the purpose of setting `allowPrivilegeEscalation: false`?
4. Why does the container drop all Linux capabilities?
5. What does `seccompProfile: RuntimeDefault` provide?
6. Why is `/tmp` mounted as an `emptyDir` when `readOnlyRootFilesystem` is enabled?
7. What response should you expect when a privileged Pod is submitted to a namespace enforcing `restricted`?

## Official references

- [Use Pod Security Admission in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/use-psa)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [NGINX unprivileged container image](https://hub.docker.com/r/nginxinc/nginx-unprivileged/)
