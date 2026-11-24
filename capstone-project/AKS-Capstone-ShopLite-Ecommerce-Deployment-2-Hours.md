# AKS Capstone Project – Deploy and Operate a Secure E-Commerce Application
 
**Platform:** Azure Kubernetes Service (AKS)  
**Execution environment:** Azure Portal → Cloud Shell → Bash  
**Project type:** Individual or small-team hands-on capstone

---

## 1. Case Study: ShopLite – An Online Retail Platform

ShopLite is a growing online retailer that wants to move its e-commerce application from a single-server environment to Azure Kubernetes Service (AKS). The business expects variable customer traffic during promotional campaigns and needs its application to remain available, secure and observable.

The current application consists of a web application tier and a database tier. The web application serves the online store, handles customer requests and connects to a PostgreSQL database that stores product, inventory and order information.

The existing deployment has several operational limitations:

- Application settings and database credentials are embedded in configuration files.
- The database does not have a persistent storage configuration, creating a risk of data loss when Pods are replaced.
- The application is not exposed through a controlled HTTP entry point.
- Workload-to-workload communication is not restricted.
- Application Pods do not have consistent security settings.
- The platform team cannot easily scale the web tier during traffic spikes.
- Logs, metrics and alerts are not being used consistently to identify application issues.

You have been assigned as the **AKS implementation engineer**. Your responsibility is to deploy a secure, observable and scalable version of ShopLite on an existing AKS cluster and demonstrate that it can handle a basic operational scenario.

## 2. Project Objective

Design, deploy and validate a two-tier e-commerce application on AKS. The solution must use Kubernetes-native configuration, Services, persistent storage, Ingress, autoscaling, network isolation, Pod security and Azure Monitor capabilities.

At the end of the exercise, a user should be able to reach the ShopLite web application through an Ingress endpoint, while the application communicates with the database internally through a Kubernetes Service.

## 3. Application Architecture to Build

Implement the following logical architecture. The trainer will provide the sample application images and starter application files required for the two tiers.

```text
                     Customer / Browser
                             |
                             | HTTP / HTTPS
                             v
                    Ingress Controller
                             |
                             v
                  ShopLite Web Application
                    Deployment + Service
                             |
                Internal application traffic
                             |
                             v
                   PostgreSQL Database
                   StatefulSet + Service
                             |
                             v
                   Azure Disk PersistentVolume

        Supporting AKS capabilities:
        - ConfigMap and Kubernetes Secret
        - NetworkPolicy
        - Pod Security Admission / securityContext
        - Horizontal Pod Autoscaler (HPA)
        - Azure Monitor Container Insights
        - Log Analytics and KQL
        - Azure Monitor alert
```

**Design requirement:** The web tier must be exposed to users through an Ingress resource. The database must remain internal and must not be exposed using a public LoadBalancer or NodePort Service.

## 4. Business and Technical Requirements

### A. AKS access and cluster inspection

1. Connect to the existing AKS cluster from Azure Portal Cloud Shell using Bash.
2. Verify the active subscription and Kubernetes context.
3. Inspect the cluster nodes, namespaces and node-pool information.
4. Identify the cluster's network configuration and note whether a supported NetworkPolicy engine is available.

**Expected result:** Participants can access the cluster and identify the environment in which ShopLite will run.

### B. Namespace and application configuration

1. Create a dedicated namespace named `shoplite`.
2. Apply the Pod Security Admission `restricted` enforcement label to the namespace, subject to the supplied application image supporting the Restricted requirements.
3. Create a ConfigMap for non-sensitive application settings, such as application mode, database host, database name and application port.
4. Create an Opaque Secret for the database username and password.
5. Reference the ConfigMap and Secret from the application workloads. Do not hardcode credentials in the container image or commit clear-text credentials to the project repository.

**Expected result:** ShopLite runs in an isolated namespace and obtains configuration from Kubernetes resources rather than from hardcoded image values.

### C. Database deployment and persistent storage

