# Plausible Analytics Kubernetes Deployment

This repository contains Kubernetes manifests and GitHub Actions workflows to deploy [Plausible Analytics](https://plausible.io/) to a Kubernetes cluster with Traefik ingress and cert-manager.

## Architecture

The deployment consists of:

- **Plausible Analytics** - Web analytics platform (port 8000)
- **PostgreSQL 16** - User data and configuration storage
- **ClickHouse 23.3** - Analytics events storage
- **Traefik Ingress** - HTTP/HTTPS routing with automatic TLS
- **cert-manager** - Automatic Let's Encrypt SSL certificates

## Prerequisites

- Kubernetes cluster with Traefik ingress controller
- cert-manager with `letsencrypt-prod` ClusterIssuer configured
- kubectl access to the cluster
- GitHub repository with Actions enabled

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy.yml      # Main deployment workflow
│       └── rollback.yml    # Rollback workflow
├── k8s/
│   ├── 01-namespace.yaml
│   ├── 02-configmap.yaml
│   ├── 03-postgres-pvc.yaml
│   ├── 04-postgres-statefulset.yaml
│   ├── 05-postgres-service.yaml
│   ├── 06-clickhouse-pvc.yaml
│   ├── 07-clickhouse-statefulset.yaml
│   ├── 08-clickhouse-service.yaml
│   ├── 09-plausible-deployment.yaml
│   ├── 10-plausible-service.yaml
│   ├── 11-https-redirect-middleware.yaml
│   └── 12-ingress.yaml
└── README.md
```

## Setup Instructions

### 1. Generate Required Secrets

Generate random secrets for Plausible:

```bash
# SECRET_KEY_BASE (64 characters)
openssl rand -base64 48 | tr -d '\n'

# TOTP_VAULT_KEY (32 characters)
openssl rand -base64 24 | tr -d '\n'

# POSTGRES_PASSWORD
openssl rand -base64 16 | tr -d '\n'

# CLICKHOUSE_PASSWORD (optional, can be empty)
openssl rand -base64 16 | tr -d '\n'
```

### 2. Configure GitHub Secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions** and add the following secrets:

#### Required Secrets

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `KUBE_CONFIG` | Base64-encoded kubeconfig file | See below for encoding |
| `BASE_URL` | Full URL where Plausible will be hosted | `https://plausible.jankylabs.co.uk` |
| `SECRET_KEY_BASE` | 64-character random string | Generated above |
| `TOTP_VAULT_KEY` | 32-character random string | Generated above |
| `POSTGRES_PASSWORD` | PostgreSQL password | Generated above |
| `CLICKHOUSE_PASSWORD` | ClickHouse password | Generated above or empty |

#### Optional Secrets (for pre-creating admin account)

| Secret Name | Description |
|-------------|-------------|
| `ADMIN_USER_EMAIL` | Admin user email address |
| `ADMIN_USER_NAME` | Admin user full name |
| `ADMIN_USER_PWD` | Admin user password |

#### Optional Secrets (for email notifications)

| Secret Name | Description |
|-------------|-------------|
| `SMTP_HOST_ADDR` | SMTP server address |
| `SMTP_HOST_PORT` | SMTP server port (e.g., 587) |
| `SMTP_USER_NAME` | SMTP username |
| `SMTP_USER_PWD` | SMTP password |
| `SMTP_HOST_SSL_ENABLED` | Enable SSL (true/false) |

#### Encoding KUBE_CONFIG

```bash
# Encode your kubeconfig file
cat ~/.kube/config | base64 -w 0

# Or from a specific kubeconfig file
cat /path/to/kubeconfig | base64 -w 0
```

### 3. Deploy to Kubernetes

Once all secrets are configured:

1. Push this repository to GitHub (or create a new repository with these files)
2. Push to the `main` branch to trigger automatic deployment
3. Monitor the deployment in the **Actions** tab

The deployment will:
- Create the `plausible` namespace
- Deploy PostgreSQL and wait for it to be ready
- Deploy ClickHouse and wait for it to be ready
- Deploy Plausible application
- Create Ingress with automatic TLS certificate

### 4. Access Plausible

After deployment completes:

1. Wait 2-5 minutes for cert-manager to issue the TLS certificate
2. Visit https://plausible.jankylabs.co.uk
3. If you configured admin credentials, log in with those
4. Otherwise, register a new account (if registration is enabled)

## Deployment Details

### Resource Allocation

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| PostgreSQL | 250m | 1000m | 256Mi | 1Gi |
| ClickHouse | 500m | 2000m | 512Mi | 2Gi |
| Plausible | 500m | 2000m | 512Mi | 2Gi |

### Storage

| Component | Size | Access Mode |
|-----------|------|-------------|
| PostgreSQL | 10Gi | ReadWriteOnce |
| ClickHouse | 20Gi | ReadWriteOnce |

### Namespace Isolation

This deployment is completely isolated in the `plausible` namespace and will not affect any other services in your cluster. It uses:
- Separate namespace
- Separate databases (not shared with other apps)
- Separate storage volumes
- Separate TLS certificate
- Separate Traefik route (based on hostname)

## Configuration

### Disable User Registration

By default, registration is set to "invite_only". To change this, edit `k8s/02-configmap.yaml`:

```yaml
DISABLE_REGISTRATION: "true"  # Completely disable registration
# or
DISABLE_REGISTRATION: "false"  # Allow open registration
# or
DISABLE_REGISTRATION: "invite_only"  # Invite-only (default)
```

### Adjust Storage Size

Edit the storage size in PVC manifests:

- PostgreSQL: `k8s/03-postgres-pvc.yaml`
- ClickHouse: `k8s/06-clickhouse-pvc.yaml`

### Adjust Resource Limits

Edit the resource requests/limits in:

- PostgreSQL: `k8s/04-postgres-statefulset.yaml`
- ClickHouse: `k8s/07-clickhouse-statefulset.yaml`
- Plausible: `k8s/09-plausible-deployment.yaml`

## Manual Deployment

If you prefer to deploy manually without GitHub Actions:

```bash
# Apply all manifests in order
kubectl apply -f k8s/01-namespace.yaml

# Create secrets manually
kubectl create secret generic plausible-secrets \
  --from-literal=BASE_URL="https://plausible.jankylabs.co.uk" \
  --from-literal=SECRET_KEY_BASE="your-64-char-secret" \
  --from-literal=TOTP_VAULT_KEY="your-32-char-secret" \
  --from-literal=POSTGRES_PASSWORD="your-postgres-password" \
  --from-literal=CLICKHOUSE_PASSWORD="" \
  --from-literal=DATABASE_URL="postgresql://plausible:your-postgres-password@postgres:5432/plausible" \
  --from-literal=CLICKHOUSE_DATABASE_URL="http://clickhouse:8123/plausible_events" \
  --namespace=plausible

# Create the ClickHouse users config (referenced by 07-clickhouse-statefulset.yaml).
# Not stored in this repo: it contains a <password> element and this repo is public.
kubectl create configmap clickhouse-users \
  --from-file=allow-remote.xml=./allow-remote.xml \
  --namespace=plausible

# Apply remaining manifests
kubectl apply -f k8s/ -n plausible

# Watch deployment
kubectl get pods -n plausible -w
```

## Rollback

To rollback to a previous version:

1. Go to **Actions** tab in GitHub
2. Select **Rollback Deployment** workflow
3. Click **Run workflow**
4. Optionally specify a revision number (or leave empty for previous)

## Troubleshooting

### Check pod status
```bash
kubectl get pods -n plausible
```

### View logs
```bash
# Plausible logs
kubectl logs -n plausible deployment/plausible -f

# PostgreSQL logs
kubectl logs -n plausible statefulset/postgres -f

# ClickHouse logs
kubectl logs -n plausible statefulset/clickhouse -f
```

### Check ingress
```bash
kubectl get ingress -n plausible
kubectl describe ingress plausible-ingress -n plausible
```

### Check certificate
```bash
kubectl get certificate -n plausible
kubectl describe certificate plausible-tls -n plausible
```

### Health check
```bash
kubectl exec -n plausible deployment/plausible -- curl http://localhost:8000/api/health
```

### Database connection test
```bash
# Test PostgreSQL
kubectl exec -n plausible statefulset/postgres -- psql -U plausible -c "SELECT 1"

# Test ClickHouse
kubectl exec -n plausible statefulset/clickhouse -- clickhouse-client --query "SELECT 1"
```

## Updating Plausible

The deployment uses the `latest` tag from `ghcr.io/plausible/community-edition`. To update:

1. Restart the deployment:
```bash
kubectl rollout restart deployment/plausible -n plausible
```

2. Or trigger a new GitHub Actions deployment by pushing to main

## Backup

### PostgreSQL Backup
```bash
kubectl exec -n plausible statefulset/postgres -- \
  pg_dump -U plausible plausible > plausible-backup.sql
```

### ClickHouse Backup
```bash
kubectl exec -n plausible statefulset/clickhouse -- \
  clickhouse-client --query "BACKUP DATABASE plausible_events TO Disk('default', 'backup')"
```

## Uninstall

To completely remove Plausible:

```bash
# Delete namespace (removes all resources)
kubectl delete namespace plausible

# Note: PVCs and PVs may need manual cleanup depending on your storage class
kubectl get pv | grep plausible
```

## License

This deployment configuration is provided as-is. Plausible Analytics is licensed under AGPL-3.0.

## Support

For Plausible-specific issues, see: https://plausible.io/docs
For deployment issues, open an issue in this repository.
