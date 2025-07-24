# PostgreSQL with TLS using cert-manager

This guide shows how to deploy PostgreSQL with automatic TLS certificate management using cert-manager.

## Prerequisites

- cert-manager installed in the cluster
- A ClusterIssuer configured (update the issuerRef.name in postgresql-certificate.yaml)
- DNS record pointing postgresql.padmini.systems to your node IP

## Deployment Steps

### 1. Deploy PostgreSQL Image Catalog (via ArgoCD)
```bash
kubectl apply -f argocd-apps/postgresql-image-catalog.yaml
```

### 2. Deploy CloudNativePG Operator (via ArgoCD)
```bash
kubectl apply -f argocd-apps/cloudnative-pg-operator.yaml
```

### 2. Create PostgreSQL credentials secret
```bash
kubectl create secret generic postgresql-credentials \
  --namespace=padmini-pg \
  --from-literal=username=app_user \
  --from-literal=password="$(openssl rand -base64 32)" \
  --from-literal=postgres="$(openssl rand -base64 32)"
```

### 3. Deploy Certificate for TLS
```bash
kubectl apply -f examples/postgresql-certificate.yaml
```

### 4. Wait for certificate to be ready
```bash
kubectl wait --for=condition=Ready certificate/postgresql-tls-cert -n padmini-pg --timeout=300s
```

### 5. Deploy PostgreSQL cluster
```bash
kubectl apply -f examples/postgresql-cluster.yaml
```

### 6. Create external access service
```bash
kubectl apply -f examples/postgresql-nodeport-service.yaml
```

## Verification

### Check certificate status
```bash
kubectl get certificate -n padmini-pg
kubectl describe certificate postgresql-tls-cert -n padmini-pg
```

### Check PostgreSQL cluster
```bash
kubectl get clusters -n padmini-pg
kubectl get pods -n padmini-pg
```

### Test SSL connection
```bash
# Get the connection details
kubectl get secret postgresql-credentials -n padmini-pg -o jsonpath='{.data.password}' | base64 -d

# Connect with SSL (replace PASSWORD with the output from above)
psql "postgresql://app_user:PASSWORD@postgresql.padmini.systems:30432/myapp_db?sslmode=require"
```

## Connection Examples

### Using psql
```bash
psql "host=postgresql.padmini.systems port=30432 user=app_user dbname=myapp_db sslmode=require"
```

### Using connection string
```bash
postgresql://app_user:password@postgresql.padmini.systems:30432/myapp_db?sslmode=require
```

### Using applications
```yaml
# Environment variables for applications
POSTGRES_HOST: postgresql.padmini.systems
POSTGRES_PORT: 30432
POSTGRES_DB: myapp_db
POSTGRES_USER: app_user
POSTGRES_SSL_MODE: require
```

## Key Benefits

- ✅ Automatic certificate management and renewal via cert-manager
- ✅ Public CA certificates (no custom CA needed)
- ✅ Standard SSL/TLS connections
- ✅ CloudNativePG handles certificate mounting and PostgreSQL configuration
- ✅ Secure external access via NodePort
