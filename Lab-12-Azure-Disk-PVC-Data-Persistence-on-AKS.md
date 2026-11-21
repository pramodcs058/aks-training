# Lab 12 – Create an Azure Disk PVC and Verify Data Persistence on AKS

## Lab information

| Item | Details |
|---|---|
| **Module** | Module 4 – Storage on AKS |
| **Lab type** | Existing TOC Lab |
| **Duration** | As scheduled in the training plan |
| **Execution environment** | Azure Portal → Cloud Shell → Bash |
| **Storage driver** | Azure Disk CSI (`disk.csi.azure.com`) |
| **StorageClass** | `managed-csi` |
| **Access mode** | `ReadWriteOnce` (RWO) |

## Objective

Create an Azure Disk-backed PersistentVolumeClaim (PVC), mount it to an NGINX Pod, write sample data, delete the Pod, and recreate it using the same PVC. Verify that the data remains available after the Pod is recreated.

## Expected outcome

At the end of this lab, participants will be able to:

- Identify the Azure Disk CSI StorageClass available in AKS.
- Create a PVC that dynamically provisions an Azure Managed Disk.
- Mount the PVC into an application Pod.
- Verify the PersistentVolume (PV) and PVC binding.
- Demonstrate that data survives Pod deletion and recreation when the same PVC is reused.

## Concepts

| Concept | Explanation |
|---|---|
| **PersistentVolume (PV)** | A Kubernetes resource representing storage provisioned for use by workloads. In this lab, the PV represents an Azure Managed Disk. |
| **PersistentVolumeClaim (PVC)** | A workload's request for persistent storage, including its size and access mode. Kubernetes binds it to a suitable PV. |
| **StorageClass** | Defines how Kubernetes dynamically provisions storage. AKS provides CSI-backed StorageClasses such as `managed-csi`. |
| **Azure Disk CSI driver** | The Container Storage Interface driver that provisions, attaches and mounts Azure Managed Disks for AKS workloads. |
| **ReadWriteOnce (RWO)** | Allows a volume to be mounted read-write by one node at a time. It does not mean that only one Pod on that node can ever use the volume. This lab uses one Pod at a time. |
| **Dynamic provisioning** | Kubernetes creates the backing Azure Disk automatically when a PVC requests storage through a suitable StorageClass. |

> **Important:** Azure Disk is block storage and is normally used for workloads that need a disk attached to one node at a time. For simultaneous shared access from Pods across multiple nodes, use Azure Files with `ReadWriteMany` (RWX).

---

## Prerequisites

- An existing, running AKS cluster.
- Access to the Azure Portal and its Cloud Shell.
- Cloud Shell opened in **Bash** mode.
- `kubectl` configured to access the target AKS cluster.
- Permission to create Kubernetes namespaces, PVCs and Pods.
- Sufficient Azure quota and permissions for the AKS cluster to provision and attach a managed disk.

## Lab architecture

```text
                 AKS Cluster
                      |
                NGINX Pod
                      |
             Mount: /usr/share/nginx/html/persistent
                      |
             PersistentVolumeClaim
                 (4 Gi, RWO)
                      |
                PersistentVolume
                      |
             Azure Managed Disk
             (Azure Disk CSI)
```

The PVC is retained when the Pod is deleted. The replacement Pod mounts the same PVC, which reconnects it to the same persistent disk and its existing data.

---

## Step 1 – Open Cloud Shell and connect to AKS

