# Overview

This section contains all Kubernetes manifests required to deploy and expose the application on Amazon EKS using Helm.

It includes:
- Flask application Deployment configuration
- Horizontal Pod Autoscaler (HPA) for automatic scaling
- Cluster-internal Service for pod networking
- HTTPS Ingress using AWS Application Load Balancer (ALB)
- External Secrets configuration to fetch database credentials from AWS Secrets Manager
---

```mermaid

flowchart TD
    USER[Internet User] --> ALB[AWS ALB\nHTTPS/SSL]
    
    ALB -->|Port 3000| INGRESS[Ingress\nSSL Redirect]
    INGRESS --> SERVICE[Service\nNamespace: flask-mysql-redis-app]
    
    subgraph APP_NS[Namespace: flask-mysql-redis-app]
        POD1[Flask Pod 1\nPort 3000]
        POD2[Flask Pod 2\nPort 3000]
        POD3[Flask Pod 3\nPort 3000]
    end
    
    SERVICE --> POD1
    SERVICE --> POD2
    SERVICE --> POD3
    
    HPA[HPA\nAuto-scaling] --> POD1
    HPA --> POD2
    HPA --> POD3
    
    SECRETS[AWS Secrets Manager] --> EXT_SEC[External Secrets]
    EXT_SEC --> K8S_SEC[K8s Secret\nDatabase Creds]
    K8S_SEC --> POD1
    K8S_SEC --> POD2
    K8S_SEC --> POD3
    
    EXTERNAL_DNS[ExternalDNS] -->|Updates| R53[Route 53\nDNS Record]
    R53 --> USER
    
    POD1 -->|Port 3306| RDS[RDS MySQL]
    POD2 -->|Port 3306| RDS
    POD3 -->|Port 3306| RDS
    
    POD1 -->|Port 6379| EC[ElastiCache Redis]
    POD2 -->|Port 6379| EC
    POD3 -->|Port 6379| EC
    
    style USER fill:#e1f5e1
    style ALB fill:#fff3cd
    style RDS fill:#ffebee
    style EC fill:#e8f5e8
    style HPA fill:#f3e5f5
    style SECRETS fill:#e3f2fd
    style R53 fill:#f3e5f5
    style APP_NS fill:#f0f0f0,stroke:#000,stroke-width:1px
```

## Application Deployment - flask-app-deployment.yml

- Deploys the Flask application as Kubernetes pods
- Uses Helm values for configuration
- Connects the app to Amazon RDS (MySQL) and ElastiCache (Redis)
- Applies security best practices
- Supports autoscaling using HPA

### Deployment
Creates and manages Flask application pods in the specified namespace.

- Uses labels for pod selection
- HPA controls the number of replicas (replica count is commented)

Container Image
```yaml
image: repository:tag
````

* Image is pulled from Docker Hub
* Tag is updated automatically via CI/CD
* Same image is reused across environments

Ports

```yaml
containerPort: 3000
```

* Flask app listens on port 3000 inside the container
* Service and Ingress route traffic to this port

### Health Checks (Probes)

Prevents broken pods from receiving traffic & improves availability during deployments

* Liveness Probe: Checks `/status` & restarts the container if the app becomes unhealthy
* Readiness Probe: Checks `/status` and ensures traffic is sent only when the app is ready

### Security Context

It Reduces attack surface. Follows Kubernetes and EKS security best practices. The container runs with strict security settings:

* Runs as non-root user
* No privilege escalation
* Read-only root filesystem
* All Linux capabilities dropped
* Uses default seccomp profile

### Resource Management

* Ensures fair resource usage
* Prevents one pod from consuming all node resources
* Required for autoscaling to work correctly

```yaml
requests:
  cpu: 50m
  memory: 128Mi
limits:
  cpu: 100m
  memory: 256Mi
```

### Environment Variables

Sensitive data is not stored in plain text

Env variables are configured using Helm values and Kubernetes Secrets:

* Database host (RDS)
* Redis host (ElastiCache)
* Ports for MySQL and Redis

Secrets are loaded securely using:

```yaml
envFrom:
  - secretRef
```

### Temporary Storage

Needed because root filesystem is read-only. They are Used for temporary runtime files

```yaml
emptyDir volume mounted at /tmp
```
---

## Flask Application Autoscaling (HPA) - flask-app-hpa.yml

This defines a Kubernetes Horizontal Pod Autoscaler (HPA) for the Flask application deployed using Helm.

The HPA automatically adjusts the number of Flask pods based on CPU and memory usage.

---

## Flask Application Service - flask-app-service.yml

This file defines a Kubernetes Service for the Flask application.
The Service provides a stable network endpoint to access Flask pods inside the cluster.

---

## Flask HTTPS Ingress (AWS ALB)

This Ingress exposes the Flask application to the internet using an AWS Application Load Balancer with HTTPS. It is deployed using a Helm template and is managed by the AWS Load Balancer Controller.

- The ALB is created as internet-facing and listens on both HTTP (80) and HTTPS (443).  
- HTTP traffic is redirected to HTTPS using an ALB redirect action.

- TLS is terminated at the ALB using an ACM certificate.  
- A modern SSL policy is enforced for secure communication.

- The ALB uses IP target mode, so traffic is routed directly to pod IPs.  
- Health checks are performed on the `/status` endpoint to ensure only healthy pods receive traffic.

- Multiple Ingress resources can share the same ALB using a common group name.

- ExternalDNS automatically creates or updates the Route 53 DNS record for the application domain and points it to the ALB.

- The Ingress is handled by the `alb` ingress class, ensuring it is processed by the AWS Load Balancer Controller.

- Routing rules match the application domain.  
 * HTTP requests are redirected to HTTPS.  
 * HTTPS requests are forwarded to the Flask Service.  
 * The Service load-balances traffic across the Flask pods.

Traffic flow:
User → Route 53 → ALB → Flask Service → Flask Pods

---

## Flask Application Secrets (External Secrets)

This manifest creates Kubernetes secrets using External Secrets Operator.

Secrets are fetched from AWS Secrets Manager through a ClusterSecretStore.

The ExternalSecret resource runs in the application namespace and creates a Kubernetes Secret named `flask-app-secret`.

Secrets are refreshed every 10 seconds to keep values in sync with AWS.

The secret store reference points to `aws-secret-store`, which is configured at cluster level.

The created Kubernetes Secret is owned by this ExternalSecret and is retained even if the ExternalSecret is deleted.

Database credentials are pulled from a single AWS Secrets Manager secret.
- `DATABASE_USER` is mapped from the `username` field
- `DATABASE_PASSWORD` is mapped from the `password` field

These secrets are later consumed by the Flask Deployment using `envFrom`.

Flow: 

AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → Flask Pods

---
