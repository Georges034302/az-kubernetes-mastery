# AKS Professional Training: Hands-On Lab Series
### Advanced Kubernetes on Azure – 6 Structured Sessions with 25 Labs

## 📚 Technical Introduction

This hands-on training series provides practical, end-to-end experience with **Azure Kubernetes Service (AKS)**, covering cluster scaling, governance, security, networking, observability, deployments, service mesh, and data/ML integration using Databricks.  

Across **6 in-depth sessions**, you will build real-world AKS architectures, enforce enterprise-grade controls, and deploy production-ready workloads through modern DevOps and cloud-native patterns.

---

### **Prerequisites**
- Understanding of cloud fundamentals (IaaS/PaaS/SaaS).
- Familiarity with Kubernetes concepts (pods, deployments, services).
- Azure CLI, kubectl, and basic YAML knowledge.
- Access to an Azure subscription (personal or sandbox).
- Optional: Experience with Helm, GitHub Actions, or Databricks.

---

# 📘 Lab Sessions

---

<details>
  <summary><strong>Session 01: Cluster Auto Scaling & Upgrades</strong></summary>

Master AKS autoscaling from **cluster-level** (CA) to **pod-level** (HPA/VPA), and implement **safe upgrade strategies** using surge settings and node pool separation.

**Labs for this session:**
- **[lab_1_a_hpa-autoscaling.md](session01/lab_1_a_hpa-autoscaling.md)**  
  *Deploy CPU-intensive workload and observe HPA-driven pod autoscaling.*

- **[lab_1_b_create-user-nodepool.md](session01/lab_1_b_create-user-nodepool.md)**  
  *Create a user node pool and schedule batch workloads using labels, taints, and tolerations.*

- **[lab_1_c_nodepool-upgrade-surge.md](session01/lab_1_c_nodepool-upgrade-surge.md)**  
  *Perform a node pool upgrade using `--max-surge` and validate workload continuity.*

</details>

---

<details>
  <summary><strong>Session 02: Manage Add-Ons and RBAC</strong></summary>

Enable enterprise-grade governance through AKS add-ons such as **Azure Policy**, **KEDA**, **OSM**, and **Key Vault CSI**, and design multi-team RBAC frameworks using Microsoft Entra ID.

**Labs for this session:**
- **[lab_2_a_enable-azure-policy.md](session02/lab_2_a_enable-azure-policy.md)**  
  *Enforce DenyPrivilegedContainer with the Azure Policy add-on.*

- **[lab_2_b_keda-queue-autoscaling.md](session02/lab_2_b_keda-queue-autoscaling.md)**  
  *Deploy KEDA and observe event-driven scaling using Azure Queue Storage.*

- **[lab_2_c_rbac-auth-with-entra.md](session02/lab_2_c_rbac-auth-with-entra.md)**  
  *Authenticate to AKS using Entra ID and test RBAC permissions.*

- **[lab_2_d_namespace-rbac.md](session02/lab_2_d_namespace-rbac.md)**  
  *Create namespace-scoped RBAC for multi-team isolation.*

</details>

---

<details>
  <summary><strong>Session 03: Security & Networking</strong></summary>

Implement layered security using **Key Vault CSI**, **Pod Security Admission**, **Network Policies**, and **Ingress/LoadBalancer patterns**, and validate runtime protection using Defender.

**Labs for this session:**
- **[lab_3_a_keyvault-csi-mount.md](session03/lab_3_a_keyvault-csi-mount.md)**  
  *Mount Key Vault secrets into pods using the CSI driver.*

- **[lab_3_b_psa-deny-privileged.md](session03/lab_3_b_psa-deny-privileged.md)**  
  *Enforce restricted Pod Security admission and block privileged pods.*

- **[lab_3_c_networkpolicy-denyall.md](session03/lab_3_c_networkpolicy-denyall.md)**  
  *Apply deny-all policy and verify pod-to-pod isolation.*

