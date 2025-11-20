# MinIO Setup

## Credentials Issue

If you're experiencing permission issues (cannot delete buckets, set policies, or get tokens), it's likely because MinIO was initialized with different credentials than what's in the secret.

MinIO stores root credentials in its data directory at first initialization. Changing the secret after MinIO has been initialized won't automatically update the credentials.

## Solution: Reset MinIO (if data loss is acceptable)

**⚠️ WARNING: This will delete all data in MinIO!**

1. Delete the StatefulSet (this will delete the pod):
   ```bash
   kubectl delete statefulset minio -n minio
   ```

2. Delete the PVC to remove all data:
   ```bash
   kubectl delete pvc minio-data-minio-0 -n minio
   ```

3. Reapply the StatefulSet:
   ```bash
   kubectl apply -f k8s/production/infras/minio/statefulset.yaml
   ```

4. MinIO will reinitialize with the credentials from the secret (`minio.admin` / `minio.password`)

## Alternative: Update Credentials Without Data Loss

If you need to keep existing data, you can update credentials using MinIO's admin API:

1. Access MinIO console at `https://minio.washgo247.com`
2. Use the current root credentials to log in
3. Go to Identity → Users → select the root user → Change Password
4. Or use MinIO Client (mc) to update credentials

## Current Credentials

- **Username**: `minio.admin` (from secret)
- **Password**: `minio.password` (from secret)

These credentials are base64 encoded in `secret.yaml`:
- `MINIO_ROOT_USER`: `bWluaW8uYWRtaW4=`
- `MINIO_ROOT_PASSWORD`: `bWluaW8ucGFzc3dvcmQ=`

## Verify Credentials

To verify what credentials MinIO is actually using, check the pod logs:
```bash
kubectl logs -n minio statefulset/minio
```

Look for lines showing the root user that MinIO initialized with.

