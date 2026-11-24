# Lab 23 – Troubleshoot CrashLoopBackOff on AKS

## Lab details

| Item | Description |
|---|---|
| **Environment** | Azure Portal – Cloud Shell (Bash) |
| **Estimated duration** | 20 minutes |
| **Difficulty** | Intermediate |
| **Namespace** | `crashloop-lab` |
| **Workload** | NGINX Deployment with an intentionally incorrect startup argument |

## Objective

Troubleshoot a Deployment experiencing `CrashLoopBackOff`. Inspect container logs, Pod events and the Deployment specification, identify the incorrect NGINX configuration-file path, correct the Deployment and verify that the application becomes healthy.

## Concepts

- **CrashLoopBackOff:** Kubernetes has observed a container repeatedly start and terminate. It waits for progressively longer intervals before trying to restart the container again.
- **Container logs:** Application output written to standard output and standard error. Use `kubectl logs --previous` to retrieve output from the container instance that terminated most recently.
- **Command and args:** In a Pod specification, `command` overrides the image entrypoint and `args` overrides the image's default command arguments. An invalid command or argument can prevent an application from starting.
- **Deployment:** Maintains the desired number of Pod replicas and replaces Pods when the Pod template changes or a Pod fails.

## Prerequisites

- Access to an existing, running AKS cluster.
- Permission to create and update Deployments and Pods in the cluster.
- Azure Portal Cloud Shell opened in **Bash** mode.
- `kubectl` configured to communicate with the target AKS cluster.

> **Note:** This lab intentionally creates a failing workload in a dedicated namespace. Do not run the exercise in a production namespace.

---

## Step 1 – Connect to the AKS cluster

In the Azure portal, open **Cloud Shell**, select **Bash**, and run:

```bash
az account show --output table
kubectl cluster-info
kubectl get nodes
```

**Explanation:** These commands confirm the active Azure subscription, Kubernetes API connectivity and node readiness before deploying the lab workload.

Set the namespace variable:

```bash
NAMESPACE=crashloop-lab
```

**Explanation:** The variable keeps the commands in this lab targeted to one dedicated namespace.

## Step 2 – Create a dedicated namespace

```bash
kubectl create namespace "$NAMESPACE"
```

Verify the namespace:

```bash
kubectl get namespace "$NAMESPACE"
```

**Explanation:** A namespace isolates the lab resources from workloads used by other teams or exercises.

## Step 3 – Deploy NGINX with an incorrect configuration path

The following Deployment uses a valid NGINX image but supplies a configuration-file path that does not exist in the image. NGINX will exit during startup, causing Kubernetes to restart the container.

Create the manifest:

```bash
cat <<'EOF' > crash-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crash-demo
  namespace: crashloop-lab
  labels:
    app: crash-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crash-demo
  template:
    metadata:
      labels:
        app: crash-demo
    spec:
      containers:
        - name: nginx
          image: nginx:stable-alpine
          imagePullPolicy: IfNotPresent
          command: ["nginx"]
          args:
            - "-g"
            - "daemon off;"
            - "-c"
            - "/etc/nginx/nginx-lab-invalid.conf"
          ports:
            - name: http
              containerPort: 80
EOF
```

Apply the manifest:

```bash
kubectl apply -f crash-demo.yaml
```

**Explanation:** The container starts NGINX with an invalid `-c` configuration-file path. NGINX cannot open the file and exits, giving Kubernetes a failed container to investigate.

## Step 4 – Observe the Pod status

```bash
kubectl get pods -n "$NAMESPACE" -w
```

Wait until the Pod displays `CrashLoopBackOff`, then press **Ctrl+C** to stop watching.

List the Deployment and ReplicaSet:

```bash
kubectl get deployment,replicaset,pods -n "$NAMESPACE"
```

**Explanation:** `CrashLoopBackOff` is a waiting state that indicates repeated container failures and a restart delay. It is not the original cause; inspect logs and events to determine why the process exits.

> **Observation:** The Pod may first show `ContainerCreating`, `Error` or `CrashLoopBackOff`. The exact timing depends on image availability and Kubernetes restart backoff.

## Step 5 – Inspect the Pod configuration and events

