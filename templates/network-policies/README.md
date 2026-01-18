# Network Policies

This directory contains Kubernetes NetworkPolicies used to control network traffic for the Flask application running on Amazon EKS.
These policies are used to secure the application by allowing only the required traffic and blocking everything else.

<p align="center">
  <img src="./Network_policies.png" alt="EKS project Architecture" width="1000">
</p>


By default, Kubernetes allows all pods to talk to each other.
In production, this is not secure.

These NetworkPolicies:
- Block all unnecessary traffic
- Allow only required connections
- Protect the application from unwanted access

---

### 1. Default Deny Policy
This policy blocks all incoming and outgoing traffic for pods in the namespace.

Why:
- Acts as a security baseline
- Ensures nothing is allowed unless explicitly permitted

---

### 2. Prometheus to Flask
Allows traffic from the `monitoring` namespace to the Flask application.

Why:
- Prometheus needs access to collect application metrics
- Only monitoring traffic is allowed, nothing else

---

### 3. ALB to Flask
Allows traffic from within the VPC to the Flask application.

Why:
- AWS Application Load Balancer runs inside the VPC
- Required for external user traffic to reach the application
- Blocks traffic from outside the VPC

---

### 4. Flask Egress (Outgoing Traffic)
Allows the Flask application to send traffic only to required services.

Allowed:
- DNS (CoreDNS) for name resolution
- MySQL (Amazon RDS) for database access
- Redis (Amazon ElastiCache) for caching

Why:
- Application cannot work without DNS, database, and cache
- All other outbound traffic is blocked for security

---

## Summary

- All traffic is blocked by default
- Only required ingress and egress traffic is allowed
- Follows least-privilege security model
- Suitable for production EKS workloads

