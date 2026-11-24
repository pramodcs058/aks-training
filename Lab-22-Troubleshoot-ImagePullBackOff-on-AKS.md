# Lab 22 – Troubleshoot ImagePullBackOff on AKS

## Lab details

| Item | Description |
|---|---|
| **Module** | Module 8 – Upgrades, Troubleshooting & Capstone Project |
| **Lab type** | Troubleshooting exercise |
| **Estimated duration** | 15–20 minutes |
| **Execution environment** | Azure Portal – Cloud Shell – Bash |
| **Prerequisites** | Access to an existing AKS cluster and permission to create Kubernetes resources in a namespace |

## Lab objective

Deploy an application with an intentionally incorrect container image reference. Inspect the Pod status, events and Deployment configuration to identify why Kubernetes cannot pull the image.

## Learning outcomes

By the end of this lab, participants will be able to:

- Recognize the `ErrImagePull` and `ImagePullBackOff` Pod states.
- Use `kubectl get`, `kubectl describe` and `kubectl get events` to investigate a failed image pull.
- Identify an incorrect image repository or tag from Kubernetes event messages.
- Correct the image reference and verify that the Deployment recovers.

## Concepts

- **ImagePullBackOff:** Kubernetes cannot download a container image. It retries the image pull with an increasing delay; the Pod generally remains in the `Pending` phase while the container is waiting.
- **ErrImagePull:** An image-pull attempt has failed. After repeated failures, the waiting reason commonly changes to `ImagePullBackOff`.
- **Pod events:** Events record scheduling, image-pull and container lifecycle activity. The `Events` section of `kubectl describe pod` is usually the first place to look for the specific failure message.
- **Deployment:** A Deployment manages the desired number of Pod replicas and their Pod template, including the container image reference.

## Scenario

A team has deployed an NGINX application to AKS, but the Pod never starts. The Deployment references a container image tag that does not exist in the public NGINX image repository. Your task is to identify the incorrect reference, correct it and confirm that the application Pod becomes Ready.

> **Important:** This lab intentionally deploys an invalid image tag. Do not use this manifest for a production workload.

---

## Step 1 – Open Azure Cloud Shell

