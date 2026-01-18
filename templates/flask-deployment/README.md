This README is designed for a professional GitHub repository or internal documentation. It explains not just **what** the files are, but **how they work together** to create a production-grade environment.

---

# Flask Production Stack on AWS EKS

This repository contains the Kubernetes Helm templates for a high-availability Flask application. The architecture leverages AWS-managed services (RDS, ElastiCache) and industry-standard Kubernetes operators (AWS Load Balancer Controller, External Secrets).

## 🏗️ System Architecture

1. **Traffic Flow:** User → Route53 → **Application Load Balancer (ALB)** → Target Group (IP mode) → **Flask Pods**.
2. **Configuration:** Environment variables for endpoints; **External Secrets Operator** for sensitive credentials.
3. **Persistence:** State is stored in **RDS (MySQL)**; Caching is handled by **ElastiCache (Valkey/Redis)**.
4. **Scaling:** The **Horizontal Pod Autoscaler (HPA)** monitors CPU/RAM and adjusts the replica count dynamically.

---

## 📂 Manifest Explanations

### 1. `flask-app-deployment.yml` (The Brain)

Defines how the Flask containers run.

* **Security Context:** Implements a "Restricted" security profile. The filesystem is **read-only** (preventing malware injection), and it runs as a **non-root user**.
* **Probes:** * *Readiness:* Ensures the app is connected to DB/Redis before receiving traffic.
* *Liveness:* Automatically restarts the container if the Python process hangs.


* **Volumes:** Maps a virtual `tmp` directory in memory since the container's own disk is locked.

### 2. `flask-https-ingress.yml` (The Gateway)

Controls how the world reaches your app via the **AWS Load Balancer Controller**.

* **SSL Redirect:** A "Logic-based" annotation that catches HTTP (80) requests and forces them to HTTPS (443).
* **Target Type (IP):** Traffic goes directly from the ALB to the Pod IP, skipping the "Kube-proxy" hop for lower latency.
* **Group Name:** Allows this Ingress to share a single ALB with other services (like ArgoCD or Grafana) to save costs.

### 3. `flask-app-secrets.yml` (The Vault)

Integrates with **AWS Secrets Manager**.

* Instead of storing passwords in plain text or Git, this manifest tells the cluster: *"Go to AWS, find the secret at this ARN, and sync the 'password' field into a Kubernetes secret."*
* **Refresh Interval:** Automatically updates the password in the cluster if it is rotated in AWS.

### 4. `flask-app-hpa.yml` (The Muscle)

Automates the scaling of your application.

* **CPU (50%) & Memory (70%):** If the average load exceeds these targets, the HPA will trigger the EKS Node Group to scale up (if needed) and spin up more Flask pods.

---

## 🛠️ Configuration & Environment Variables

The application expects the following environment variables to be injected (defined in the Deployment):

| Variable | Source | Description |
| --- | --- | --- |
| `DATABASE_HOST` | `values.yaml` | The RDS Endpoint. |
| `REDIS_HOST` | `values.yaml` | The ElastiCache Endpoint. |
| `DATABASE_USER` | AWS Secrets Manager | Synced via External Secrets. |
| `DATABASE_PASSWORD` | AWS Secrets Manager | Synced via External Secrets. |

---

## 🚀 How to Deploy

### 1. Prepare AWS Secrets

Ensure you have a secret in AWS Secrets Manager containing:

```json
{
  "username": "admin",
  "password": "your-secure-password"
}

```

### 2. Install via Helm

```bash
helm upgrade --install flask-app ./manifests \
  --namespace flask-app \
  --create-namespace \
  -f values.yaml

```

### 3. Verify the ALB

Check the Ingress status to find the Load Balancer DNS:

```bash
kubectl get ingress flask-app-ingress -n flask-app

```

---

## 🛡️ Security Features

* **Network Privacy:** The app is deployed in Private Subnets; only the ALB is Public.
* **No Hardcoded Secrets:** Zero credentials exist in the source code.
* **Resource Quotas:** CPU/Memory requests and limits prevent a single pod from crashing the entire Node.

---

**Would you like me to generate the corresponding `values.yaml` file so you can test this deployment immediately?**
