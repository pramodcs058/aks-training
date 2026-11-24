# AKS Training Labs

A hands-on collection of Azure Kubernetes Service (AKS) labs covering cluster setup, Kubernetes workloads, configuration, networking, storage, scaling, security, monitoring, and troubleshooting.

The labs use Azure CLI, `kubectl`, Azure Cloud Shell, and Bash unless otherwise stated. Lab 0 is a PDF guide for cluster setup; Labs 1–24 provide Markdown instructions.

## Prerequisites

- An Azure subscription with access to an AKS cluster or permission to create one
- Azure CLI and `kubectl`, or Azure Cloud Shell
- Permission to create and delete the resources used by the labs
- Basic familiarity with Kubernetes concepts such as Pods, Deployments, Services, and Namespaces

> Some labs create Azure resources that may incur charges. Review the cleanup instructions and remove lab resources when finished.

## Getting Started

1. Open Azure Cloud Shell in Bash or use a terminal with Azure CLI and `kubectl`.
2. Sign in and select the training subscription:

   ```bash
   az login
   az account set --subscription "<SUBSCRIPTION_NAME_OR_ID>"
   ```

3. Start with [Lab 0 – AKS Cluster Setup](Lab-0-Steps-AKS-Cluster.pdf), [Lab 1 – Connect to an Existing AKS Cluster](Lab-1-Connect-to-Existing-AKS-Cluster.md), or [Lab 3 – Provision an AKS Cluster](Lab-3-Provision-AKS-Cluster-and-Configure-kubectl.md).
4. Follow each lab from its dedicated Markdown or PDF file and run cleanup when the lab is complete.

## Lab Guide

### Cluster setup and access

| Lab | Topic |
|---|---|
| [Lab 0 – AKS Cluster Setup](Lab-0-Steps-AKS-Cluster.pdf) | Prepare an AKS cluster for the training exercises. |
| [Lab 1 – Connect to an Existing AKS Cluster](Lab-1-Connect-to-Existing-AKS-Cluster.md) | Configure Azure CLI and `kubectl`, inspect contexts, nodes, namespaces, and system Pods. |
| [Lab 2 – Inspect and Scale AKS Node Pools](Lab-2-Inspect-and-Scale-AKS-Node-Pools.md) | Inspect and scale AKS node pools. |
| [Lab 3 – Provision an AKS Cluster and Configure kubectl](Lab-3-Provision-AKS-Cluster-and-Configure-kubectl.md) | Create an AKS cluster with Azure CLI and verify access with `kubectl`. |

### Workloads and configuration

| Lab | Topic |
|---|---|
| [Lab 4 – Deploy NGINX with ConfigMap Environment Variables](Lab-4-Deploy-NGINX-with-ConfigMap-Environment-Variables.md) | Inject environment variables into an NGINX deployment using a ConfigMap. |
| [Lab 5 – Deploy an Application with ConfigMap and Secret](Lab-5-Deploy-Application-with-ConfigMap-and-Secret.md) | Configure application settings and sensitive values with ConfigMaps and Secrets. |
| [Lab 6 – Deploy a Two-Tier Application with ConfigMap and Secret](Lab-6-Deploy-Two-Tier-Application-with-ConfigMap-and-Secret.md) | Build a multi-tier application with configuration and credential separation. |

### Networking and application exposure

| Lab | Topic |
|---|---|
| [Lab 7 – Deploy an Application Using a ClusterIP Service](Lab-7-Deploy-Application-Using-ClusterIP-Service.md) | Provide internal service discovery with ClusterIP. |
| [Lab 8 – Expose an Application with NodePort and LoadBalancer](Lab-8-Expose-Application-with-NodePort-and-LoadBalancer.md) | Expose workloads using NodePort and Azure Load Balancer services. |
| [Lab 9 – Expose an Application with NGINX Ingress and NetworkPolicy](Lab-9-Expose-Application-with-NGINX-Ingress-and-NetworkPolicy.md) | Route HTTP traffic through NGINX Ingress and apply network isolation. |
| [Lab 17 – Apply Default Deny and Allow NetworkPolicy](Lab-17-Apply-Default-Deny-and-Allow-NetworkPolicy-on-AKS.md) | Implement default-deny and targeted allow NetworkPolicies. |

