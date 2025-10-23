# Zuul Base Job Configuration for MinIO Log Storage

This document describes the manual configuration required in the `ci-sean-mooney` config repository to enable automatic log uploads to MinIO object storage for all Zuul jobs.

## Overview

MinIO has been deployed as S3-compatible object storage for Zuul job logs. To enable automatic log uploads, you need to configure the base job in your `ci-sean-mooney` config repository to use the `upload-logs-s3` role from zuul-jobs.

## Configuration Repository

**Repository**: `SeanMooney/ci-sean-mooney` (GitHub)
**Type**: Config-project (trusted)
**Branch**: `main`

## Required Changes

### 1. Update Base Job Definition

Add the `upload-logs-s3` post-run playbook to your base job.

**File**: `zuul.d/jobs.yaml` (or wherever your base job is defined)

```yaml
- job:
    name: base
    parent: null
    description: Base job for all projects in the tenant
    pre-run:
      - playbooks/base/pre.yaml
    post-run:
      - playbooks/base/post.yaml
      - playbooks/base/upload-logs.yaml  # Add this for log uploads
    secrets:
      - name: zuul_log_targets
        secret: minio-logs-secret
    roles:
      - zuul: zuul/zuul-jobs  # Ensure zuul-jobs is available
```

**Key points**:
- `parent: null` - This is a base job (only allowed in config-projects)
- `upload-logs.yaml` - Post-run playbook that calls upload-logs-s3 role
- `zuul_log_targets` - Secret containing MinIO S3 configuration
- `roles: zuul: zuul/zuul-jobs` - Makes zuul-jobs roles available

### 2. Create Upload Logs Playbook

Create a new playbook to handle log uploads.

**File**: `playbooks/base/upload-logs.yaml`

```yaml
- hosts: all
  roles:
    - role: upload-logs-s3
```

**Alternative with failover** (if you add multiple storage backends later):

```yaml
- hosts: all
  roles:
    - role: upload-logs-failover
```

### 3. Verify zuul-jobs Integration

Ensure your tenant configuration includes `zuul/zuul-jobs` as an untrusted-project (already configured):

**Repository**: `zuul-homelab`
**File**: `apps/zuul/configs/tenant-config.yaml`

```yaml
- tenant:
    name: main
    source:
      github:
        config-projects:
          - SeanMooney/ci-sean-mooney:
              load-branch: main
        untrusted-projects:
          - SeanMooney/ci-jobs
      opendev:
        untrusted-projects:
          - zuul/zuul-jobs  # ✓ Already configured
```

## MinIO Configuration

The following configuration is already deployed in the `zuul-homelab` repository:

**Secret**: `minio-logs-secret` (namespace: `zuul-system`)

```yaml
zuul_log_targets:
  - target: s3
    args:
      upload_logs_s3_endpoint: https://minio-api.teim.app
      zuul_log_bucket: zuul-logs
      zuul_log_aws_access_key: zuul-upload
      zuul_log_aws_secret_key: <encrypted>
      zuul_log_create_indexes: true
```

**Bucket**: `zuul-logs`
**Access Policy**: Public download (anonymous read access)
**Service Account**: `zuul-upload` with upload permissions

## Log URL Format

After successful upload, logs will be available at:

```
https://minio-api.teim.app/zuul-logs/{build.uuid}/
```

Example:
```
https://minio-api.teim.app/zuul-logs/a1b2c3d4-e5f6-7890-abcd-ef1234567890/
```

## Testing the Configuration

### 1. Commit Changes to Config Repository

```bash
cd ci-sean-mooney/
git add zuul.d/jobs.yaml playbooks/base/upload-logs.yaml
git commit -m "Configure base job to upload logs to MinIO S3"
git push
```

### 2. Wait for Zuul to Reload Configuration

Zuul scheduler will automatically reload the tenant configuration when it detects changes in the config-project.

Monitor the scheduler logs:
```bash
kubectl logs -n zuul-system statefulset/zuul-scheduler --tail=50 -f
```

Look for messages like:
```
INFO zuul.Scheduler: Reconfiguration complete
INFO zuul.Scheduler: Reconfigured tenants: ['main']
```

