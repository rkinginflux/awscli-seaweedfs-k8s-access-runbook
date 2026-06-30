# Runbook: Configure AWS CLI access to SeaweedFS S3 on local Kubernetes (LoadBalancer)

## Scope
This document captures the exact setup flow used to enable `aws` CLI access to a SeaweedFS S3 endpoint running in a local Kubernetes cluster.

## Environment
- Kubernetes context: `kubernetes-admin@kubernetes`
- Namespace: `s3`
- S3 service: `seaweedfs-s3`
- S3 API port: `8333`
- Load balancer provider: MetalLB
- Assigned service IP: `192.168.0.17`

## What was done

### 1) Verified cluster and SeaweedFS S3 service state
Commands used:
```bash
kubectl get svc -A
kubectl get pods -n s3 -o wide
kubectl get svc -n s3 seaweedfs-s3 -o wide
```

Result: `seaweedfs-s3` was initially `ClusterIP` and not externally reachable.

### 2) Installed AWS CLI
Command used:
```bash
brew install awscli
```

Verification:
```bash
aws --version
```

### 3) Confirmed SeaweedFS S3 IAM user/key state
Used SeaweedFS shell in filer pod to inspect users and access keys:
```bash
kubectl exec -n s3 seaweedfs-filer-0 -- sh -lc 'weed shell -master=seaweedfs-master-0.seaweedfs-master.s3:9333 <<"EOF"
s3.user.list
s3.accesskey.list -user admin
EOF'
```

Then created an additional admin access key for CLI use.

Security note:
- Do **not** publish real access keys or secrets in docs/commits.
- Rotate credentials if they were ever exposed.

### 4) Configured local AWS profile
Created profile `local-s3` in:
- `~/.aws/credentials`
- `~/.aws/config`

Template:
```ini
# ~/.aws/credentials
[local-s3]
aws_access_key_id = <YOUR_ACCESS_KEY>
aws_secret_access_key = <YOUR_SECRET_KEY>
```

```ini
# ~/.aws/config
[profile local-s3]
region = us-east-1
output = json
```

### 5) Diagnosed credential override issue
Observed failure:
- `InvalidAccessKeyId` despite correct profile content.

Root cause:
- Shell environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) were overriding profile credentials.

Fix:
```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
export AWS_PROFILE=local-s3
```

### 6) Initially validated via port-forward
Temporary access path:
```bash
kubectl port-forward -n s3 svc/seaweedfs-s3 8333:8333
```

Validation command:
```bash
aws --endpoint-url http://127.0.0.1:8333 s3 ls
```

### 7) Switched to LoadBalancer (permanent endpoint)
Service patch:
```bash
kubectl patch svc -n s3 seaweedfs-s3 --type merge -p '{"spec":{"type":"LoadBalancer"}}'
```

Verification:
```bash
kubectl get svc -n s3 seaweedfs-s3 -o wide
```

Result:
- Service type: `LoadBalancer`
- External endpoint: `http://192.168.0.17:8333`

### 8) Final AWS CLI verification against LoadBalancer endpoint
```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
export AWS_PROFILE=local-s3

aws --endpoint-url http://192.168.0.17:8333 s3 ls
aws --endpoint-url http://192.168.0.17:8333 s3 ls s3://lab/
aws --endpoint-url http://192.168.0.17:8333 s3 cp /tmp/file.txt s3://lab/file.txt
```

## Operational notes
- Keep endpoint explicit with `--endpoint-url` for S3-compatible targets.
- Prefer profile-based auth over env vars for repeatability.
- If commands unexpectedly fail auth, run:
```bash
aws configure list --profile local-s3
```
and check for env-var precedence.

## Quick start (copy/paste)
```bash
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
export AWS_PROFILE=local-s3
aws --endpoint-url http://192.168.0.17:8333 s3 ls
```