### Storage

| Lab | Topic |
|---|---|
| [Lab 10 – Explore StorageClasses and Create PVC on AKS](Lab-10-Explore-StorageClasses-and-Create-PVC-on-AKS.md) | Inspect StorageClasses and create a PersistentVolumeClaim. |
| [Lab 11 – Configure Azure Files RWX Shared Storage](Lab-11-Configure-Azure-Files-RWX-Shared-Storage-on-AKS.md) | Provision shared ReadWriteMany storage with Azure Files. |
| [Lab 12 – Azure Disk PVC Data Persistence](Lab-12-Azure-Disk-PVC-Data-Persistence-on-AKS.md) | Use an Azure Disk PersistentVolumeClaim for workload data persistence. |

### Scaling

| Lab | Topic |
|---|---|
| [Lab 13 – Scale a Kubernetes Deployment Manually](Lab-13-Scale-Kubernetes-Deployment-Manually-on-AKS.md) | Scale workload replicas manually. |
| [Lab 14 – Configure HPA CPU Autoscaling](Lab-14-Configure-HPA-CPU-Autoscaling-on-AKS.md) | Configure CPU-based Horizontal Pod Autoscaling. |
| [Lab 15 – Observe HPA Scale-Out and Scale-In](Lab-15-Observe-HPA-Scale-Out-and-Scale-In-on-AKS.md) | Generate load and observe HPA scale-out and scale-in behavior. |

### Security

| Lab | Topic |
|---|---|
| [Lab 16 – Apply Pod Security Admission Baseline](Lab-16-Apply-Pod-Security-Admission-Baseline-on-AKS.md) | Apply the Pod Security Standards Baseline profile. |
| [Lab 18 – Deploy a Non-Root Pod with Restricted Pod Security](Lab-18-Deploy-Non-Root-Pod-with-Restricted-Pod-Security-on-AKS.md) | Deploy a restricted, non-root workload securely. |

### Monitoring and troubleshooting

| Lab | Topic |
|---|---|
| [Lab 19 – Basic AKS Workload Troubleshooting with kubectl](Lab-19-Basic-AKS-Workload-Troubleshooting-with-kubectl.md) | Diagnose workloads using `kubectl get`, `describe`, `logs`, and events. |
| [Lab 20 – Analyze AKS Logs with KQL in Log Analytics](Lab-20-Analyze-AKS-Logs-with-KQL-in-Log-Analytics.md) | Query AKS logs and workload data with Kusto Query Language. |
| [Lab 21 – Enable Container Insights and Troubleshoot with KQL](Lab-21-Enable-Container-Insights-and-Troubleshoot-Failing-Pod-with-KQL.md) | Enable Container Insights and use it to investigate failing Pods. |
| [Lab 22 – Troubleshoot ImagePullBackOff](Lab-22-Troubleshoot-ImagePullBackOff-on-AKS.md) | Diagnose and repair image pull failures. |
| [Lab 23 – Troubleshoot CrashLoopBackOff](Lab-23-Troubleshoot-CrashLoopBackOff-on-AKS.md) | Investigate container startup failures and repeated restarts. |
| [Lab 24 – Troubleshoot Three Broken Deployments](Lab-24-Troubleshoot-Three-Broken-Deployments-on-AKS.md) | Practice a troubleshooting capstone covering ImagePullBackOff, CrashLoopBackOff, and scheduling failure. |

## Common Validation Commands

```bash
az account show --output table
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods --all-namespaces
```

## Safety and Cleanup

- Confirm the active Azure subscription and `kubectl` context before applying or deleting resources.
- Use dedicated resource groups and namespaces for lab resources where instructed.
- Re-run a lab's cleanup commands when finished.
- Do not use lab credentials or test manifests in a production cluster.

## License

No license file is currently included in this repository. Check with the repository owner before redistributing or reusing the training material.
