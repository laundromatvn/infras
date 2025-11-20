# MinIO Credentials Guide

## Current Situation

Your MinIO was likely initialized with **default credentials** before the secret was properly configured.

## Try These Credentials First

**Default MinIO credentials (most likely):**
- **Username**: `minioadmin`
- **Password**: `minioadmin`

Try logging into `https://minio.washgo247.com` with these credentials.

## Credentials in Secret

The secret file contains:
- **Username**: `minio.admin` (base64: `bWluaW8uYWRtaW4=`)
- **Password**: `minio.password` (base64: `bWluaW8ucGFzc3dvcmQ=`)

However, these credentials are only used during **first initialization**. If MinIO was already running when you set the secret, it will continue using the old credentials stored in `/data/.minio.sys/`.

## Solution: Reset MinIO to Use Secret Credentials

If you want to use the credentials from the secret (`minio.admin` / `minio.password`), you need to reset MinIO:

**⚠️ WARNING: This will delete all data!**

```bash
# 1. Delete the StatefulSet
kubectl delete statefulset minio -n minio

# 2. Delete the PVC (this deletes all data)
kubectl delete pvc minio-data-minio-0 -n minio

# 3. Recreate the StatefulSet
kubectl apply -f k8s/production/infras/minio/statefulset.yaml

# 4. Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=minio -n minio --timeout=300s
```

After reset, you can login with:
- **Username**: `minio.admin`
- **Password**: `minio.password`

## Alternative: Update Secret to Match Current Credentials

If you want to keep existing data and use the current credentials, update the secret to match what MinIO is using (likely `minioadmin`/`minioadmin`):

```bash
# Encode the credentials
echo -n "minioadmin" | base64
echo -n "minioadmin" | base64

# Then update secret.yaml with the new base64 values
```

