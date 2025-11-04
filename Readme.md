# MinIO Object Storage - Complete Setup Guide

> **Production-Grade MinIO Deployment with IAM & Service Account Configuration**

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Status](https://img.shields.io/badge/status-production--ready-brightgreen.svg)
![Docker](https://img.shields.io/badge/docker-compose-blue.svg)
![MinIO](https://img.shields.io/badge/minio-latest-orange.svg)

## 📋 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step-by-Step Deployment](#step-by-step-deployment)
- [CI/CD Integration](#cicd-integration)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Maintenance](#maintenance)
- [Support & Contributing](#support--contributing)

---

## Overview

Complete deployment guide for **MinIO** object storage with Docker Compose, bucket creation, user management, and service account key generation. This guide provides production-ready configuration for S3-compatible object storage with fine-grained access control.

**Key Features:**
- 🐳 Docker Compose orchestration
- 🔐 IAM policy-based bucket access control
- 🔑 Service account credentials for CI/CD integration
- 📊 Audit logging and monitoring
- 🛡️ Enterprise-grade security hardening

**Technology Stack:**
- MinIO (S3-compatible object storage)
- Docker & Docker Compose
- Ubuntu/Linux (tested on Ubuntu 20.04+)

---

## Prerequisites

**System Requirements:**
- Docker Engine 20.10 or higher
- Docker Compose v2.0 or higher
- 4GB RAM minimum, 8GB recommended
- 20GB disk space minimum 
- Root or sudo access

**Network:**
- IP: `192.168.11.12` (adjustable)
- Ports: `9000` (API), `9001` (Console)

**Installation Check:**
```bash
docker --version
docker compose --version
```

---

## Step-by-Step Deployment

### Step 1: Install MinIO Client (mc)

MinIO CLI client for server management and operations.

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
mc --version
```

**Expected Output:**
```
mc version RELEASE.2025-11-04T...
```

---

### Step 2: Deploy MinIO Server

#### Create Directory Structure

```bash
sudo mkdir -p /opt/minio
cd /opt/minio
```

#### Create Docker Compose File

```bash
sudo nano docker-compose.yml
```

**Paste the following:**

```yaml
services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    ports:
      - 9000:9000
      - 9001:9001
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: stongPassword
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"

volumes:
  minio_data:
```

**Save and exit:** `Ctrl+X`, then `Y`, then `Enter`

#### Start MinIO Server

```bash
sudo docker compose up -d
```

**Check Status:**
```bash
docker ps | grep minio
```

**Expected Output:**
```
CONTAINER ID   IMAGE                                COMMAND             PORTS                            NAMES
abc123def456   quay.io/minio/minio:latest          "minio server..."   0.0.0.0:9000->9000/tcp         minio
```

#### Verify Deployment Health

```bash
curl http://192.168.11.12:9000/minio/health/live
```

**Expected Output:**
```json
{"status":"ok"}
```

---

### Step 3: Configure MinIO Client Alias

Establish connection to MinIO server with CLI.

```bash
mc alias set myminio http://192.168.11.12:9000 admin stongPassword --insecure
```

**Verify Connection:**

```bash
mc admin info myminio
```

**Expected Output (sample):**
```
●  192.168.11.12:9000
   Uptime: 2 minutes
   Version: 2025-11-04T...
   Network In/Out: 0B / 0B
```

---

### Step 4: Create Build Artifacts Bucket

Create object storage bucket for CI/CD artifacts.

```bash
mc mb myminio/build-artifacts
```

**Verify Bucket Creation:**

```bash
mc ls myminio
```

**Expected Output:**
```
[2025-11-04 14:38:27 +06]     0B build-artifacts/
```

---

### Step 5: Create IAM Policy for Bucket Access

Define granular access permissions for bucket operations.

#### Create Policy JSON File

```bash
nano build-artifacts-rw-policy.json
```

**Paste the following:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::build-artifacts"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::build-artifacts/*"
    }
  ]
}
```

**Save and exit:** `Ctrl+X`, then `Y`, then `Enter`

#### Create Policy in MinIO

```bash
mc admin policy create myminio build-artifacts-rw build-artifacts-rw-policy.json
```

**Verify Policy Creation:**

```bash
mc admin policy ls myminio
```

**Check Policy Details:**

```bash
mc admin policy info myminio build-artifacts-rw
```

**Policy Permissions:**
- ✅ List bucket contents
- ✅ Download objects (GetObject)
- ✅ Upload objects (PutObject)
- ✅ Delete objects (DeleteObject)
- ❌ Create new buckets (restricted)
- ❌ Delete buckets (restricted)

---

### Step 6: Create IAM User

Create application user for service account.

```bash
mc admin user add myminio sctdev stongPassword
```

**Verify User Creation:**

```bash
mc admin user list myminio
```

**Expected Output (sample):**
```
sctdev
```

---

### Step 7: Attach Policy to User

Bind IAM policy to user account.

```bash
mc admin policy attach myminio build-artifacts-rw --user sctdev
```

**Verify Policy Attachment:**

```bash
mc admin user info myminio sctdev
```

**Expected Output (includes):**
```
Policy: build-artifacts-rw
```

---

### Step 8: Generate Service Account Keys

Create API credentials for programmatic access (CI/CD pipelines).

#### Option A: Auto-Generate Keys (Recommended ⭐)

```bash
mc admin user svcacct add myminio sctdev \
  --policy build-artifacts-rw \
  --name "build-artifact-service" \
  --description "Access to build-artifact bucket only"
```

**Sample Output:**
```
AccessKey: K7X9MPLE2ABC3DEF
SecretKey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

⚠️ **CRITICAL: Save these credentials immediately** — they are shown only once and cannot be retrieved later.

#### Option B: Custom Keys (Manual)

```bash
mc admin user svcacct add myminio sctdev \
  --access-key "BUILD-ARTIFACT-SVC" \
  --secret-key "MySecretKey123456789" \
  --policy build-artifacts-rw \
  --name "build-artifact-service" \
  --description "Access to build-artifact bucket only"
```

**Key Requirements:**
- Access Key: 3-20 characters (alphanumeric)
- Secret Key: 8-40 characters (alphanumeric + special chars)

---

### Step 9: Verify Service Account

#### List Service Accounts

```bash
mc admin user svcacct ls myminio sctdev
```

#### Check Service Account Details

```bash
mc admin user svcacct info myminio <ACCESS_KEY> --policy
```

Replace `<ACCESS_KEY>` with your generated access key.

---

### Step 10: Test Access with Service Account

Validate that service account has correct bucket permissions.

#### Configure Test Alias

```bash
mc alias set cicd http://192.168.11.12:9000 <ACCESS_KEY> <SECRET_KEY> --insecure
```

Replace:
- `<ACCESS_KEY>`: Your generated access key
- `<SECRET_KEY>`: Your generated secret key

#### Test Operations

**✅ Upload test file:**
```bash
echo "test content" > testfile.txt
mc cp testfile.txt cicd/build-artifacts/
```

**✅ List bucket contents:**
```bash
mc ls cicd/build-artifacts
```

**✅ Download file:**
```bash
mc cat cicd/build-artifacts/testfile.txt
```

**✅ Delete file:**
```bash
mc rm cicd/build-artifacts/testfile.txt
```

**❌ Verify limited access (should fail):**
```bash
mc mb cicd/unauthorized-bucket
```

**Expected:** `Permission denied` (proves bucket isolation works)

---

### Step 11: Access Web Console (Optional)

MinIO web UI for bucket management and monitoring.

**URL:** `http://192.168.11.12:9001`

**Login Credentials:**
- Username: `admin`
- Password: `stongPassword`

⚠️ **Note:** Service accounts cannot login to the web console — they are API-only credentials.

---

## Complete Command Summary

Run these commands in sequence for rapid deployment:

```bash
# 1️⃣ Install MinIO Client
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# 2️⃣ Deploy MinIO
sudo mkdir -p /opt/minio
cd /opt/minio
# Create docker-compose.yml file (see Step 2)
sudo docker compose up -d

# 3️⃣ Configure alias
mc alias set myminio http://192.168.11.12:9000 admin stongPassword --insecure

# 4️⃣ Create bucket
mc mb myminio/build-artifacts

# 5️⃣ Create policy file (build-artifacts-rw-policy.json) and apply
mc admin policy create myminio build-artifacts-rw build-artifacts-rw-policy.json

# 6️⃣ Create user
mc admin user add myminio sctdev stongPassword

# 7️⃣ Attach policy
mc admin policy attach myminio build-artifacts-rw --user sctdev

# 8️⃣ Generate service account keys
mc admin user svcacct add myminio sctdev \
  --policy build-artifacts-rw \
  --name "build-artifact-service" \
  --description "Access to build-artifact bucket only"

# 9️⃣ Test access
mc alias set cicd http://192.168.11.12:9000 <ACCESS_KEY> <SECRET_KEY> --insecure
mc ls cicd/build-artifacts
```

---

## CI/CD Integration

### Bitbucket Pipelines

```yaml
pipelines:
  default:
    - step:
        name: Upload Build Artifacts
        script:
          - apt-get update && apt-get install -y wget
          - wget -q https://dl.min.io/client/mc/release/linux-amd64/mc
          - chmod +x mc
          - ./mc alias set storage http://192.168.11.12:9000 $MINIO_ACCESS_KEY $MINIO_SECRET_KEY --insecure
          - ./mc cp build-output.tar.gz storage/build-artifacts/${BITBUCKET_BUILD_NUMBER}/
          - echo "Artifact uploaded successfully"
```

**Repository Variables (Settings → Pipelines → Repository variables):**
- `MINIO_ACCESS_KEY`: Your generated access key
- `MINIO_SECRET_KEY`: Your generated secret key (secured)

### GitLab CI

```yaml
stages:
  - upload

upload_artifacts:
  stage: upload
  script:
    - wget -q https://dl.min.io/client/mc/release/linux-amd64/mc
    - chmod +x mc
    - ./mc alias set storage http://192.168.11.12:9000 $MINIO_ACCESS_KEY $MINIO_SECRET_KEY --insecure
    - ./mc cp build-output.tar.gz storage/build-artifacts/${CI_PIPELINE_ID}/
  only:
    - main
```

**GitLab CI/CD Variables (Settings → CI/CD → Variables):**
- `MINIO_ACCESS_KEY`: Your generated access key
- `MINIO_SECRET_KEY`: Your generated secret key (masked)

### GitHub Actions

```yaml
name: Upload to MinIO

on:
  push:
    branches:
      - main

jobs:
  upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install MinIO Client
        run: |
          wget -q https://dl.min.io/client/mc/release/linux-amd64/mc
          chmod +x mc
          sudo mv mc /usr/local/bin/
      - name: Upload Artifacts
        env:
          MINIO_ACCESS_KEY: ${{ secrets.MINIO_ACCESS_KEY }}
          MINIO_SECRET_KEY: ${{ secrets.MINIO_SECRET_KEY }}
        run: |
          mc alias set storage http://192.168.11.12:9000 $MINIO_ACCESS_KEY $MINIO_SECRET_KEY --insecure
          mc cp build-output.tar.gz storage/build-artifacts/${{ github.run_id }}/
```

**GitHub Secrets (Settings → Secrets → Actions):**
- `MINIO_ACCESS_KEY`: Your generated access key
- `MINIO_SECRET_KEY`: Your generated secret key

---

## Troubleshooting

### Issue: Connection Refused

**Check Docker status:**
```bash
docker ps
docker logs minio --tail 50
```

**Restart MinIO:**
```bash
cd /opt/minio
sudo docker compose restart
```

### Issue: Policy Not Attaching

**Verify policy exists:**
```bash
mc admin policy ls myminio
mc admin policy info myminio build-artifacts-rw
```

**Re-attach policy:**
```bash
mc admin policy attach myminio build-artifacts-rw --user sctdev
```

### Issue: Service Account Key Length Error

**Error:** `access key length should be between 3 and 20`

**Solution:** Use auto-generation or ensure:
- Access Key: 3-20 characters
- Secret Key: 8-40 characters

### Issue: Permission Denied

**Check user policies:**
```bash
mc admin user info myminio sctdev
```

**Check service account policies:**
```bash
mc admin user svcacct info myminio <ACCESS_KEY> --policy
```

### Issue: Bucket Not Accessible

**Verify bucket exists:**
```bash
mc ls myminio
```

**Check permissions:**
```bash
mc admin user info myminio sctdev
```

---

## Security Best Practices

### ✅ Production Checklist

- [ ] **Change default credentials** — Update `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`
- [ ] **Enable TLS/SSL** — Remove `--insecure` flag, implement SSL certificates
- [ ] **Rotate keys** — Cycle service account keys every 90 days
- [ ] **Network hardening** — Restrict ports 9000/9001 via firewall rules
- [ ] **Audit logging** — Enable MinIO audit logs for compliance
- [ ] **Strong passwords** — Minimum 12+ characters, mixed case & special chars
- [ ] **Principle of least privilege** — Limit service account permissions
- [ ] **Monitor access logs** — Review logs weekly for anomalies
- [ ] **Backup credentials** — Store keys in secure vault (HashiCorp Vault, AWS Secrets Manager)
- [ ] **Disable console access** — For CI/CD accounts, use API-only credentials

### Security Hardening Commands

**Enable audit logging:**
```bash
mc admin config set myminio audit/targets webhook=http://logger-server:8000/minio/audit
```

**Rotate service account:**
```bash
mc admin user svcacct rm myminio <OLD_ACCESS_KEY>
mc admin user svcacct add myminio sctdev --policy build-artifacts-rw
```

---

## Maintenance

### Backup Bucket Data

```bash
mc mirror myminio/build-artifacts /backup/build-artifacts --overwrite
```

### Restore Bucket Data

```bash
mc mirror /backup/build-artifacts myminio/build-artifacts --overwrite
```

### View Storage Usage

```bash
mc admin info myminio
mc du myminio/build-artifacts --recursive
```

### Delete Old Artifacts (older than 30 days)

```bash
mc rm --recursive --force --older-than 30d myminio/build-artifacts/
```

### Monitor Bucket Growth

```bash
watch -n 5 'mc du myminio/build-artifacts'
```

### Performance Tuning

**Check resource usage:**
```bash
docker stats minio
```

---

## File Structure

```
/opt/minio/
├── docker-compose.yml              # MinIO deployment configuration
├── build-artifacts-rw-policy.json  # IAM bucket access policy
└── minio_data/                     # Persistent volume (managed by Docker)

/home/sctapp/
└── build-artifacts-rw-policy.json  # Policy backup location
```

---

## Support & Contributing

**Contributing:**
1. Fork the repository
2. Create feature branch (`git checkout -b feature/enhancement`)
3. Commit changes (`git commit -am 'Add enhancement'`)
4. Push to branch (`git push origin feature/enhancement`)
5. Submit pull request

---

## References

- [MinIO Official Documentation](https://min.io/docs/minio/linux/index.html)
- [MinIO Client (mc) Reference Guide](https://min.io/docs/minio/linux/reference/minio-mc.html)
- [AWS S3 IAM Policy Reference](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html)
- [Docker Compose Documentation](https://docs.docker.com/compose/)

---

## License

This project is licensed under the **MIT License** — see [LICENSE](LICENSE) file for details.

```
MIT License

Copyright (c) 2025 Abubakkar Khan

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.
```

---

## Changelog

### [1.0.0] - 2025-11-04
- ✨ Initial production release
- ✅ Complete deployment guide with 11-step process
- ✅ IAM policy and service account configuration
- ✅ CI/CD integration examples (Bitbucket, GitLab, GitHub Actions)
- ✅ Security hardening guidelines
- ✅ Comprehensive troubleshooting section

---

## Authors & Contributors

**Maintained by:** Abubakkar Khan

- **Lead:** Abubakkar Khan (System Engineer, Cybersecurity Architect)
- **Team:** SCT DevOps & Infrastructure

---

**Environment:** Production  
**Last Updated:** November 4, 2025  
**Status:** ✅ Stable & Production-Ready

---

*For the latest updates, visit: https://github.com/schertech/minio-setup*