1. Inspect the available StorageClasses and identify the Azure Disk CSI StorageClass supplied by the cluster.
2. Create a PersistentVolumeClaim (PVC) for the PostgreSQL data directory using `ReadWriteOnce` access.
3. Deploy PostgreSQL as a StatefulSet (or use the trainer-provided database workload manifest) and mount the PVC at the database data path.
4. Expose PostgreSQL internally using a ClusterIP Service.
5. Verify that the PVC is `Bound` and that the database Pod is Ready.
6. Create a sample product or order record using the application-provided initialization process or database test command.

**Expected result:** The database is reachable only within the cluster and stores its data on an Azure Disk-backed persistent volume. The data must not depend on the lifetime of an individual Pod.

### D. Web application deployment and Services

1. Deploy the ShopLite web application using the image supplied by the trainer.
2. Configure the web workload to read its application settings from the ConfigMap and database credentials from the Secret.
3. Define CPU and memory requests and limits for the web Pods. CPU requests are required for CPU-based HPA calculations.
4. Create a ClusterIP Service for the web application.
5. Verify that the web application can resolve and connect to the database by using the database Service DNS name.

**Expected result:** The web and database tiers communicate through Kubernetes Service discovery, without relying on Pod IP addresses.

### E. Ingress and application exposure

1. Create an Ingress resource that routes external HTTP/HTTPS requests to the ShopLite web Service.
2. Use the Ingress controller already installed in the training cluster. Configure the host name or test endpoint according to the trainer's environment.
3. Verify that the application landing page or product catalogue is accessible through the Ingress endpoint.
4. Document why the database Service remains `ClusterIP` and why the web application uses Ingress instead of exposing every Pod directly.

**Expected result:** Users can access ShopLite through one controlled entry point while the database remains private to the cluster.

### F. Network isolation

1. Apply a default-deny ingress NetworkPolicy in the `shoplite` namespace.
2. Add an allow rule for traffic from the web application Pods to the database Pods on the PostgreSQL TCP port (`5432`).
3. Add the necessary ingress-controller-to-web rule, using the controller namespace and Pod labels supplied by the trainer.
4. Deploy or use a temporary test/debug Pod to verify that an unauthorized workload cannot connect to the database.
5. Verify that the ShopLite web tier can still connect to the database and that the test/debug Pod is denied.

**Expected result:** Only explicitly permitted application traffic reaches the database. NetworkPolicy enforcement must be supported by the AKS cluster's network-policy engine.

### G. Pod and workload security

1. Ensure the application Pods comply with the namespace's Restricted Pod Security requirements.
2. Configure the applicable `securityContext` settings, including non-root execution, disabled privilege escalation, dropped Linux capabilities and `seccompProfile: RuntimeDefault`.
3. Use a read-only root filesystem where the supplied image supports it. Mount explicitly required writable paths using an appropriate volume such as `emptyDir`.
4. Confirm that an intentionally non-compliant test Pod is rejected by Pod Security Admission.

**Expected result:** The application runs with reduced container privileges and the namespace rejects workloads that violate its enforced Pod Security Standard.

### H. Autoscaling and workload validation

1. Configure an HPA for the ShopLite web Deployment using CPU utilization as the scaling metric.
2. Set a minimum of **1 replica**, a maximum of **3 replicas**, and a target CPU utilization appropriate to the supplied workload (for example, **60%**).
3. Generate HTTP load using the trainer-provided load command or an in-cluster test client.
4. Observe the HPA metrics and the web Deployment's desired and available replica counts.
5. Stop the load and observe whether the replica count scales down, subject to HPA metric collection and stabilization behaviour.
6. Inspect the node-pool autoscaling configuration. Do not change cluster-level autoscaler settings during this time-boxed exercise.

**Expected result:** The web tier can scale horizontally in response to CPU demand, and participants can explain the difference between Pod autoscaling and node-pool autoscaling.

### I. Monitoring, KQL and alerting

