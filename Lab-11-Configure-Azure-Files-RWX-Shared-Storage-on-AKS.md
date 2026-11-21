# Lab 11 – Configure Azure Files Shared Storage (RWX) for Multiple Pods on AKS

## Lab objective

Configure persistent shared storage on AKS using the **Azure Files CSI driver**. Mount one Azure Files-backed PersistentVolumeClaim (PVC) into two application Pods and verify that data written by one Pod is accessible from the other.

## Learning outcomes

By the end of this lab, participants will be able to:

- Explain why Azure Files is suitable for shared storage across multiple Pods.
- Identify the Azure Files CSI StorageClass available in an AKS cluster.
- Create a `ReadWriteMany` (RWX) PVC using dynamic provisioning.
- Mount the same PVC into two Pods.
- Verify shared file access from both Pods.

## Concepts

| Concept | Explanation |
|---|---|
| **Azure Files** | A managed file share service in Azure. AKS Pods can mount the same share and access shared files. |
| **Azure Files CSI driver** | The Kubernetes storage driver (`file.csi.azure.com`) that provisions and mounts Azure file shares for Pods. |
| **PersistentVolume (PV)** | A cluster storage resource backed in this lab by an Azure file share. |
| **PersistentVolumeClaim (PVC)** | A workload's request for storage. Kubernetes binds it to a suitable PV. |
| **ReadWriteMany (RWX)** | An access mode that allows a volume to be mounted read/write by multiple nodes. Azure Files supports shared access by multiple Pods. |

## Prerequisites

- An existing, running AKS cluster.
- Access to the AKS cluster with permissions to create Deployments, Pods and PVCs.
- Azure Files CSI driver enabled on the AKS cluster. AKS clusters normally have the Azure Files CSI driver available.
- Azure Portal access and an active **Cloud Shell – Bash** session.
- The cluster nodes must be able to reach the Azure Files service endpoints. Network restrictions, including blocked SMB connectivity, can prevent mounts.

> **Lab environment:** All commands in this lab are run from Azure Portal Cloud Shell using Bash. Replace the sample resource group and cluster values with your AKS details.

---

## Step 1 – Connect Cloud Shell to the AKS cluster

Set the AKS resource group, cluster name and subscription. Then retrieve the cluster credentials.

```bash
RESOURCE_GROUP="<your-aks-resource-group>"
AKS_CLUSTER="<your-aks-cluster-name>"
SUBSCRIPTION_ID="<your-subscription-id>"

az account set --subscription "$SUBSCRIPTION_ID"
az aks get-credentials --resource-group "$RESOURCE_GROUP" --name "$AKS_CLUSTER"
kubectl get nodes
```

**Explanation:** `az aks get-credentials` downloads the cluster access details and merges them into the Cloud Shell kubeconfig. `kubectl get nodes` confirms that the current context can communicate with the cluster.

## Step 2 – Verify the Azure Files CSI driver and StorageClass

List the StorageClasses and confirm that the Azure Files CSI driver is available.

```bash
kubectl get storageclass
kubectl get csidriver file.csi.azure.com
kubectl get storageclass azurefile-csi -o wide
```

**Explanation:** The AKS built-in `azurefile-csi` StorageClass provisions a Standard Azure Files share. The `azurefile-csi-premium` class is available for Premium Azure Files in supported configurations. This lab uses `azurefile-csi` to keep the exercise cost-conscious.

Set the StorageClass name used by the PVC:

```bash
STORAGE_CLASS="azurefile-csi"
kubectl get storageclass "$STORAGE_CLASS"
```

> **If `azurefile-csi` is not present:** Run `kubectl get storageclass -o custom-columns=NAME:.metadata.name,PROVISIONER:.provisioner`. Select an existing StorageClass whose provisioner is `file.csi.azure.com`, then update `STORAGE_CLASS` to that name. Do not continue with a disk StorageClass such as `managed-csi`; Azure Disk is not the shared file storage used in this lab.

## Step 3 – Create a namespace

```bash
NAMESPACE="azurefiles-rwx-lab"
kubectl create namespace "$NAMESPACE"
```

**Explanation:** A namespace keeps the lab resources separate from other workloads in the cluster.

## Step 4 – Create a ReadWriteMany PVC