1. Sign in to the [Azure Portal](https://portal.azure.com/).
2. Open **Cloud Shell** and select **Bash**.
3. Select the subscription and connect to the AKS cluster if Cloud Shell is not already configured for it.

```bash
az account show --output table
```

Displays the active Azure subscription and account context.

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

Confirms that `kubectl` can reach the Kubernetes API server and that the cluster has available nodes.

## Step 2 – Inspect the Azure Disk StorageClass

```bash
kubectl get storageclass
```

Lists the StorageClasses available in the AKS cluster.

```bash
kubectl get storageclass managed-csi -o yaml
```

Displays the configuration of the standard AKS Azure Disk CSI StorageClass, including its provisioner, binding mode and reclaim policy.

```bash
kubectl get storageclass managed-csi \
  -o jsonpath='Provisioner: {.provisioner}{"\n"}Binding mode: {.volumeBindingMode}{"\n"}Reclaim policy: {.reclaimPolicy}{"\n"}'
```

Confirms that the provisioner is `disk.csi.azure.com` and displays the StorageClass binding and reclaim settings.

> **Check before continuing:** This lab uses the AKS `managed-csi` StorageClass. If it is not present, run `kubectl get storageclass -o wide` and select an existing StorageClass whose provisioner is `disk.csi.azure.com`. Use that StorageClass name in Step 3. Do not create a new StorageClass or change a cluster-wide StorageClass as part of this lab.

## Step 3 – Create a namespace

```bash
kubectl create namespace aks-azuredisk-lab
```

Creates an isolated namespace for the lab resources.

```bash
kubectl get namespace aks-azuredisk-lab
```

Verifies that the namespace exists.

## Step 4 – Create the PersistentVolumeClaim

Create a manifest file:

```bash
cat > azure-disk-pvc.yaml <<'YAML'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azure-disk-pvc
  namespace: aks-azuredisk-lab
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 4Gi
YAML
```

The PVC requests a 4 GiB Azure Disk using the `managed-csi` StorageClass and `ReadWriteOnce` access mode.

```bash
kubectl apply -f azure-disk-pvc.yaml
```

Creates the PVC. Kubernetes uses the selected StorageClass to arrange dynamic provisioning.

```bash
kubectl get pvc -n aks-azuredisk-lab
```

Displays the PVC status. It may show `Pending` until a Pod uses it when the StorageClass uses `WaitForFirstConsumer`.

> **Why might the PVC remain Pending?** `WaitForFirstConsumer` delays volume provisioning and binding until Kubernetes knows which node will run a Pod that uses the claim. This allows the disk to be provisioned with the correct topology, such as an availability zone.

## Step 5 – Create an application Pod that mounts the PVC

Create the Pod manifest:

```bash
cat > azure-disk-pod.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: azure-disk-app
  namespace: aks-azuredisk-lab
  labels:
    app: azure-disk-app
spec:
  containers:
    - name: nginx
      image: nginx:stable-alpine
      ports:
        - name: http
          containerPort: 80
      volumeMounts:
        - name: persistent-storage
          mountPath: /usr/share/nginx/html/persistent
  volumes:
    - name: persistent-storage
      persistentVolumeClaim:
        claimName: azure-disk-pvc
YAML
```

Defines an NGINX Pod and mounts the PVC at `/usr/share/nginx/html/persistent`. Files written to this directory are stored on the Azure Disk rather than only in the container's writable layer.

```bash
kubectl apply -f azure-disk-pod.yaml
```

Creates the NGINX Pod and triggers storage provisioning if it has not happened already.

```bash
kubectl wait --for=condition=Ready pod/azure-disk-app \
  -n aks-azuredisk-lab --timeout=300s
```

Waits for the Pod to become Ready. Azure Disk provisioning and attachment can take a few minutes.

## Step 6 – Verify the PVC and PV binding

```bash
kubectl get pvc -n aks-azuredisk-lab
```

Confirms that `azure-disk-pvc` reaches the `Bound` state.

```bash
kubectl get pv
```

Lists the PersistentVolumes in the cluster. Identify the PV bound to `azure-disk-pvc`.

```bash
PV_NAME=$(kubectl get pvc azure-disk-pvc \
  -n aks-azuredisk-lab -o jsonpath='{.spec.volumeName}')

echo "$PV_NAME"
kubectl get pv "$PV_NAME" -o wide
```

Retrieves the PV name from the PVC and displays the PV details, including its capacity, access mode, reclaim policy and claim reference.

```bash
kubectl describe pod azure-disk-app -n aks-azuredisk-lab
```

Displays the Pod's volume configuration and recent events, which can help identify provisioning, scheduling or mount issues.

## Step 7 – Write sample data to the mounted disk

```bash
kubectl exec azure-disk-app -n aks-azuredisk-lab -- \
  sh -c 'printf "AKS Azure Disk persistence test - created before Pod deletion.\n" > /usr/share/nginx/html/persistent/aks-persistence.txt'
```

Writes a sample file into the mounted Azure Disk volume from inside the NGINX container.

```bash
kubectl exec azure-disk-app -n aks-azuredisk-lab -- \
  cat /usr/share/nginx/html/persistent/aks-persistence.txt
```

Reads the file from the mounted path and confirms that the sample data was written successfully.

Optional: verify that NGINX can serve the file over HTTP from inside the container.

```bash
kubectl exec azure-disk-app -n aks-azuredisk-lab -- \
  wget -qO- http://127.0.0.1/persistent/aks-persistence.txt
```

Requests the file through the local NGINX process inside the Pod. The response should contain the sample text.

## Step 8 – Record the Pod identity and delete the Pod

```bash
OLD_POD_UID=$(kubectl get pod azure-disk-app \
  -n aks-azuredisk-lab -o jsonpath='{.metadata.uid}')

echo "Original Pod UID: $OLD_POD_UID"
```

Records the current Pod UID so it can be compared with the replacement Pod.

```bash
kubectl delete pod azure-disk-app -n aks-azuredisk-lab
```

Deletes the application Pod. This does **not** delete the PVC, PV or its data.

```bash
kubectl get pvc -n aks-azuredisk-lab
kubectl get pv "$PV_NAME" -o wide
```

Verifies that the PVC still exists and that its PV remains associated with the claim even though the Pod has been deleted.

## Step 9 – Recreate the Pod using the same PVC

```bash
kubectl apply -f azure-disk-pod.yaml
```

Recreates the Pod from the same manifest, which references the existing `azure-disk-pvc` claim.

```bash
kubectl wait --for=condition=Ready pod/azure-disk-app \
  -n aks-azuredisk-lab --timeout=300s
```

Waits for Kubernetes to schedule the replacement Pod and for the Azure Disk to attach and mount successfully.

```bash
NEW_POD_UID=$(kubectl get pod azure-disk-app \
  -n aks-azuredisk-lab -o jsonpath='{.metadata.uid}')

echo "Original Pod UID:  $OLD_POD_UID"
echo "New Pod UID:      $NEW_POD_UID"
```

Shows that the replacement Pod has a different UID, confirming that Kubernetes created a new Pod instance.

## Step 10 – Verify data persistence

```bash
kubectl exec azure-disk-app -n aks-azuredisk-lab -- \
  cat /usr/share/nginx/html/persistent/aks-persistence.txt
```

Reads the file from the replacement Pod. The original sample text should still be present, demonstrating that the data was stored on the persistent Azure Disk and not only in the previous container's writable layer.

```bash
kubectl get pvc -n aks-azuredisk-lab
kubectl get pv "$PV_NAME" -o wide
kubectl get pod azure-disk-app -n aks-azuredisk-lab -o wide
```

Confirms that the PVC is still `Bound`, the same PV is in use, and the replacement Pod is running.

---

## Expected observations

| Check | Expected result |
|---|---|
| StorageClass provisioner | `disk.csi.azure.com` |
| PVC status after Pod scheduling | `Bound` |
| PV status | `Bound` |
| Pod status | `Running` / `Ready` |
| File before Pod deletion | Sample text is readable |
| Pod UID after recreation | Different from the original UID |
| File after Pod recreation | Original sample text is still readable |

## Troubleshooting

| Issue | What to check |
|---|---|
| PVC remains `Pending` | Check whether the StorageClass uses `WaitForFirstConsumer`; ensure the Pod that references the PVC has been created. Run `kubectl describe pvc azure-disk-pvc -n aks-azuredisk-lab`. |
| Pod remains `Pending` | Run `kubectl describe pod azure-disk-app -n aks-azuredisk-lab` and inspect scheduling events, node resources, topology and disk quota. |
| Pod shows `ContainerCreating` for a long time | Inspect Pod events and PVC events. Azure Disk provisioning or attachment may still be in progress. |
| Mount or attach failure | Run `kubectl describe pod azure-disk-app -n aks-azuredisk-lab` and review the events for CSI, disk, zone or permission errors. |
| Pod recreation takes longer than expected | Wait for the old Pod to terminate and the disk to detach before expecting the replacement Pod to become Ready. Inspect `kubectl get pods -o wide` and Pod events. |
| Sample file is missing | Confirm that the Pod references `azure-disk-pvc` and that the file was written under `/usr/share/nginx/html/persistent`, the mounted volume path. |
| `managed-csi` does not exist | Run `kubectl get storageclass -o wide` and use an existing StorageClass whose provisioner is `disk.csi.azure.com`. Update `storageClassName` in the PVC manifest accordingly. |

Useful diagnostic commands:

```bash
kubectl describe pvc azure-disk-pvc -n aks-azuredisk-lab
kubectl describe pod azure-disk-app -n aks-azuredisk-lab
kubectl get events -n aks-azuredisk-lab --sort-by='.lastTimestamp'
kubectl get pvc -n aks-azuredisk-lab -o wide
kubectl get pv
kubectl get pods -n aks-azuredisk-lab -o wide
```

## Cleanup

> **Cost and data warning:** The `managed-csi` StorageClass normally uses a `Delete` reclaim policy. Deleting the PVC (including by deleting its namespace) can delete the associated PV and Azure Managed Disk. Confirm the reclaim policy before cleanup. Deleting the PVC permanently removes the lab data when the reclaim policy is `Delete`.

```bash
STORAGE_CLASS=$(kubectl get pvc azure-disk-pvc \
  -n aks-azuredisk-lab -o jsonpath='{.spec.storageClassName}')

kubectl get storageclass "$STORAGE_CLASS" \
  -o jsonpath='StorageClass: {.metadata.name}{"\n"}Reclaim policy: {.reclaimPolicy}{"\n"}'
```

After completing the lab and confirming that the data is no longer needed, remove the namespace:

```bash
kubectl delete namespace aks-azuredisk-lab
```

If the selected StorageClass uses `Retain`, the Azure Disk may remain after the PVC and namespace are deleted. Check for a released PV and arrange cleanup of the retained disk through the appropriate Azure administration process. Do not manually delete a disk that is still attached to or referenced by an active PV.

## Knowledge check

1. What is the difference between a PersistentVolume and a PersistentVolumeClaim?
2. Which CSI provisioner is used by the Azure Disk driver?
3. What does `ReadWriteOnce` mean?
4. Why might a PVC using `WaitForFirstConsumer` remain in `Pending` until a Pod is scheduled?
5. Why does deleting a Pod not delete the data stored in its PVC?
6. What happens to the underlying Azure Disk when a PVC is deleted under a `Delete` reclaim policy?
7. Which Azure storage service would you choose when multiple Pods on different nodes need shared RWX access?

## Key takeaways

- A PVC requests persistent storage; a PV represents the provisioned storage resource.
- The Azure Disk CSI driver dynamically provisions and attaches Azure Managed Disks for AKS workloads.
- A Pod accesses the disk through a PVC mounted at a path inside its container.
- Deleting and recreating a Pod does not remove data held by the PVC, provided the same claim and PV are reused.
- `ReadWriteOnce` is intended for read-write mounting by one node at a time. Use Azure Files when shared RWX access across nodes is required.
- Always review the StorageClass reclaim policy and clean up unused disks to avoid unnecessary Azure charges.

## References

- [Microsoft Learn – Create and manage persistent volumes with Azure Disks in AKS](https://learn.microsoft.com/en-us/azure/aks/create-volume-azure-disk)
- [Microsoft Learn – Storage concepts in AKS](https://learn.microsoft.com/en-us/azure/aks/concepts-storage)
- [Microsoft Learn – CSI storage drivers in AKS](https://learn.microsoft.com/en-us/azure/aks/csi-storage-drivers)
- [Kubernetes documentation – Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
