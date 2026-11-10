# SWYNEX Cloud Architecture Outline

**Internship Task 1 – SWYNEX Technologies**
**Author:** A. Antony Selva Jasfer

## 1. Purpose and Scope

This document outlines, at a conceptual level, the cloud architecture for a typical three-tier web application (for example, an AI/data-analytics web portal). It covers four pillars: **Compute, Storage, Networking, and Identity**. The design is provider-neutral; Section 7 maps each generic component to AWS, Azure and GCP equivalents.

Out of scope: detailed sizing, cost modelling, IaC code, and CI/CD pipelines.

## 2. Architecture at a Glance

```
                          Users (Browser / Mobile)
                                    |
                         DNS  ->  CDN + WAF
                                    |
                        Public Load Balancer (HTTPS)
        ============================|===================================
        |  Virtual Network (VPC / VNet)                                 |
        |                           |                                   |
        |   Public subnet:   NAT Gateway, Bastion (optional)            |
        |                           |                                   |
        |   Private subnet (App):   |                                   |
        |      [ Web/API containers or VMs  -- auto-scaled ]            |
        |      [ Background workers / Functions ]                       |
        |                           |                                   |
        |   Private subnet (Data):  |                                   |
        |      [ Managed relational DB (primary + standby) ]            |
        |      [ Cache (in-memory) ]                                    |
        ================================================================
                 |                      |                    |
          Object Storage          Secrets / Key Mgmt     Monitoring & Logs
        (files, backups, models)   (encryption keys)     (metrics, alerts)

   Identity & Access Management spans every layer (users, services, admins)
```

## 3. Compute

| Component | Role | Notes |
|---|---|---|
| **Application tier** | Runs the web front-end and REST API | Containers (preferred) or VMs, behind a load balancer |
| **Auto-scaling group** | Adds or removes instances based on CPU / request load | Min 2 instances across 2 availability zones for high availability |
| **Background workers / serverless functions** | Asynchronous jobs: report generation, ML inference, email | Triggered by a message queue; scale to zero when idle |
| **Message queue** | Decouples the API from slow jobs | Improves resilience and response time |

**Design choices:** stateless application servers so any instance can serve any request; horizontal scaling over vertical scaling; immutable images for repeatable deployments.

## 4. Storage

| Type | Purpose | Examples of data |
|---|---|---|
| **Object storage** | Cheap, durable, scalable file storage | User uploads, static assets, logs, ML model artifacts, backups |
| **Managed relational database** | Structured, transactional data | Users, orders, application records |
| **In-memory cache** | Low-latency reads | Sessions, frequently accessed query results |
| **Block storage** | Disks attached to VMs where needed | OS volumes |

**Design choices:**
- Multi-AZ database with automated backups and point-in-time recovery.
- Object storage with versioning and lifecycle rules (hot -> infrequent -> archive).
- Encryption at rest (managed keys) for every storage type.

## 5. Networking

- **Virtual network** divided into **public** and **private** subnets across at least two availability zones.
- **Public subnet:** holds only the load balancer and NAT gateway. Nothing else is reachable from the internet.
- **Private subnets:** application tier and data tier. The database accepts traffic only from the application tier.
- **Security groups / firewall rules:** least-privilege, allow-list based (e.g., load balancer -> app on 443/8080; app -> DB on the DB port only).
- **CDN + WAF:** caches static content close to users and blocks common web attacks (SQL injection, XSS, bot traffic).
- **DNS + TLS:** managed DNS with TLS certificates; HTTPS enforced everywhere, TLS in transit between tiers where practical.
- **Private endpoints:** access to object storage and other managed services without crossing the public internet.
- **Admin access:** via a bastion host or provider session manager / VPN, never open SSH/RDP to the internet.

## 6. Identity and Access Management (IAM)

| Concern | Approach |
|---|---|
| **End-user authentication** | Managed identity service (user pools / external IdP) with email + password, social login and MFA |
| **Workforce / admin access** | Single sign-on with MFA; role-based access control (RBAC); no shared accounts |
| **Service-to-service access** | Workload identities / IAM roles attached to compute, with no long-lived keys stored in code |
| **Least privilege** | Each role gets only the permissions its job requires; reviewed periodically |
| **Secrets management** | Passwords, API keys and certificates held in a secrets manager and key management service, rotated automatically |
| **Auditing** | All admin and API activity logged to a tamper-resistant audit trail |

## 7. Provider Mapping

| Generic component | AWS | Azure | GCP |
|---|---|---|---|
| Virtual network | VPC | Virtual Network (VNet) | VPC Network |
| Load balancer | Application Load Balancer | Application Gateway / Load Balancer | Cloud Load Balancing |
| Containers / VMs | ECS / EKS / EC2 | AKS / Container Apps / VMs | GKE / Cloud Run / Compute Engine |
| Auto-scaling | Auto Scaling Groups | VM Scale Sets | Managed Instance Groups |
| Serverless functions | Lambda | Azure Functions | Cloud Functions |
| Message queue | SQS | Service Bus / Queue Storage | Pub/Sub |
| Object storage | S3 | Blob Storage | Cloud Storage |
| Managed SQL database | RDS / Aurora | Azure SQL / PostgreSQL Flexible Server | Cloud SQL |
| In-memory cache | ElastiCache | Azure Cache for Redis | Memorystore |
| CDN | CloudFront | Azure Front Door / CDN | Cloud CDN |
| WAF | AWS WAF | Azure WAF | Cloud Armor |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Identity (users) | Cognito | Entra External ID | Identity Platform |
| Identity (IAM) | IAM | Microsoft Entra ID + RBAC | Cloud IAM |
| Secrets / keys | Secrets Manager / KMS | Key Vault | Secret Manager / Cloud KMS |
| Monitoring and logs | CloudWatch / CloudTrail | Azure Monitor | Cloud Monitoring / Logging |

## 8. Cross-Cutting Concerns

- **Availability:** multi-AZ deployment, health checks, automatic failover.
- **Security:** defence in depth: edge (WAF), network (subnets, firewalls), identity (IAM, MFA), data (encryption).
- **Observability:** centralised logs, metrics dashboards and alerts on error rate, latency and saturation.
- **Backup and disaster recovery:** automated backups, cross-region copy of critical data, documented recovery targets (RPO/RTO).
- **Cost awareness:** auto-scaling, storage lifecycle tiers, and resource tagging for cost tracking.

## 9. Request Flow (Example)

1. A user opens the site; DNS resolves to the CDN.
2. The CDN serves static assets from cache; dynamic requests pass the WAF to the load balancer.
3. The load balancer forwards the request to a healthy app instance in a private subnet.
4. The app checks the user's token with the identity service.
5. The app reads or writes the database (via cache when possible) and stores files in object storage.
6. Slow tasks are placed on a queue and processed by background workers.
7. Logs and metrics flow to the monitoring service for alerting.

## 10. Summary

The architecture keeps the application tier stateless and auto-scaled, isolates data in private subnets, stores files in durable object storage, and uses centralised identity with least-privilege roles. Because it is expressed with generic building blocks, it can be implemented on AWS, Azure or GCP with the mappings above.
