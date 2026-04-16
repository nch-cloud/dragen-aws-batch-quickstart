# DRAGEN on AWS — Infrastructure Documentation

## Overview

This repository is an AWS Quick Start (Partner Solution) for deploying Illumina's **DRAGEN** (Dynamic Read Analysis for GENomics) platform on AWS. It uses CloudFormation to set up an AWS Batch environment that runs DRAGEN genomics analysis jobs on FPGA-accelerated EC2 instances (F1 family — `f2.6xlarge` or `f2.12xlarge`).

---

## Deployment Architecture

### Template Chain

The templates form a nested stack hierarchy. There are two entry points:

| Entry Point | Use Case |
|---|---|
| `dragen-main.template.yaml` | Deploys into a **new VPC** (creates VPC first, then calls `dragen.template.yaml`) |
| `dragen.template.yaml` | Deploys into an **existing VPC** — this is the core orchestrator |

### Nested Stack Hierarchy (inside `dragen.template.yaml`)

```
dragen.template.yaml
├── docker-bucket-repository.template.yaml   → Creates ECR repo + S3 artifact bucket
├── copy.template.yaml                       → Lambda copies dragen.zip from QS S3 bucket to artifact bucket
├── container-build.template.yaml            → CodePipeline + CodeBuild builds Docker image, pushes to ECR
├── batch.template.yaml                      → Creates AWS Batch compute environments, job queue, job definition
└── clean-bucket-repository.template.yaml    → Cleanup resources on stack deletion
    ├── clean-bucket.template.yaml           → Lambda empties S3 bucket
    └── clean-repository.template.yaml       → Lambda deletes ECR repository
```

A CloudFormation `WaitCondition` ensures the Docker image is fully built before the Batch environment is created.

---

## AMI Configuration

The AMI is a **DRAGEN Marketplace AMI**, hardcoded per region in `dragen.template.yaml` under the `AWSAMIRegionMap` mapping:

| Region | AMI ID |
|---|---|
| us-east-1 | `ami-0b200106f98d8acd2` |
| us-west-2 | `ami-07a7d63c4157030dc` |
| eu-west-2 | `ami-0ece6aa6f1f8d79f2` |
| ap-southeast-2 | `ami-0ffc6f405569e4661` |

- **Marketplace product code:** `9uotaksivr7km6tn0wa0sy2fw`
- The AMI comes pre-loaded with DRAGEN FPGA binaries, the `/opt/edico` directory, and FPGA libraries.
- The DRAGEN shared library version is hardcoded in the Batch job definition volume mount as `/usr/lib64/libdragen.so.4.4.4`.

### How to Change the AMI

1. Find the new AMI IDs via the AWS Marketplace or by running:
   ```bash
   aws ec2 describe-images --filters "Name=product-code,Values=9uotaksivr7km6tn0wa0sy2fw"
   ```
2. Update the `AWSAMIRegionMap` mapping in `dragen.template.yaml`.
3. Check if the DRAGEN library version changed — update the volume mount path in `batch.template.yaml` if needed (e.g., `libdragen.so.4.4.4` → `libdragen.so.X.Y.Z`).

---

## Docker Container

### Build Process

During deployment:

1. `dragen.zip` (located at `app/packages/dragen/dragen.zip`) is copied to the artifact S3 bucket.
2. CodePipeline picks it up; CodeBuild builds the Docker image using the Dockerfile at `app/source/dragen/Dockerfile`.
3. The image is pushed to the ECR repository.

### Container Contents

- **Base image:** OracleLinux 8
- **Runtime:** Python 3.12, boto3, requests
- **Entry point:** `dragen_qs.py`
- **Supporting scripts:**
  - `d_haul` — file transfer utility (S3 download/upload/import, HTTP downloads, multipart support)
  - `scheduler/aws_utils.py` — S3 operations (parallel directory downloads, uploads with server-side encryption)
  - `scheduler/scheduler_utils.py` — directory creation, time utilities
  - `scheduler/logger.py` — logging to file/stdout/syslog

---

## How a Job Runs (`dragen_qs.py`)

When AWS Batch runs a job, it launches the Docker container on an F1 instance. The container executes the following steps:

1. **Parse arguments** — processes the DRAGEN command-line arguments passed to the Batch job
2. **Download reference hash tables** from S3 (supports both directory prefixes and `.tar` archives)
3. **Download input files** (FASTQ, BAM, CRAM, BED files, etc.) from S3 or HTTP URLs
4. **Initialize FPGA** if needed (partial reconfig via `/opt/edico/bin/dragen --partial-reconfig`)
5. **Check/reset DRAGEN board state** (`/opt/edico/bin/dragen_reset`)
6. **Run DRAGEN** — executes `/opt/edico/bin/dragen` with the processed arguments
7. **Upload results** back to S3
8. **Clean up** local output directories

---

## AWS Batch Configuration

### Compute Environments

| Environment | Type | Priority | Notes |
|---|---|---|---|
| `dragen-ondemand` | EC2 On-Demand | 1 (preferred) | Reliable, higher cost |
| `dragen-spot` | Spot Instances | 2 (fallback) | Cost-optimized, configurable bid percentage (default 50%) |

### Instance Types

| Instance Type | vCPUs | Memory | FPGA |
|---|---|---|---|
| `f2.6xlarge` | 24 | 240 GB | Yes |
| `f2.12xlarge` | 48 | 480 GB | Yes |