1. Sign in to the [Azure portal](https://portal.azure.com/).
2. Open **Cloud Shell** from the portal toolbar.
3. Select **Bash** as the shell environment.
4. Confirm that `kubectl` can reach the intended AKS cluster:

```bash
kubectl cluster-info
kubectl get nodes
```

The node list should show the cluster nodes. If these commands fail, verify that Cloud Shell is connected to the correct cluster and context before continuing.

## Step 2 – Create a namespace

```bash
kubectl create namespace image-pull-lab
```

A namespace isolates the resources created for this exercise from workloads in other namespaces.

Verify the namespace:

```bash
kubectl get namespace image-pull-lab
```

## Step 3 – Deploy NGINX with an incorrect image tag

The manifest below deliberately uses a tag that does not exist. The Deployment can be created successfully, but its Pod will not become Ready because the node cannot retrieve the referenced image.

```bash
kubectl apply -n image-pull-lab -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: image-pull-demo
  labels:
    app: image-pull-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: image-pull-demo
  template:
    metadata:
      labels:
        app: image-pull-demo
    spec:
      containers:
        - name: nginx
          image: nginx:lab-tag-that-does-not-exist
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 80
EOF
```

**What this manifest does:** It creates a Deployment named `image-pull-demo` with one replica. The intentionally invalid image tag is `lab-tag-that-does-not-exist`; `Always` ensures the node checks the registry rather than relying on a locally cached image.

Verify that the Deployment was created:

```bash
kubectl get deployment image-pull-demo -n image-pull-lab
```

The Deployment object can exist even when its Pod cannot start. Look at the `READY` and `AVAILABLE` columns rather than assuming that successful `kubectl apply` means the application is running.

## Step 4 – Inspect the Pod status

List the Pods and watch the status change:

```bash
kubectl get pods -n image-pull-lab -o wide
```

```bash
kubectl get pods -n image-pull-lab -w
```

Press **Ctrl+C** to stop watching after the Pod shows `ErrImagePull` or `ImagePullBackOff`.

The transition may take a short time. Kubernetes initially reports an image-pull error and then applies a back-off delay before retrying. The precise timing and displayed message can vary.

**Expected observation:** The Pod is not Ready. Its status may show `ErrImagePull` briefly and then `ImagePullBackOff`. The Pod phase can remain `Pending` because the container has not started.

## Step 5 – Inspect Pod details using kubectl describe

Get the Pod name:

```bash
POD_NAME=$(kubectl get pods -n image-pull-lab -l app=image-pull-demo -o jsonpath='{.items[0].metadata.name}')
echo "$POD_NAME"
```

Describe the Pod:

```bash
kubectl describe pod "$POD_NAME" -n image-pull-lab
```

Review these sections in the output:

- **Containers:** Check the image name and container state.
- **State / Waiting:** Look for `ErrImagePull` or `ImagePullBackOff` and its message.
- **Events:** Look for `Failed`, `Failed to pull image` or `Back-off pulling image` messages.

**Expected finding:** The event message should indicate that the requested repository/tag or image manifest cannot be found. The exact text depends on the registry response and Kubernetes/container-runtime version.

## Step 6 – Review namespace events

List events in chronological order:

```bash
kubectl get events -n image-pull-lab --sort-by='.metadata.creationTimestamp'
```

To focus on events associated with the failing Pod:

```bash
kubectl get events -n image-pull-lab --field-selector involvedObject.name="$POD_NAME" --sort-by='.metadata.creationTimestamp'
```

Inspect the **TYPE**, **REASON** and **MESSAGE** columns. `Warning` events with reasons such as `Failed` or `BackOff` provide evidence of the image-pull failure.

## Step 7 – Inspect the Deployment configuration

Describe the Deployment:

```bash
kubectl describe deployment image-pull-demo -n image-pull-lab
```

Display the image configured in the Deployment's Pod template:

```bash
kubectl get deployment image-pull-demo -n image-pull-lab -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Inspect the ReplicaSet created by the Deployment:

```bash
kubectl get replicasets -n image-pull-lab -l app=image-pull-demo
```

**Root cause:** The Deployment references `nginx:lab-tag-that-does-not-exist`. The requested image tag is invalid, so the container runtime cannot retrieve the image from the registry. This is an image reference problem, not an application process failure.

> **Troubleshooting distinction:** If the event instead reports `401 Unauthorized`, `403 Forbidden`, a missing image-pull Secret, DNS failure or a network timeout, the cause is likely registry authentication, authorization or connectivity—not necessarily an incorrect tag. Always diagnose from the actual event message.

## Step 8 – Correct the image reference

Update the Deployment to use the valid official NGINX `stable` tag:

```bash
kubectl set image deployment/image-pull-demo nginx=nginx:stable -n image-pull-lab
```

This updates the container image in the Deployment's Pod template and triggers a new rollout.

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/image-pull-demo -n image-pull-lab --timeout=180s
```

If the cluster cannot reach the public registry or is subject to registry rate limits, the replacement image may also fail to pull. In that case, inspect the new Pod events rather than assuming the original problem remains.

## Step 9 – Verify application recovery

```bash
kubectl get deployment image-pull-demo -n image-pull-lab
kubectl get pods -n image-pull-lab -o wide
```

Confirm that:

- The Deployment shows `1/1` Ready and `1` available replica.
- The replacement Pod shows `Running` and `1/1` Ready.
- The Pod uses the corrected `nginx:stable` image.

Verify the image configured in the Deployment:

```bash
kubectl get deployment image-pull-demo -n image-pull-lab -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Optionally, confirm that NGINX responds from inside the Pod:

```bash
POD_NAME=$(kubectl get pods -n image-pull-lab -l app=image-pull-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD_NAME" -n image-pull-lab -- nginx -t
```

A successful configuration test reports that the NGINX configuration syntax is valid.

## Step 10 – Clean up

Delete the namespace and all resources created for this lab:

```bash
kubectl delete namespace image-pull-lab
```

Verify that the namespace has been removed:

```bash
kubectl get namespace image-pull-lab
```

A `NotFound` response confirms that the namespace no longer exists.

---

## Troubleshooting reference

| Observation | Possible meaning | What to inspect |
|---|---|---|
| `ErrImagePull` | An image-pull attempt failed | Pod events in `kubectl describe pod` |
| `ImagePullBackOff` | Kubernetes is retrying a failed image pull with a delay | Image name, tag and most recent event message |
| `manifest unknown` / `not found` | Repository or tag may not exist | Image repository and tag in the Deployment |
| `401 Unauthorized` | Registry authentication failed | Registry credentials and image-pull configuration |
| `403 Forbidden` | Registry access is denied | Registry permissions, network rules or private endpoint configuration |
| `FailedToRetrieveImagePullSecret` | Referenced image-pull Secret is unavailable | Secret name, namespace and Pod template |
| DNS or connection timeout | Node cannot reach or resolve the registry endpoint | AKS outbound connectivity, DNS and firewall rules |
| Pod remains `Pending` | The Pod has not completed startup; image pull is one possible cause | Pod conditions, scheduling details and events |

**Note:** `kubectl logs` is generally not useful while the image cannot be pulled because the application container has not started. Use Pod status and events to diagnose this failure.

## Validation checklist

- [ ] Connected to the correct AKS cluster from Azure Cloud Shell.
- [ ] Created the `image-pull-lab` namespace.
- [ ] Deployed the NGINX Deployment with the intentionally incorrect image tag.
- [ ] Observed `ErrImagePull` or `ImagePullBackOff`.
- [ ] Used `kubectl describe pod` to inspect the container state and events.
- [ ] Used `kubectl get events` to identify the image-pull error.
- [ ] Inspected the Deployment image reference and identified the incorrect tag.
- [ ] Updated the image to `nginx:stable`.
- [ ] Verified that the Deployment and Pod became Ready.
- [ ] Deleted the lab namespace.

## Knowledge check

1. What is the difference between `ErrImagePull` and `ImagePullBackOff`?
2. Why can a Deployment be created successfully even when its Pod cannot start?
3. Which section of `kubectl describe pod` provides the most useful evidence for an image-pull failure?
4. What does a `manifest unknown` or `not found` event message usually indicate?
5. Why is `kubectl logs` not normally the first troubleshooting command for `ImagePullBackOff`?
6. Name two other causes of image-pull failures that are not related to an incorrect image tag.

## Key takeaways

- A successful Deployment creation does not guarantee that its Pods can start.
- `ImagePullBackOff` indicates that Kubernetes cannot pull the specified container image and is retrying with a back-off delay.
- Use `kubectl get pods` to identify the symptom, `kubectl describe pod` and `kubectl get events` to find evidence, and Deployment inspection to verify the configured image reference.
- Correct the root cause, then verify the rollout and Pod readiness.

## Official references

- [Microsoft Learn – Troubleshoot AKS image pull errors from Azure Container Registry](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connectivity/cannot-pull-image-from-acr-to-aks-cluster)
- [Kubernetes – Images and ImagePullBackOff](https://kubernetes.io/docs/concepts/containers/images/)
- [Kubernetes – kubectl describe](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_describe/)
- [Microsoft Learn – Troubleshoot app connection and workload issues in AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/connection-issues-application-hosted-aks-cluster)
