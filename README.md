Multi-Cluster RideShare - Production-Grade Multi-Region Architecture on AWS
Team Members

Light Osita (Secondary Cluster - us-east-1)
Jibike (Primary Cluster - us-east-2)


📋 Project Overview
Business Justification
RideShare has expanded to multiple cities across different time zones. The platform must:

Handle regional traffic patterns efficiently
Comply with data residency requirements
Maintain 99.99% uptime during peak usage periods
Survive complete regional failures without service interruption

This project transforms our Week 1 single-cluster deployment into a production-grade, multi-cluster architecture using AWS managed services.

🏗️ Multi-Cluster Architecture
Architecture Diagram
Internet
    │
    ▼
Route 53 (Global DNS - Failover Routing)
teleiosdupsy.space
    │
    ├──────────────────────────────────────┐
    ▼                                      ▼
EKS Cluster (Primary)              EKS Cluster (Secondary)
jibike-rideshare-cluster           light-rideshare-cluster
us-east-2                          us-east-1
    │                                      │
NGINX Ingress                      NGINX Ingress
NLB (us-east-2)                    NLB (us-east-1)
    │                                      │
    └──────────────┬───────────────────────┘
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
Amazon RDS PostgreSQL    Upstash Redis
(Shared Instance)        (Managed Redis)
us-east-1                Global
Architecture Evolution
Week 1Week 2Single EKS ClusterTwo EKS Clusters (Multi-Region)StatefulSets (Redis + PostgreSQL)Managed Services (RDS + Redis)Local DNSRoute 53 Global DNS with FailoverSingle Point of FailureActive-Passive Redundancy
Key Components
Regional Infrastructure (Per Cluster):

EKS Cluster with full microservices deployment
NGINX Ingress Controller for traffic management
Network Load Balancer for health checks
External Secrets Operator for secrets management

Global Infrastructure:

Route 53 with health-based failover routing
Amazon RDS PostgreSQL (shared managed database)
Upstash Redis (managed Redis with TLS)
AWS Secrets Manager for secret storage

Microservices (Deployed to Both Clusters):

rider-service (port 3001)
driver-service (port 3003)
trip-service (port 3005)
matching-service (port 3004)
email-service (port 3002)
frontend (port 3000)


✅ Technical Achievements

 Two EKS clusters configured as multi-cluster setup
 Cross-cluster kubeconfig access configured
 rideshare-prod namespace on both clusters
 Region-specific ConfigMaps deployed to both clusters
 Shared RDS PostgreSQL connected from both clusters
 Managed Redis (Upstash) connected with TLS
 AWS Secrets Manager secrets updated with REDIS_URL
 External Secrets Operator syncing on both clusters
 Route 53 hosted zone created for teleiosdupsy.space
 Failover DNS records configured (Primary + Secondary)
 Route 53 health checks for both clusters
 Migration from StatefulSets to managed services completed


🔧 Prerequisites

AWS Account with appropriate IAM permissions
AWS CLI configured (aws configure)
kubectl installed
eksctl installed
helm installed
Git
Access to both EKS clusters


🚀 Quick Start Guide (< 15 minutes)
1. Clone the Repository
bashgit clone https://github.com/lightosita/Telieos-Ride-Project.git
cd Telieos-Ride-Project
git checkout week2-multi-cluster
2. Configure Cluster Access
bash# Configure your cluster (secondary)
aws eks update-kubeconfig --region us-east-1 --name light-rideshare-cluster

# Configure partner's cluster (primary)
aws eks update-kubeconfig --region us-east-2 --name jibike-rideshare-cluster --alias partner-primary

# Verify both clusters
kubectl config get-contexts
3. Create Namespaces on Both Clusters
bash# Secondary cluster
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl create namespace rideshare-prod --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace rideshare-app --dry-run=client -o yaml | kubectl apply -f -

# Primary cluster
kubectl config use-context partner-primary
kubectl create namespace rideshare-prod --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace rideshare-app --dry-run=client -o yaml | kubectl apply -f -
4. Deploy Region ConfigMaps
bash# Secondary cluster (us-east-1)
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl apply -f light-rideshare-k8s/multi-cluster/configmap-secondary.yaml

# Primary cluster (us-east-2)
kubectl config use-context partner-primary
kubectl apply -f light-rideshare-k8s/multi-cluster/configmap-primary.yaml
5. Deploy Applications to Both Clusters
bash# Secondary cluster
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl apply -f light-rideshare-k8s/applications/