1. Verify that Azure Monitor Container Insights is enabled for the AKS cluster and connected to the supplied Log Analytics workspace.
2. Confirm that ShopLite Pod inventory, container logs and Kubernetes events are being ingested.
3. Use KQL to investigate at least:
   - ShopLite Pod status and restart counts (`KubePodInventory`).
   - Application container logs (`ContainerLogV2`).
   - Kubernetes warning events (`KubeEvents`).
4. Create or validate one Azure Monitor alert using the alert rule/action group prepared for the training environment. The alert should identify an application health signal such as sustained high CPU or repeated Pod restarts.
5. Capture the query result or alert configuration as evidence.

**Expected result:** Participants can use centralized monitoring data to identify workload behaviour and demonstrate an alert for an application-health condition.

### J. Troubleshooting and operational readiness

1. Use `kubectl get`, `kubectl describe`, `kubectl logs` and `kubectl get events` to validate the application and investigate any deployment issue.
2. Demonstrate one controlled recovery action, such as correcting a configuration value or fixing a deliberately incorrect image reference supplied by the trainer.
3. Verify that the web application is available, the database remains healthy and the stored test record is still present.
4. Record basic production-readiness recommendations for node-pool sizing, availability zones, Pod Disruption Budgets, upgrade strategy, maintenance windows, backup/restore and cost management.

**Expected result:** Participants demonstrate a repeatable troubleshooting process and provide an operational handover for the deployed application.

## 5. End Product Expected from Participants

At the end of the two-hour capstone, participants are expected to deliver a working **ShopLite e-commerce application deployed on AKS**, together with the Kubernetes configuration and operational evidence.

### Deployed application

- A ShopLite web application accessible through the Ingress endpoint.
- A PostgreSQL database running inside the AKS cluster.
- A ClusterIP Service for internal database access.
- A persistent Azure Disk PVC attached to the database workload.
- A successful web-to-database connectivity check and at least one sample record stored in the database.

### Kubernetes resources

Participants should have created or configured the following resources in the `shoplite` namespace:

- ConfigMap and Opaque Secret.
- Web application Deployment and ClusterIP Service.
- Database StatefulSet (or trainer-provided database workload) and ClusterIP Service.
- Azure Disk PersistentVolumeClaim.
- Ingress resource.
- Default-deny and allow NetworkPolicies.
- HPA for the web Deployment.
- Pod security labels and container `securityContext` configuration.

### Operational evidence

Participants must provide:

1. The Ingress URL or endpoint and a screenshot/record showing the ShopLite page loading.
2. Output showing web and database Pods in `Running` and Ready state.
3. PVC status showing `Bound` and evidence that the sample database record is available.
4. Evidence that the web tier can reach the database and an unauthorized test Pod cannot.
5. HPA output showing target CPU, minimum/maximum replicas and current/desired replica count.
6. At least three KQL query results covering Pod inventory, container logs and Kubernetes events.
7. One Azure Monitor alert rule or existing alert configuration validated for the application.
8. A short troubleshooting note describing the issue tested, the diagnostic commands/queries used, the root cause and the corrective action.

### Handover package

Submit a project folder or repository containing:

- Kubernetes YAML manifests (or Helm values, if used) for the application resources.
- A README with deployment steps, the Ingress endpoint, validation steps and cleanup commands.
- A configuration inventory identifying ConfigMap keys and Secret names. **Do not commit clear-text credentials or production secrets.**
- A short architecture diagram and operational-readiness notes.
- Screenshots or terminal outputs supporting the acceptance criteria above.

## 6. Acceptance Criteria

The capstone is considered complete when the following checks pass:

- [ ] The ShopLite application is reachable through the Ingress endpoint.
- [ ] The web tier connects to PostgreSQL through an internal ClusterIP Service.
- [ ] Database credentials are supplied through a Kubernetes Secret, not hardcoded in the image.
- [ ] Non-sensitive application settings are supplied through a ConfigMap.
- [ ] The database uses an Azure Disk-backed PVC and the PVC is `Bound`.
- [ ] A sample record remains available after the database Pod is recreated or rescheduled, where the provided lab environment permits this validation.
- [ ] NetworkPolicy allows required application traffic and blocks an unauthorized test workload.
- [ ] The web workload satisfies the namespace's Restricted Pod Security requirements.
- [ ] HPA is configured and its metrics/replica status are observable.
- [ ] Container Insights and KQL provide usable workload diagnostics.
- [ ] One application-health alert is created or validated.
- [ ] Participants can explain the recovery action and key production-readiness considerations.