### 3. Trigger a Test Job

Create a simple test job in `ci-jobs` repository:

**File**: `zuul.d/jobs.yaml`

```yaml
- job:
    name: test-log-upload
    parent: base
    description: Test job to verify log uploads to MinIO
    run: playbooks/test.yaml
    nodeset:
      nodes:
        - name: test-node
          label: ubuntu-22-04-pod
```

**File**: `playbooks/test.yaml`

```yaml
- hosts: all
  tasks:
    - name: Generate test output
      shell: |
        echo "Testing log upload to MinIO"
        echo "Build UUID: {{ zuul.build }}"
        echo "This output should appear in MinIO logs"

    - name: Create some log files
      shell: |
        mkdir -p {{ ansible_user_dir }}/logs
        echo "Test log content" > {{ ansible_user_dir }}/logs/test.log
        date > {{ ansible_user_dir }}/logs/timestamp.log
```

### 4. Verify Log Upload

After the job completes:

1. **Check job output** for upload-logs-s3 execution:
   ```
   TASK [upload-logs-s3 : Upload logs to S3]
   ```

2. **Access logs via MinIO**:
   ```
   https://minio-api.teim.app/zuul-logs/<build-uuid>/
   ```

3. **Check MinIO console** (optional):
   - URL: `https://minio-console.teim.app`
   - Username: `minio-admin`
   - Password: Retrieved from secret

   To get the password:
   ```bash
   kubectl get secret -n minio-system minio-root-credentials \
     -o jsonpath='{.data.root-password}' | base64 -d
   ```

## Troubleshooting

### Logs Not Uploading

1. **Check secret is available to jobs**:
   ```bash
   kubectl get secret -n zuul-system minio-logs-secret
   ```

2. **Verify job has access to secret**:
   Check job output for errors like:
   ```
   ERROR: Secret 'minio-logs-secret' not found
   ```

3. **Check MinIO connectivity**:
   ```bash
   kubectl exec -it -n zuul-system deployment/zuul-executor-0 -- \
     curl -I https://minio-api.teim.app
   ```

### Upload Fails with 403 Forbidden

The `zuul-upload` service account may not have correct permissions. Verify with MinIO console or recreate the init job:

```bash
kubectl delete job -n minio-system minio-init
kubectl create -f infrastructure/minio/init-job.yaml
```

### Logs Uploaded But Not Accessible

Check bucket policy allows public downloads:

```bash
kubectl exec -it -n minio-system minio-0 -- \
  mc anonymous get-json minio/zuul-logs
```

Expected: `"Effect": "Allow"` with `"Principal": "*"`

## Advanced Configuration

### Custom Log Path

To customize the S3 path structure, add to `zuul_log_targets`:

```yaml
zuul_log_targets:
  - target: s3
    args:
      # ... existing config ...
      zuul_log_path: "{{ zuul.tenant }}/{{ zuul.project.name }}/{{ zuul.build }}"
```

### Log Retention / Lifecycle Policy

Configure automatic deletion of old logs via MinIO console or mc client:

```bash
kubectl exec -it -n minio-system minio-0 -- sh

# Inside the container
mc alias set local http://localhost:9000 "$MINIO_ROOT_USER" "$MINIO_ROOT_PASSWORD"

# Set lifecycle rule to delete logs after 90 days
mc ilm add --expiry-days 90 local/zuul-logs
```

### Multiple Upload Targets (Failover)

To add redundancy, configure multiple storage backends:

```yaml
zuul_log_targets:
  - target: s3
    args:
      upload_logs_s3_endpoint: https://minio-api.teim.app
      zuul_log_bucket: zuul-logs
      # ... credentials ...
  - target: swift
    args:
      # Swift configuration as backup
```

Then use `upload-logs-failover` role instead of `upload-logs-s3`.

## References

- [Zuul upload-logs-s3 role documentation](https://zuul-ci.org/docs/zuul-jobs/latest/log-roles.html#role-upload-logs-s3)
- [MinIO S3 API compatibility](https://min.io/docs/minio/linux/developers/python/API.html)
- [Zuul job content documentation](https://zuul-ci.org/docs/zuul/latest/job-content.html)