Create the PVC manifest. It requests 5 GiB of Azure Files storage and uses the Azure Files CSI StorageClass.

```bash
cat > azurefiles-pvc.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files-pvc
  namespace: ${NAMESPACE}
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ${STORAGE_CLASS}
  resources:
    requests:
      storage: 5Gi
EOF
```

Apply the manifest and inspect the claim:

```bash
kubectl apply -f azurefiles-pvc.yaml
kubectl get pvc -n "$NAMESPACE"
kubectl describe pvc shared-files-pvc -n "$NAMESPACE"
```

**Explanation:** The PVC asks Kubernetes for a 5 GiB volume with `ReadWriteMany` access. The Azure Files CSI driver dynamically provisions the underlying Azure file share and the corresponding PV.

> **Expected status:** The PVC should become `Bound`. If it remains `Pending`, inspect `kubectl describe pvc` and the Events section before proceeding.

## Step 5 – Deploy two Pods that mount the same PVC

Create a Deployment with two BusyBox Pods. Both replicas mount the same PVC at `/mnt/azurefiles`.

```bash
cat > azurefiles-app.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azurefiles-demo
  namespace: ${NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: azurefiles-demo
  template:
    metadata:
      labels:
        app: azurefiles-demo
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: file-client
          image: busybox:1.36
          command: ["/bin/sh", "-c", "while true; do sleep 3600; done"]
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 250m
              memory: 128Mi
          volumeMounts:
            - name: shared-files
              mountPath: /mnt/azurefiles
      volumes:
        - name: shared-files
          persistentVolumeClaim:
            claimName: shared-files-pvc
EOF
```

Apply the Deployment and wait for both replicas to become ready:

```bash
kubectl apply -f azurefiles-app.yaml
kubectl rollout status deployment/azurefiles-demo -n "$NAMESPACE" --timeout=180s
kubectl get pods -n "$NAMESPACE" -o wide
```

**Explanation:** The Deployment creates two application Pods. Each Pod references the same PVC, so both containers mount the same Azure file share.

## Step 6 – Identify the two Pods

Retrieve the names of the two running Pods:

```bash
POD_A=$(kubectl get pods -n "$NAMESPACE" -l app=azurefiles-demo -o jsonpath='{.items[0].metadata.name}')
POD_B=$(kubectl get pods -n "$NAMESPACE" -l app=azurefiles-demo -o jsonpath='{.items[1].metadata.name}')

echo "Pod A: $POD_A"
echo "Pod B: $POD_B"
```

Check that both Pods are `Running` and `Ready` before testing shared access:

```bash
kubectl get pods -n "$NAMESPACE" -l app=azurefiles-demo
```

## Step 7 – Write a file from Pod A

Create a file in the mounted Azure Files directory from the first Pod:

```bash
kubectl exec -n "$NAMESPACE" "$POD_A" -- sh -c 'echo "File created by Pod A: $HOSTNAME" > /mnt/azurefiles/shared-check.txt'

kubectl exec -n "$NAMESPACE" "$POD_A" -- ls -l /mnt/azurefiles
kubectl exec -n "$NAMESPACE" "$POD_A" -- cat /mnt/azurefiles/shared-check.txt
```

**Explanation:** The file is written to the Azure Files share mounted at `/mnt/azurefiles`, not to the container's temporary writable layer.

## Step 8 – Read the same file from Pod B

Read the file from the second Pod:

```bash
kubectl exec -n "$NAMESPACE" "$POD_B" -- ls -l /mnt/azurefiles
kubectl exec -n "$NAMESPACE" "$POD_B" -- cat /mnt/azurefiles/shared-check.txt
```

**Expected result:** Pod B displays the content written by Pod A. This confirms that both Pods can access the same shared Azure file share.

## Step 9 – Write from Pod B and read from Pod A

Append another line from Pod B:

```bash
kubectl exec -n "$NAMESPACE" "$POD_B" -- sh -c 'echo "Second write from Pod B: $HOSTNAME" >> /mnt/azurefiles/shared-check.txt'
```

Read the updated file from Pod A:

```bash
kubectl exec -n "$NAMESPACE" "$POD_A" -- cat /mnt/azurefiles/shared-check.txt
```

**Expected result:** The file contains entries written by both Pods. This demonstrates read/write access to the shared volume from multiple Pods.

