# DEPLOYMENT.md — Multi-Cluster RideShare (SwiftRide)

## Prerequisites

Ensure the following tools are installed and configured before deploying:

| Tool | Minimum Version | Purpose |
|---|---|---|
| AWS CLI | v2.x | Cluster kubeconfig, Secrets Manager |
| kubectl | v1.28+ | Kubernetes resource management |
| eksctl | v0.170+ | EKS cluster operations |
| helm | v3.x | NGINX Ingress, ESO installation |
| git | any | Clone repository |

AWS credentials must have permissions for: EKS, EC2, Secrets Manager, Route 53, and IAM.

---

## Repository Setup

```bash
git clone https://github.com/lightosita/Multi-Cluster-RideShare.git
cd Multi-Cluster-RideShare
git checkout week2-multi-cluster
```

---

## Step 1 — Configure Cluster Access

```bash
# Secondary cluster (Light Osita — us-east-1)
aws eks update-kubeconfig \
  --region us-east-1 \
  --name light-rideshare-cluster

# Primary cluster (Jibike — us-east-2)
aws eks update-kubeconfig \
  --region us-east-2 \
  --name jibike-rideshare-cluster \
  --alias partner-primary

# Confirm both contexts are present
kubectl config get-contexts
```

---

## Step 2 — Create Namespaces

Run on **both clusters**:

```bash
for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  kubectl create namespace rideshare-app --dry-run=client -o yaml | kubectl apply -f -
  kubectl create namespace rideshare-prod --dry-run=client -o yaml | kubectl apply -f -
done
```

---

## Step 3 — Install External Secrets Operator

Run on **both clusters**:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  helm upgrade --install external-secrets \
    external-secrets/external-secrets \
    --namespace external-secrets \
    --create-namespace \
    --set installCRDs=true
done
```

---

## Step 4 — Configure AWS Secrets Manager Access

The ESO `SecretStore` requires an IAM role with `secretsmanager:GetSecretValue` permission. This is pre-configured in the cluster's node role. Apply the `SecretStore` to both clusters:

```bash
for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  kubectl apply -f light-rideshare-k8s/platform/secrets/secret-store.yaml
done
```

Verify the store is ready:

```bash
kubectl get secretstore -n rideshare-app
# STATUS column should show: Valid
```

---

## Step 5 — Install NGINX Ingress Controller

Run on **both clusters**:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=LoadBalancer \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
done
```

Note the NLB hostnames after a few minutes — you will need them for Route 53:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

## Step 6 — Deploy Region ConfigMaps

```bash
# Secondary cluster (us-east-1)
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl apply -f light-rideshare-k8s/multi-cluster/configmap-secondary.yaml

# Primary cluster (us-east-2)
kubectl config use-context partner-primary
kubectl apply -f light-rideshare-k8s/multi-cluster/configmap-primary.yaml
```

---

## Step 7 — Deploy All Application Manifests

This applies Deployments, Services, HPAs, ExternalSecrets, and PodDisruptionBudgets.

```bash
for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  kubectl apply -f light-rideshare-k8s/applications/
done
```

Apply PodDisruptionBudgets separately if not already in the applications folder:

```bash
for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  kubectl apply -f light-rideshare-k8s/platform/pod-disruption-budgets.yaml
done
```

---

## Step 8 — Verify Deployments

```bash
# Check on secondary cluster
kubectl config use-context arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster
kubectl get pods -n rideshare-app
kubectl get pdb -n rideshare-app
kubectl get externalsecrets -n rideshare-app
kubectl get ingress -n rideshare-app

# Check on primary cluster
kubectl config use-context partner-primary
kubectl get pods -n rideshare-app
kubectl get pdb -n rideshare-app
```

All pods should be in `Running` state. All ExternalSecrets should show `SecretSynced = True`. All PDBs should show `ALLOWED DISRUPTIONS = 1`.

---

## Step 9 — Route 53 Health Checks & DNS

In the AWS Console (or via CLI), confirm:

1. Health checks exist for both NLB endpoints (`primary-us-east-2`, `secondary-us-east-1`)
2. Failover `A` alias records exist in hosted zone `Z034534514RGK1L1IR2XU` for `www.teleiosdupsy.space`

Test DNS resolution:

```bash
nslookup www.teleiosdupsy.space
curl -I https://www.teleiosdupsy.space
```

---

## Step 10 — CI/CD (GitHub Actions)

The pipeline in `.github/workflows/multi-cluster-deploy.yml` runs automatically on push to `week2-multi-cluster`. It requires the following GitHub Actions secrets:

| Secret Name | Description |
|---|---|
| `AWS_ROLE_ARN` | IAM role ARN for OIDC auth |
| `PRIMARY_CLUSTER_NAME` | `jibike-rideshare-cluster` |
| `SECONDARY_CLUSTER_NAME` | `light-rideshare-cluster` |
| `PRIMARY_CLUSTER_REGION` | `us-east-2` |
| `SECONDARY_CLUSTER_REGION` | `us-east-1` |
| `REGISTRY` | Container image registry URL |

---

## Failover Testing

### Simulate Primary Failure

```bash
kubectl config use-context partner-primary
kubectl scale deployment --all --replicas=0 -n rideshare-app

# Wait ~60 seconds, then verify secondary is serving traffic
curl https://www.teleiosdupsy.space/api/v1/riders/health
curl https://www.teleiosdupsy.space/api/v1/drivers/health

# Restore primary
kubectl scale deployment --all --replicas=2 -n rideshare-app
```

Expected behaviour: Route 53 detects the unhealthy health check within 30–90 seconds and automatically reroutes all traffic to the secondary cluster. RTO target is < 2 minutes.

---

## Cleanup

```bash
# Scale down both clusters first (saves cost)
for CTX in arn:aws:eks:us-east-1:221693237976:cluster/light-rideshare-cluster partner-primary; do
  kubectl config use-context $CTX
  kubectl scale deployment --all --replicas=0 -n rideshare-app
done

# Then in AWS Console:
# 1. Delete Route 53 health checks
# 2. Delete Route 53 A records in teleiosdupsy.space
# 3. Delete Route 53 hosted zone (if no longer needed)
# 4. Screenshot AWS Console showing no active billed resources
```

Or use the automated cleanup script:

```bash
chmod +x cleanup.sh
./cleanup.sh
```