### Job Queue

- Name: `dragen-queue`
- Tries On-Demand first, then falls back to Spot.

### Launch Template

The launch template UserData:
- Configures the ECS agent with Docker cleanup policies
- Downgrades Docker to version `25.0.5` (compatibility fix)
- Pulls and restarts the ECS agent

### Security Group

- Allows **HTTPS egress only** (port 443, `0.0.0.0/0`)
- If DRAGEN needs to reach other services, additional egress rules must be added.

---

## IAM Roles

### Batch Roles (in `batch.template.yaml`)

| Role | Trust Principal | Purpose |
|---|---|---|
| `BatchServiceRole` | `batch.amazonaws.com` | Lets AWS Batch manage compute resources. Uses managed policy `AWSBatchServiceRole`. |
| `SpotFleetRole` | `spotfleet.amazonaws.com` | Lets EC2 Spot Fleet tag instances. Uses managed policy `AmazonEC2SpotFleetTaggingRole`. |
| `DragenInstanceRole` | `ec2.amazonaws.com` | EC2 instance profile for Batch compute instances. Has `AmazonEC2ContainerServiceforEC2Role` + S3 read/write to genomics bucket. |
| `DragenJobRole` | `ecs-tasks.amazonaws.com` | ECS task role (what the Docker container runs as). Has S3 read/write to genomics bucket. |

### S3 Permissions on DragenInstanceRole and DragenJobRole

Both roles get identical S3 permissions scoped to the `GenomicsS3Bucket`:

- `s3:GetBucketLocation`
- `s3:ListBucket`
- `s3:ListBucketVersions`
- `s3:GetObject`
- `s3:GetObjectVersion`
- `s3:PutObject`
- `s3:ListMultipartUploadParts`
- `s3:AbortMultipartUpload`

**To add access to additional S3 buckets or AWS services:** Add policies to `DragenInstanceRole` (for host-level access) or `DragenJobRole` (for container-level access).

### CI/CD Roles (in `container-build.template.yaml`)

| Role | Purpose |
|---|---|
| `CodeBuildServiceRole` | Lets CodeBuild write logs, pull/push ECR images, read/write artifact S3 bucket. |
| `CodePipelineServiceRole` | Lets CodePipeline orchestrate the build. Has broad S3 permissions on artifact bucket, CodeBuild start/get, IAM PassRole, CloudWatch Logs. |

### Utility Roles

| Role | Template | Purpose |
|---|---|---|
| `CopyRole` | `copy.template.yaml` | Lambda role that copies `dragen.zip` from QS source bucket to artifact bucket. |
| `CleanupRole` | `dragen.template.yaml` | Lambda role for stack deletion cleanup. Can delete ECR repository contents and empty artifact S3 bucket. |

---

## Supported Regions

Deployment is restricted to regions where the DRAGEN marketplace AMI is available:

- `us-east-1`
- `us-west-2`
- `eu-west-2`
- `ap-southeast-2`

This is enforced by a CloudFormation `Rules` assertion in the templates.

---

## Updating for a New DRAGEN Version

### Step-by-Step Process

1. **Update the AMI**
   - Find the new DRAGEN marketplace AMI IDs for each supported region.
   - Update the `AWSAMIRegionMap` mapping in `dragen.template.yaml`.

2. **Update the library path**
   - Check if the DRAGEN shared library version changed.
   - In `batch.template.yaml`, update the volume mount path (e.g., `/usr/lib64/libdragen.so.4.4.4`).

3. **Update the Docker image**
   - If the new version requires different OS packages or Python dependencies, update `app/source/dragen/Dockerfile`.

4. **Update the wrapper scripts**
   - If Illumina changed command-line arguments or behavior, update `dragen_qs.py` and related scripts.

5. **Repackage and redeploy**
   - Rebuild `app/packages/dragen/dragen.zip` with the updated source files.
   - Redeploy the CloudFormation stack. CodePipeline will automatically rebuild the Docker image.

### Pulling Upstream Changes

If this repo tracks the original Quick Start repository:

```bash
git remote add upstream <original-repo-url>
git fetch upstream
git merge upstream/main
# Resolve any conflicts with your customizations
```

---

## CI/CD & Testing

| Component | Purpose |
|---|---|
| `.taskcat.yml` | TaskCat test configuration for the project |
| `ci/taskcat.yml` | Additional TaskCat test definitions |
| `ci/batch-params.t1.json` | Test parameters for functional tests |
| `.project_automation/static_tests/` | cfn-lint validation of CloudFormation templates |
| `.project_automation/functional_tests/` | Full stack deployment tests via TaskCat |
| `.project_automation/publication/` | Publishes templates to regional S3 buckets |

---

## Things to Watch Out For

| Item | Risk | Recommendation |
|---|---|---|
| `CodePipelineServiceRole` and `CleanupRole` | Very broad S3 permissions (essentially `s3:*` on artifact bucket) | Tighten if security is a concern |
| Security group | Only allows HTTPS egress (port 443) | Add rules if DRAGEN needs other network access |
| Docker version pin | UserData downgrades to `docker-ce-25.0.5` | Could break if base AMI changes significantly |
| ECS agent | Pulled as `amazon/amazon-ecs-agent:latest` | Consider pinning a specific version |
| Library path | `/usr/lib64/libdragen.so.4.4.4` is hardcoded | Must be updated when upgrading DRAGEN versions |