- **[lab_3_d_expose-with-loadbalancer.md](session03/lab_3_d_expose-with-loadbalancer.md)**  
  *Expose workloads using a public LoadBalancer.*

- **[lab_3_e_ingress-routing.md](session03/lab_3_e_ingress-routing.md)**  
  *Deploy NGINX Ingress and configure dynamic path-based routing.*

- **[lab_3_f_defender-egress-validation.md](session03/lab_3_f_defender-egress-validation.md)**  
  *Simulate threats and validate egress control using Azure Firewall.*

</details>

---

<details>
  <summary><strong>Session 04: Monitoring, Logging & Multi-Region Awareness</strong></summary>

Build complete observability pipelines using **Azure Monitor**, **Log Analytics (KQL)**, **Prometheus**, **Grafana**, and **Fluent Bit**, then extend operations across regions with multi-cluster failover.

**Labs for this session:**
- **[lab_4_a_kql-pod-restarts.md](session04/lab_4_a_kql-pod-restarts.md)**  
  *Query restarts and failed deployments with KQL.*

- **[lab_4_b_prometheus-grafana.md](session04/lab_4_b_prometheus-grafana.md)**  
  *Deploy Prometheus & Grafana and visualize cluster metrics.*

- **[lab_4_c_fluentbit-central-logging.md](session04/lab_4_c_fluentbit-central-logging.md)**  
  *Forward AKS logs to Log Analytics using Fluent Bit.*

- **[lab_4_d_multi-region-failover.md](session04/lab_4_d_multi-region-failover.md)**  
  *Deploy workloads across two AKS regions and simulate failover.*

</details>

---

<details>
  <summary><strong>Session 05: Deployments & Service Mesh Integration</strong></summary>

Deploy applications using **Helm**, implement **rolling/blue-green/canary** deployments with **Flagger**, and secure microservices with **Istio** through mTLS, traffic rules, and mesh observability.

**Labs for this session:**
- **[lab_5_a_helm-microservice-deployment.md](session05/lab_5_a_helm-microservice-deployment.md)**  
  *Package and deploy a microservice using Helm.*

- **[lab_5_b_flagger-canary.md](session05/lab_5_b_flagger-canary.md)**  
  *Automate canary rollout (10–50–100) with Flagger + Prometheus.*

- **[lab_5_c_istio-mtls.md](session05/lab_5_c_istio-mtls.md)**  
  *Enable strict mTLS and secure service-to-service communication.*

- **[lab_5_d_kiali-jaeger-observability.md](session05/lab_5_d_kiali-jaeger-observability.md)**  
  *Trace requests and visualize mesh topology using Kiali + Jaeger.*

</details>

---

<details>
  <summary><strong>Session 06: AKS & Databricks Integration</strong></summary>

Integrate ML workflows end-to-end—from Databricks notebooks and MLflow tracking to AKS-based model deployment using CI/CD, managed identities, and secure connectivity.

**Labs for this session:**
- **[lab_6_a_explore-databricks.md](session06/lab_6_a_explore-databricks.md)**  
  *Explore Databricks workspace, Spark cluster, and MLflow experiments.*

- **[lab_6_b_aks-api-query-databricks.md](session06/lab_6_b_aks-api-query-databricks.md)**  
  *Deploy AKS API that securely queries Databricks datasets.*

- **[lab_6_c_mlflow-automated-deployment.md](session06/lab_6_c_mlflow-automated-deployment.md)**  
  *Automate ML model packaging and deployment to AKS.*

- **[lab_6_d_managed-identity-keyvault.md](session06/lab_6_d_managed-identity-keyvault.md)**  
  *Enable secure authentication between AKS → Key Vault → Databricks.*

</details>

---

# 🧑‍🏫 Author: Georges Bou Ghantous, Ph.D.

This repository delivers advanced AKS training through hands-on labs spanning cluster scaling, governance, pod security, networking, observability, deployment automation, service mesh, and ML/Databricks integration.