# Primary cluster
kubectl config use-context partner-primary
kubectl apply -f light-rideshare-k8s/applications/
6. Verify Deployments
bashkubectl get pods -n rideshare-app
kubectl get ingress -n rideshare-app
kubectl get externalsecrets -n rideshare-app

🌍 Route 53 Configuration
DNS Setup

Domain: www.teleiosdupsy.space
Hosted Zone ID: Z034534514RGK1L1IR2XU
Routing Policy: Failover

Records
RecordTypeRoutingTargetwww.teleiosdupsy.spaceA (Alias)Failover PrimaryPartner NLB (us-east-2)www.teleiosdupsy.spaceA (Alias)Failover SecondaryYour NLB (us-east-1)
Health Checks
NameEndpointProtocolprimary-us-east-2Partner NLB (us-east-2)HTTPSsecondary-us-east-1Your NLB (us-east-1)HTTPS

🔄 Failover Testing
Test 1: Simulate Primary Cluster Failure
bash# Scale down primary cluster
kubectl config use-context partner-primary
kubectl scale deployment --all --replicas=0 -n rideshare-app

# Verify Route 53 routes to secondary
nslookup www.teleiosdupsy.space
curl -I https://www.teleiosdupsy.space

# Restore primary
kubectl scale deployment --all --replicas=2 -n rideshare-app
Test 2: Application Resilience
bash# Verify secondary serves traffic when primary is down
curl https://www.teleiosdupsy.space/api/v1/riders/health
curl https://www.teleiosdupsy.space/api/v1/drivers/health
Expected Failover Behavior

Route 53 detects unhealthy primary within 30-90 seconds
Traffic automatically routes to secondary cluster
RDS and Redis continue serving both clusters
Recovery time objective (RTO): < 2 minutes


💰 Cost Optimization Strategies
1. Use Spot Instances

Dev/test node groups use Spot instances (70% savings)
Mixed instance strategy for production workloads

2. Right-Size Resources

Start with t3.medium nodes
Use AWS Compute Optimizer recommendations
Current resource limits: CPU 250m, Memory 256Mi per service

3. Auto-Shutdown

Scale down clusters during non-working hours
Use Lambda functions for automated scheduling
Estimated savings: 60% on compute costs

4. Optimize Data Transfer

Keep read traffic regional where possible
Use VPC endpoints for AWS services
Shared RDS instance across cohort reduces per-team costs

5. Clean Up Promptly

Run cleanup script immediately after demo
Delete Route 53 health checks when not needed
Automate cleanup with provided script

Cost Breakdown (Estimated Monthly)
ResourceCostEKS Cluster (x2)~$0.10/hr eachEC2 Nodes t3.medium (x2 per cluster)~$0.0416/hr eachRDS PostgreSQL (shared)Shared across cohortUpstash RedisFree tierRoute 53 Hosted Zone$0.50/monthRoute 53 Health Checks$0.50/month each

🧹 Cleanup Instructions
bash# Run cleanup script
chmod +x cleanup.sh
./cleanup.sh
Manual Cleanup Steps
bash# Scale down both clusters
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl scale deployment --all --replicas=0 -n rideshare-app

kubectl config use-context partner-primary
kubectl scale deployment --all --replicas=0 -n rideshare-app
Then in AWS Console:

Delete Route 53 health checks
Delete Route 53 records in teleiosdupsy.space
Delete Route 53 hosted zone
Take screenshot of AWS Console showing no active resources


📁 Repository Structure
Telieos-Ride-Project/
├── light-rideshare-k8s/
│   ├── applications/
│   │   ├── driver-service/
│   │   │   ├── configmap.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── external-secret.yaml
│   │   │   ├── hpa.yaml
│   │   │   └── service.yaml
│   │   ├── rider-service/
│   │   ├── trip-service/
│   │   ├── matching-service/
│   │   ├── email-service/
│   │   └── frontend/
│   ├── multi-cluster/
│   │   ├── configmap-primary.yaml
│   │   └── configmap-secondary.yaml
│   ├── platform/
│   │   ├── ingress/
│   │   └── secrets/
│   └── aws/
├── .github/
│   └── workflows/
│       └── multi-cluster-deploy.yml
├── cleanup.sh
└── README.md