Get the Pod name:

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=crash-demo -o jsonpath='{.items[0].metadata.name}')
echo "$POD"
```

Describe the Pod:

```bash
kubectl describe pod "$POD" -n "$NAMESPACE"
```

Review the following fields in the output:

- **State / Last State:** Current and previous container states.
- **Reason:** May show `CrashLoopBackOff` or `Error`.
- **Exit Code:** A non-zero exit code indicates that the process terminated with an error.
- **Restart Count:** Number of container restarts.
- **Events:** May contain `BackOff` or `Back-off restarting failed container` messages.

List recent namespace events:

```bash
kubectl get events -n "$NAMESPACE" --sort-by='.metadata.creationTimestamp'
```

**Explanation:** Pod events provide lifecycle and restart information. They help confirm that Kubernetes is repeatedly restarting the container, while the application logs are used to identify the startup error.

## Step 6 – Retrieve the application logs

Retrieve logs from the most recently terminated container instance:

```bash
kubectl logs "$POD" -n "$NAMESPACE" --previous
```

If the Pod has not restarted yet, or there is no previous container instance, retrieve the current container logs:

```bash
kubectl logs "$POD" -n "$NAMESPACE"
```

**Explanation:** `--previous` displays the logs from the preceding terminated container instance. This is especially useful when the current container is restarting frequently.

**Expected log message:** The NGINX error should indicate that it cannot open `/etc/nginx/nginx-lab-invalid.conf` because the file does not exist. The exact wording may vary by NGINX image version.

## Step 7 – Inspect the Deployment command and arguments

Display the container image, command and arguments configured in the Deployment template:

```bash
kubectl get deployment crash-demo -n "$NAMESPACE" -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}{.spec.template.spec.containers[0].command}{"\n"}{.spec.template.spec.containers[0].args}{"\n"}'
```

You can also inspect the complete Deployment specification:

```bash
kubectl get deployment crash-demo -n "$NAMESPACE" -o yaml
```

**Explanation:** The Deployment template confirms which image, command and arguments Kubernetes uses whenever it creates a replacement Pod. In this lab, the incorrect configuration-file path is supplied through `args`.

**Root cause:** NGINX is being instructed to load a configuration file that is not present in the container image. NGINX exits during startup, and Kubernetes repeatedly restarts it.

## Step 8 – Correct the Deployment

Update the arguments to remove the invalid `-c` configuration-file override. NGINX will then use its default configuration file.

```bash
kubectl patch deployment crash-demo -n "$NAMESPACE" --type='json' -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args","value":["-g","daemon off;"]}]'
```

**Explanation:** Updating the Pod template causes the Deployment to create a replacement Pod. The corrected command keeps NGINX running in the foreground, as required for a container workload.

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/crash-demo -n "$NAMESPACE" --timeout=180s
```

## Step 9 – Verify the application recovery

Check the Deployment and Pod status:

```bash
kubectl get deployment crash-demo -n "$NAMESPACE"
kubectl get pods -n "$NAMESPACE" -o wide
```

Expected result:

- Deployment shows `READY` as `1/1`.
- The new Pod shows `Running` and `READY` as `1/1`.
- The new Pod should have no repeated restarts.

Check the NGINX configuration:

```bash
POD=$(kubectl get pods -n "$NAMESPACE" -l app=crash-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -n "$NAMESPACE" -- nginx -t
```

**Explanation:** `nginx -t` checks the NGINX configuration syntax and confirms that the default configuration can be loaded.

Optionally, verify that NGINX responds over HTTP by forwarding a local Cloud Shell port:

```bash
kubectl port-forward deployment/crash-demo 8080:80 -n "$NAMESPACE"
```

Keep the command running. Open another Cloud Shell tab and run:

```bash
curl -I http://127.0.0.1:8080
```

A successful response should include an HTTP `200 OK` status. Stop the port-forward command with **Ctrl+C** when finished.

## Step 10 – Clean up lab resources

Delete the lab namespace and its resources:

```bash
kubectl delete namespace "$NAMESPACE"
```

Remove the local manifest:

```bash
rm -f crash-demo.yaml
```

**Explanation:** Deleting the namespace removes the Deployment, ReplicaSet and Pods created for this exercise.

---

## Troubleshooting notes

| Symptom | What to check |
|---|---|
| Pod remains in `ContainerCreating` | Check `kubectl describe pod` for image-pull, volume or scheduling events. |
| Pod shows `ImagePullBackOff` | Confirm the image name and tag and check registry access. This is different from the startup-configuration failure used in this lab. |
| `kubectl logs` returns no useful output | Try `kubectl logs POD_NAME --previous -n crashloop-lab` after the container has restarted. |
| Pod remains in `CrashLoopBackOff` after the patch | Inspect the new Pod's logs and events, and confirm that the Deployment arguments no longer contain the invalid path. |
| Rollout does not complete | Run `kubectl describe deployment crash-demo -n crashloop-lab` and `kubectl describe pod POD_NAME -n crashloop-lab` to inspect the latest status and events. |
| Port-forward does not respond | Confirm the Pod is `Running` and `Ready`, then check that the port-forward command is still running. |

## Completion checklist

- [ ] Connected to the correct AKS cluster from Azure Cloud Shell (Bash).
- [ ] Created the `crashloop-lab` namespace.
- [ ] Deployed NGINX with an intentionally invalid configuration-file path.
- [ ] Observed the Pod enter `CrashLoopBackOff`.
- [ ] Inspected Pod status, restart count and events.
- [ ] Retrieved the previous container logs using `kubectl logs --previous`.
- [ ] Identified the incorrect NGINX configuration-file path in the Deployment arguments.
- [ ] Corrected the Deployment and verified that the Pod is `Running` and `Ready`.
- [ ] Deleted the lab namespace and local manifest.

## Knowledge check

1. What does the `CrashLoopBackOff` status indicate?
2. Why is `kubectl logs --previous` useful when a container restarts repeatedly?
3. Which section of the Deployment manifest defines the container command and arguments?
4. What was the root cause of the failure in this exercise?
5. Why does changing the Deployment Pod template cause Kubernetes to create a new Pod?

## Expected outcome

Participants can use `kubectl get`, `kubectl describe`, `kubectl logs --previous` and Deployment inspection to diagnose an application startup failure, identify an invalid container argument, correct the Deployment and verify that the workload recovers.

## References

- [Microsoft Learn – Pod stuck in CrashLoopBackOff mode](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/pod-stuck-crashloopbackoff-mode)
- [Kubernetes – Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes – Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes – kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/)