## Step 10 – Inspect the PVC and dynamically provisioned PV

```bash
kubectl get pvc shared-files-pvc -n "$NAMESPACE"
kubectl get pv

PV_NAME=$(kubectl get pvc shared-files-pvc -n "$NAMESPACE" -o jsonpath='{.spec.volumeName}')
echo "Provisioned PV: $PV_NAME"
kubectl describe pv "$PV_NAME"
```

**Explanation:** The PVC should show `Bound` and `RWX`. The PV details identify the CSI provisioner and the storage resource associated with the claim.

---

## Expected outcome

| Validation | Expected result |
|---|---|
| Azure Files CSI driver | `file.csi.azure.com` is listed as an available CSI driver. |
| PVC | `shared-files-pvc` reaches `Bound` status. |
| Access mode | PVC displays `RWX` / `ReadWriteMany`. |
| Pods | Two `azurefiles-demo` Pods reach `Running` and `Ready`. |
| Pod A write | A file is created in `/mnt/azurefiles`. |
| Pod B read | Pod B reads the same content written by Pod A. |
| Shared write | A line appended by Pod B is visible when Pod A reads the file. |

## Troubleshooting

| Issue | What to check |
|---|---|
| PVC remains `Pending` | Run `kubectl describe pvc shared-files-pvc -n "$NAMESPACE"` and review Events. Confirm the StorageClass exists and uses `file.csi.azure.com`. |
| Pod remains `ContainerCreating` | Run `kubectl describe pod <pod-name> -n "$NAMESPACE"`. Review mount errors and CSI events. |
| Mount fails with network errors | Confirm AKS nodes can reach the Azure Storage endpoint. Check storage account firewall, private endpoint, DNS and required network connectivity. |
| Permission denied when writing | Inspect the Azure Files mount options and the container's user ID. The default StorageClass permissions may not match a non-root application. |
| Pod B cannot see Pod A's file | Confirm both Pods reference `shared-files-pvc` and both mount it at `/mnt/azurefiles`. Check that the file was written successfully. |
| `file.csi.azure.com` StorageClass is not found | Use an existing StorageClass with provisioner `file.csi.azure.com`, or ask the cluster administrator to verify Azure Files CSI support. |

Useful diagnostic commands:

```bash
kubectl describe pvc shared-files-pvc -n "$NAMESPACE"
kubectl describe pods -n "$NAMESPACE" -l app=azurefiles-demo
kubectl get events -n "$NAMESPACE" --sort-by=.metadata.creationTimestamp
kubectl get csidriver file.csi.azure.com
```

## Cleanup

Delete the Deployment first so that the Pods release the shared volume. Then delete the PVC.

```bash
kubectl delete deployment azurefiles-demo -n "$NAMESPACE" --wait=true
kubectl delete pvc shared-files-pvc -n "$NAMESPACE" --wait=true
kubectl get pv
kubectl delete namespace "$NAMESPACE"
```

> **Storage-cost note:** The built-in `azurefile-csi` StorageClass normally uses a `Delete` reclaim policy. Deleting the PVC causes the dynamically provisioned Azure file share to be deleted with its PV. Confirm the PV reclaim policy before cleanup if you selected a different StorageClass. The storage account itself may remain in the AKS node resource group.

## Knowledge check

1. Why is Azure Files a suitable storage option when multiple Pods need read/write access to the same files?
2. What does `ReadWriteMany` mean, and how is it different from `ReadWriteOnce`?
3. What is the role of the Azure Files CSI driver?
4. What is the relationship between a StorageClass, PVC and PV?
5. Why can both Pods read a file written by the other Pod?
6. What would you inspect first if the PVC remains in `Pending` status?

## Key takeaway

**A single Azure Files-backed PVC with `ReadWriteMany` access can be mounted by multiple AKS Pods.** The Azure Files CSI driver handles dynamic provisioning and mounting, allowing Pods to share files without placing the data inside individual container file systems.

## References

- [Microsoft Learn – Create and manage persistent volumes with Azure Files in AKS](https://learn.microsoft.com/azure/aks/create-volume-azure-files)
- [Microsoft Learn – CSI storage drivers in AKS](https://learn.microsoft.com/azure/aks/csi-storage-drivers)
- [Kubernetes – Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes – Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