## 7. Trainer-Provided Prerequisites

To make the project achievable in **120 minutes**, the following should be prepared before the session. Cluster creation and add-on installation are intentionally outside the capstone time limit.

- An existing AKS cluster with participant access through Azure Portal Cloud Shell (Bash).
- A supported Linux node pool with sufficient capacity for the web and database workloads.
- Metrics Server available for CPU-based HPA.
- Azure Disk CSI driver and a usable Azure Disk StorageClass.
- An Ingress controller already installed and reachable; provide its namespace/Pod labels and the hostname or test endpoint participants should use.
- A supported NetworkPolicy engine enabled and enforcing policies.
- Azure Monitor Container Insights already enabled, with a Log Analytics workspace receiving AKS data.
- An Azure Monitor action group or a pre-created alert template that participants can use.
- A sample ShopLite web application image compatible with Restricted Pod Security and capable of connecting to the supplied PostgreSQL image.
- A tested set of starter manifests, sample DB initialization data and a short load-generation command.
- Sufficient RBAC permissions to create namespace-scoped resources, PVCs, Services, Ingresses, NetworkPolicies and HPAs.

**Recommended starter assets:** Provide a small repository with the web/database image references, a README, sample application settings and optional manifest skeletons. Participants should complete and configure the Kubernetes resources rather than spend the two hours writing application code or building container images.

## 8. Scope and Time-Boxing Notes

This is a two-hour capstone and should be treated as a **production-readiness implementation exercise**, not a full enterprise production rollout. The cluster is pre-provisioned, application images are supplied, and cluster-level add-ons are ready before the session.

The following topics should be inspected or documented rather than changed during the two-hour exercise: AKS network plugin migration, private-cluster conversion, authorized API server IP changes, Kubernetes/node-pool upgrades, multi-region design, cluster autoscaler limits and node-pool VM SKU changes. These activities can affect the shared training cluster or require additional planning and downtime.

**Optional stretch goals (only after all acceptance criteria pass):**

- Create an Azure Files RWX PVC for shared product images or uploaded assets.
- Add TLS to the Ingress using a trainer-provided certificate/secret.
- Add a Pod Disruption Budget for the web tier.
- Compare the selected Ingress design with NodePort, LoadBalancer and AGIC/Application Gateway-based approaches.
- Document a database backup/restore plan, node-pool upgrade strategy, availability-zone placement and cost-optimization recommendations such as right-sizing or Spot user node pools.

---

## 9. Evaluation Rubric (100 Marks)

| Area | Marks | Evaluation focus |
|---|---:|---|
| AKS access and resource organization | 5 | Correct cluster context, namespace and resource inspection |
| Application deployment and configuration | 15 | Web/database deployment, ConfigMap, Secret and internal service discovery |
| Persistent storage | 15 | Azure Disk PVC, binding, mount and data-persistence evidence |
| Ingress and networking | 15 | Ingress routing, internal database service and valid NetworkPolicy tests |
| Security | 15 | Restricted Pod Security, non-root execution and hardened `securityContext` |
| Autoscaling | 10 | HPA configuration, load generation and replica/metric observations |
| Monitoring and alerting | 15 | Container Insights, KQL queries and one application-health alert |
| Troubleshooting and handover | 10 | Structured diagnosis, recovery evidence, README and operational notes |
| **Total** | **100** | |

**Final deliverable:** A reachable, secure and monitored ShopLite e-commerce application on AKS, supported by Kubernetes manifests, a working persistent database, autoscaling and network controls, monitoring/alert evidence, and a concise operational handover.
