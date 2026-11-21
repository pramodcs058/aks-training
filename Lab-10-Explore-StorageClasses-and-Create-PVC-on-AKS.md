# Lab 10 – Explore StorageClasses and Create a PersistentVolumeClaim on AKS

## Lab details

| Item | Description |
|---|---|
| **Environment** | Azure Portal – Cloud Shell – Bash |
| **Platform** | Azure Kubernetes Service (AKS) |
| **Estimated duration** | 15–20 minutes |
| **Level** | Beginner to Intermediate |

## Objective

Explore the StorageClasses available in an AKS cluster, select an existing Azure Disk CSI StorageClass, and create a PersistentVolumeClaim (PVC) that dynamically provisions persistent storage. Use a test Pod to trigger provisioning when the selected StorageClass uses `WaitForFirstConsumer`, then verify the PVC is bound to a PersistentVolume (PV).

## Concepts

- **StorageClass:** Defines how Kubernetes should provision storage, including the CSI provisioner, storage tier, reclaim policy and volume-binding behaviour.
- **PersistentVolume (PV):** A cluster-level representation of a storage resource made available to Kubernetes. In this lab, AKS dynamically provisions the backing Azure managed disk.
- **PersistentVolumeClaim (PVC):** A namespaced request for storage that specifies capacity, access mode and optionally a StorageClass.
- **Dynamic provisioning:** Kubernetes uses the selected StorageClass to request storage automatically when a suitable PVC is created.
- **`ReadWriteOnce` (RWO):** Allows a volume to be mounted read-write by workloads on one node at a time. It does not mean that only one Pod can use the volume if those Pods are on the same node.

## Prerequisites

- An existing, running AKS cluster.
- Permission to connect to the cluster and create namespaces, Pods and PVCs.
- An AKS cluster with the Azure Disk CSI driver available and at least one Azure Disk CSI StorageClass.
- Azure Portal access to the subscription containing the AKS cluster.

> **Storage choice for this lab:** Select an existing StorageClass whose provisioner is `disk.csi.azure.com`, such as `managed-csi` or `managed-csi-premium` when available. This lab requests `ReadWriteOnce` storage. Do not select an Azure Files, Azure Blob or other StorageClass for this specific exercise.
>
> Creating a PVC can provision a billable Azure managed disk. Use a small request (1 Gi) and complete the cleanup steps after the lab.

---

## Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Open **Cloud Shell** and select **Bash**.
3. Select the subscription that contains your AKS cluster.
4. Set the cluster details below and retrieve the cluster credentials.

```bash
# Set these values for your environment
RESOURCE_GROUP="<your-resource-group>"
AKS_CLUSTER="<your-aks-cluster-name>"

# Confirm the active subscription
az account show --output table

# Connect kubectl to the AKS cluster
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER"

# Verify cluster connectivity
kubectl get nodes
```

**Command notes:** `az aks get-credentials` retrieves the Kubernetes credentials and merges the AKS context into the Cloud Shell kubeconfig. `kubectl get nodes` confirms that kubectl can communicate with the cluster.

## Step 2 – Explore available StorageClasses

```bash
# List StorageClasses available in the cluster
kubectl get storageclass

# Display provisioner, reclaim policy, binding mode and expansion setting
kubectl get storageclass \
  -o custom-columns='NAME:.metadata.name,PROVISIONER:.provisioner,RECLAIM:.reclaimPolicy,BINDING:.volumeBindingMode,EXPAND:.allowVolumeExpansion'
```

Identify a StorageClass whose provisioner is `disk.csi.azure.com`. AKS clusters commonly provide `managed-csi` and `managed-csi-premium`, but the available names depend on the cluster configuration.

Set the name of the StorageClass you will use:

```bash
# Replace the value with an existing Azure Disk CSI StorageClass from the list
STORAGE_CLASS="managed-csi"

# Inspect the selected StorageClass
kubectl describe storageclass "$STORAGE_CLASS"

# Confirm that it uses the Azure Disk CSI driver
kubectl get storageclass "$STORAGE_CLASS" -o jsonpath='{.provisioner}{"\n"}'
```

**Expected provisioner:** `disk.csi.azure.com`

