# ARCHITECTURE.md — Multi-Cluster RideShare (SwiftRide)

## Overview

SwiftRide is deployed as a **production-grade, active-passive multi-region** system across two AWS EKS clusters. The architecture eliminates the single points of failure present in the Week 1 single-cluster design by distributing workloads across AWS regions, migrating stateful services to fully managed AWS offerings, and implementing global DNS-based failover.

---

## High-Level Topology

```
Internet
    │
    ▼
Route 53 (teleiosdupsy.space)
Failover Routing Policy
    │
    ├─────────────────────────────────────┐
    ▼                                     ▼
EKS Primary Cluster                EKS Secondary Cluster
jibike-rideshare-cluster           light-rideshare-cluster
us-east-2 (Ohio)                   us-east-1 (N. Virginia)
    │                                     │
NGINX Ingress + NLB                NGINX Ingress + NLB
    │                                     │
    └──────────────┬──────────────────────┘
                   │
       ┌───────────┴────────────┐
       ▼                        ▼
Amazon RDS PostgreSQL       Upstash Redis
(Shared, us-east-1)         (Global, TLS)
```

---

## Cluster Configuration

| Property | Primary | Secondary |
|---|---|---|
| Cluster Name | `jibike-rideshare-cluster` | `light-rideshare-cluster` |
| AWS Region | `us-east-2` | `us-east-1` |
| Route 53 Role | Failover PRIMARY | Failover SECONDARY |
| Namespace | `rideshare-app` | `rideshare-app` |
| Node Type | t3.medium | t3.medium |

Both clusters run the same full microservices stack. In the active-passive model, Route 53 health checks determine which cluster serves live traffic at any given time.

---

## Microservices

| Service | Port | Responsibility |
|---|---|---|
| `rider-service` | 3001 | Rider registration, profiles, ride requests |
| `driver-service` | 3003 | Driver registration, availability, location |
| `trip-service` | 3005 | Trip lifecycle management, history |
| `matching-service` | 3004 | Pairs riders with available drivers |
| `email-service` | 3002 | Transactional email notifications |
| `frontend` | 3000 | React-based user interface |

Each service is deployed as a Kubernetes `Deployment` with:
- **Resource limits**: CPU 250m, Memory 256Mi per pod
- **Liveness & readiness probes** on `/health` endpoints
- **HPA** for horizontal pod autoscaling
- **PodDisruptionBudget** (`minAvailable: 1`) to protect against voluntary disruptions during node drains or upgrades

---

## Data Layer

### Amazon RDS PostgreSQL
- Shared managed PostgreSQL instance
- Accessed by both clusters over the public endpoint
- Connection credentials stored in AWS Secrets Manager and synced to pods via External Secrets Operator (ESO)
- Eliminates the PostgreSQL StatefulSet used in Week 1

### Upstash Redis (Managed)
- Global managed Redis with TLS enabled
- `REDIS_URL` secret stored in AWS Secrets Manager
- Replaces the in-cluster Redis StatefulSet from Week 1
- Zero operational overhead; no persistence management required

---

## Secrets Management

All sensitive configuration is managed through the **External Secrets Operator (ESO)** pattern:

```
AWS Secrets Manager
        │
        ▼
ExternalSecret (CRD) ──► ESO Controller ──► Kubernetes Secret
                                                    │
                                                    ▼
                                              Pod (env vars / volume mounts)
```

- `SecretStore` is configured per cluster pointing to AWS Secrets Manager in `us-east-1`
- `ExternalSecret` resources in `rideshare-app` namespace pull DB credentials, Redis URL, and API keys
- Secrets auto-rotate on a configurable sync interval

---

## Networking & Traffic Management

### NGINX Ingress Controller
- Deployed via Helm on both clusters
- Exposes services through an AWS Network Load Balancer (NLB)
- Routes traffic to internal services based on path prefixes (`/api/v1/riders`, `/api/v1/drivers`, etc.)

### Route 53 Failover Routing
- **Hosted Zone**: `teleiosdupsy.space` (ID: `Z034534514RGK1L1IR2XU`)
- Two `A` alias records pointing to Primary NLB (us-east-2) and Secondary NLB (us-east-1)
- **Health Checks** run against both NLB endpoints over HTTPS
- Failover triggers within **30–90 seconds** when the primary health check fails
- RTO target: **< 2 minutes**

---

## CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/multi-cluster-deploy.yml`) automates:

1. **Build** — Docker image build and push to container registry
2. **Deploy Primary** — `kubectl apply` to `jibike-rideshare-cluster` (us-east-2)
3. **Deploy Secondary** — `kubectl apply` to `light-rideshare-cluster` (us-east-1)
4. **Verify** — Health checks on both clusters post-deploy

Cluster access in CI is configured via OIDC-scoped AWS IAM roles stored as GitHub Actions secrets.

---

## Resilience & High Availability Summary

| Concern | Solution |
|---|---|
| Regional failure | Route 53 failover to secondary cluster |
| Node maintenance / drain | PodDisruptionBudgets prevent full service outage |
| Pod crashes | Liveness probes + Kubernetes self-healing restarts |
| Traffic spikes | HPA scales pods horizontally |
| Secret rotation | ESO syncs from AWS Secrets Manager automatically |
| Database HA | Amazon RDS managed service (Multi-AZ configurable) |
| Cache HA | Upstash Redis global replication |

---

## Evolution from Week 1

| Concern | Week 1 | Week 2 |
|---|---|---|
| Cluster topology | Single EKS cluster | Two EKS clusters (multi-region) |
| PostgreSQL | StatefulSet in-cluster | Amazon RDS (managed) |
| Redis | StatefulSet in-cluster | Upstash Redis (managed, TLS) |
| DNS | Local/NodePort | Route 53 with health-check failover |
| Secrets | ConfigMaps / manual | AWS Secrets Manager + ESO |
| CI/CD | None | GitHub Actions multi-cluster pipeline |
| Disruption protection | None | PodDisruptionBudgets on all services |
