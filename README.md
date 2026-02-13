# Open WebUI Deployment Manifests

This repository contains Kubernetes deployment manifests for Open WebUI, a self-hosted AI chat interface.

## Architecture

- **Open WebUI**: Web interface for LLM interactions (ghcr.io/open-webui/open-webui:main)
- **PostgreSQL**: CloudNativePG cluster for persistent storage
- **Storage**: Ceph RBD block storage
- **Ingress**: Gateway API HTTPRoute via Cilium Gateway

## Components

- `helmrelease.yaml` - Helm release configuration for Open WebUI
- `cloudnativepg-cluster.yaml` - PostgreSQL cluster with 3 replicas
- `httproute.yaml` - Gateway API HTTPRoute for `open-webui.animaniacs.farh.net`
- `sealedsecrets.yaml` - Placeholder for sealed secrets (DB credentials, S3 backup credentials)
- `kustomization.yaml` - Kustomize configuration

## Secrets

Before deploying, seal the secrets in `sealedsecrets.yaml`:

```bash
# Generate strong password
DB_PASSWORD=$(openssl rand -base64 32)

# Create sealed DB credentials
kubectl create secret generic open-webui-db-credentials \
  --namespace=open-webui \
  --from-literal=username=openwebui \
  --from-literal=password="${DB_PASSWORD}" \
  --from-literal=uri="postgresql://openwebui:${DB_PASSWORD}@open-webui-postgres-rw.open-webui.svc.cluster.local:5432/openwebui" \
  --dry-run=client -o yaml | kubeseal --format yaml > /tmp/db-sealed.yaml

# Create sealed S3 backup credentials (use existing Ceph credentials)
kubectl create secret generic open-webui-s3-backup-credentials \
  --namespace=open-webui \
  --from-literal=access_key=YOUR_CEPH_ACCESS_KEY \
  --from-literal=secret_key=YOUR_CEPH_SECRET_KEY \
  --dry-run=client -o yaml | kubeseal --format yaml > /tmp/s3-sealed.yaml

# Merge sealed secrets into sealedsecrets.yaml
```

## Configuration

Open WebUI is configured via environment variables in `helmrelease.yaml`:

- `DATABASE_URL`: PostgreSQL connection string (from sealed secret)
- `WEBUI_NAME`: Display name for the web interface
- `ENABLE_SIGNUP`: Allow user self-registration
- `DEFAULT_USER_ROLE`: Default role for new users

Optional Ollama integration can be enabled by uncommenting the `OLLAMA_BASE_URL` environment variable.

## Access

Once deployed, Open WebUI will be available at:
- https://open-webui.animaniacs.farh.net

## Resources

- **Open WebUI**: 100m CPU request, 256Mi-512Mi memory
- **PostgreSQL**: 100m CPU request, 256Mi-512Mi memory per instance (3 instances)
- **Storage**: 5Gi for Open WebUI data, 10Gi per PostgreSQL instance