If the provisioner is different, select an Azure Disk CSI StorageClass before continuing.

**What to observe:**

- `Provisioner` identifies the CSI driver responsible for creating the storage.
- `ReclaimPolicy` determines what happens to the backing storage after the PV is released.
- `VolumeBindingMode` determines when provisioning and binding occur.
- `AllowVolumeExpansion` indicates whether a bound volume can be expanded.

## Step 3 – Create a namespace for the lab

```bash
NAMESPACE="aks-storage-lab"
kubectl create namespace "$NAMESPACE"
```

A dedicated namespace keeps the lab's PVC and test Pod separate from other workloads.

## Step 4 – Create a PersistentVolumeClaim

Create a PVC named `lab-pvc` requesting 1 Gi of storage from the selected StorageClass.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
  namespace: ${NAMESPACE}
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ${STORAGE_CLASS}
  resources:
    requests:
      storage: 1Gi
EOF
```

**Manifest notes:** `storageClassName` selects the provisioner and storage configuration. `accessModes` requests read-write access from one node at a time, while `resources.requests.storage` specifies the requested capacity.

Check the claim:

```bash
kubectl get pvc -n "$NAMESPACE"
kubectl describe pvc lab-pvc -n "$NAMESPACE"
```

The PVC may show `Pending` at this stage. This can be expected when the StorageClass uses `WaitForFirstConsumer`.

## Step 5 – Create a test Pod that uses the PVC

A StorageClass with `WaitForFirstConsumer` waits until a Pod references the claim before provisioning and binding the volume. Create a small BusyBox Pod that mounts the PVC and writes a test file.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-test
  namespace: aks-storage-lab
spec:
  containers:
    - name: storage-test
      image: busybox:1.36
      command: ["/bin/sh", "-c"]
      args:
        - echo "AKS persistent storage test" > /data/lab-test.txt && sleep 3600
      volumeMounts:
        - name: persistent-data
          mountPath: /data
  volumes:
    - name: persistent-data
      persistentVolumeClaim:
        claimName: lab-pvc
EOF
```

The Pod mounts the PVC at `/data`. Writing a file confirms that the container can use the provisioned volume.

Wait for the Pod to become Ready:

```bash
kubectl wait --for=condition=Ready pod/storage-test \
  --namespace "$NAMESPACE" --timeout=300s
```

## Step 6 – Verify PVC and PV binding

```bash
# Verify the PVC is Bound
kubectl get pvc lab-pvc --namespace "$NAMESPACE"

# Retrieve the name of the PV bound to this claim
PV_NAME=$(kubectl get pvc lab-pvc --namespace "$NAMESPACE" \
  -o jsonpath='{.spec.volumeName}')

echo "Bound PersistentVolume: $PV_NAME"

# Inspect the PV
kubectl get pv "$PV_NAME"
kubectl describe pv "$PV_NAME"
```

**Expected result:** The PVC status should be `Bound`, and the PV should show the claim `aks-storage-lab/lab-pvc`. The PV details should identify the CSI driver and backing volume information.

## Step 7 – Verify that the Pod can write to the mounted volume

```bash
# Read the file written by the test container
kubectl exec --namespace "$NAMESPACE" storage-test -- cat /data/lab-test.txt

# Inspect the Pod and its mounted volume
kubectl describe pod storage-test --namespace "$NAMESPACE"
```

**Expected output from the file check:**

```text
AKS persistent storage test
```

This confirms that the PVC was mounted into the container and that the container can write to and read from the volume.

## Step 8 – Review the storage relationship

```bash
kubectl get storageclass "$STORAGE_CLASS"
kubectl get pvc --namespace "$NAMESPACE"
kubectl get pv "$PV_NAME"
kubectl get pod storage-test --namespace "$NAMESPACE"
```

The relationship is:

```text
StorageClass
    |
    | defines how storage is provisioned
    v
PersistentVolumeClaim (lab-pvc)
    |
    | dynamically provisions and binds to
    v
PersistentVolume (Azure Disk-backed PV)
    |
    | mounted by
    v
Pod (storage-test) --> /data/lab-test.txt
```

## Step 9 – Clean up lab resources

> **Important:** Check the PV reclaim policy before deleting the PVC. With `Delete`, Kubernetes normally deletes the dynamically provisioned backing disk when the claim is released. With `Retain`, the disk is kept and may require manual cleanup in Azure.

```bash
# Check the reclaim policy recorded on the PV
kubectl get pv "$PV_NAME" \
  -o jsonpath='{.spec.persistentVolumeReclaimPolicy}{"\n"}'

# Delete the test Pod and PVC
kubectl delete pod storage-test --namespace "$NAMESPACE"
kubectl delete pvc lab-pvc --namespace "$NAMESPACE"

# Confirm the namespace no longer contains lab resources
kubectl get all --namespace "$NAMESPACE"
```

If the reclaim policy is `Retain`, inspect the released PV and its Azure managed disk before removing anything. The underlying disk can continue to incur charges until it is explicitly cleaned up. Do not delete shared StorageClasses or resources used by other workloads.

After confirming the lab namespace is empty and no retained resource needs investigation, remove the namespace:

```bash
kubectl delete namespace "$NAMESPACE"
```

---

## Troubleshooting

| Issue | Possible cause | What to check |
|---|---|---|
| PVC remains `Pending` | The StorageClass uses `WaitForFirstConsumer` and no Pod is using the PVC yet. | Create the `storage-test` Pod, then check `kubectl describe pvc lab-pvc -n aks-storage-lab`. |
| PVC remains `Pending` after Pod creation | CSI provisioning error, unavailable capacity, unsupported StorageClass, or scheduling constraints. | Check `kubectl describe pvc`, `kubectl describe pod storage-test -n aks-storage-lab`, and the Pod/PVC events. |
| Pod remains `Pending` | The Pod cannot be scheduled, or volume provisioning/attachment has not completed. | Run `kubectl describe pod storage-test -n aks-storage-lab` and inspect node capacity, taints and events. |
| Pod reports `ContainerCreating` | The volume may still be provisioning or attaching. | Check `kubectl describe pod` and `kubectl describe pvc` for CSI events. |
| `kubectl exec` cannot find the file | The container has not finished its command or the volume is not mounted at the expected path. | Check Pod status and inspect `kubectl describe pod storage-test -n aks-storage-lab`. |
| Selected StorageClass has another provisioner | The wrong type of storage was selected for this RWO Azure Disk exercise. | Choose a StorageClass whose provisioner is `disk.csi.azure.com`. |

## Completion checklist

- [ ] Connected to the AKS cluster from Azure Portal Cloud Shell (Bash).
- [ ] Listed the available StorageClasses.
- [ ] Identified and inspected an Azure Disk CSI StorageClass.
- [ ] Created the `aks-storage-lab` namespace.
- [ ] Created the `lab-pvc` claim requesting 1 Gi.
- [ ] Created a Pod that consumes the PVC.
- [ ] Verified that the PVC reached `Bound` status.
- [ ] Identified and inspected the bound PV.
- [ ] Read the test file from the mounted volume.
- [ ] Reviewed the reclaim policy and cleaned up the lab resources.

## Knowledge check

1. What is the difference between a PersistentVolume and a PersistentVolumeClaim?
2. What role does a StorageClass play in dynamic provisioning?
3. Why might a PVC remain `Pending` when its StorageClass uses `WaitForFirstConsumer`?
4. What does the `ReadWriteOnce` access mode mean?
5. What is the difference between the `Delete` and `Retain` reclaim policies?

## Key takeaways

- A StorageClass defines how Kubernetes provisions storage; a PVC requests storage from that class.
- AKS can dynamically provision Azure-backed storage through CSI drivers.
- A PVC must be bound to a PV before the requested storage is available to a Pod.
- `WaitForFirstConsumer` delays provisioning and binding until a Pod uses the PVC, allowing the scheduler to consider Pod placement.
- Always inspect the reclaim policy and clean up dynamically provisioned Azure resources after a lab.

## References

- [AKS storage concepts – Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- [Create and manage Azure Disk volumes in AKS – Microsoft Learn](https://learn.microsoft.com/en-us/azure/aks/create-volume-azure-disk)
- [StorageClasses – Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Persistent Volumes – Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
