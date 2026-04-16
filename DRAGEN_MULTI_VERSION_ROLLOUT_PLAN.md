# DRAGEN Multi-Version Rollout Plan

## Objective

Deploy DRAGEN 4.5 alongside the existing DRAGEN 4.4 infrastructure so that the `igm-dragen` pipeline can route jobs to either version at runtime. This enables:

- Running DRAGEN 4.4 and 4.5 in parallel (e.g., for validation, comparison, or gradual migration)
- Deploying additional test/dev versions of the DRAGEN infrastructure independently
- Choosing the DRAGEN version per-case at Step Functions execution time

## Current Architecture

### dragen-aws-batch-quickstart (DRAGEN 4.4)

The existing quickstart CloudFormation stack (`igm-dragen-batch`) deploys:

| Resource | Hardcoded Name | Template |
|---|---|---|
| On-Demand Compute Environment | `dragen-ondemand` | `batch.template.yaml` |
| Spot Compute Environment | `dragen-spot` | `batch.template.yaml` |
| Job Queue | `dragen-queue` | `batch.template.yaml` |
| Job Definition | `dragen` | `batch.template.yaml` |
| ECR Repository | (auto-generated) | `docker-bucket-repository.template.yaml` |
| Artifact S3 Bucket | (auto-generated) | `docker-bucket-repository.template.yaml` |

The AMI IDs are hardcoded in `dragen.template.yaml` under the `AWSAMIRegionMap` mapping:

| Region | Current AMI (4.4) |
|---|---|
| us-east-1 | `ami-0b200106f98d8acd2` |
| us-west-2 | `ami-07a7d63c4157030dc` |
| eu-west-2 | `ami-0ece6aa6f1f8d79f2` |
| ap-southeast-2 | `ami-0ffc6f405569e4661` |

The DRAGEN shared library is mounted at `/usr/lib64/libdragen.so.4.4.4` in the Batch job definition (volume mount in `batch.template.yaml`).

### igm-dragen Pipeline

The Step Functions workflow submits Batch jobs using values from `create_sample_command.py`:

- `jobQueue`: comes from the `JOB_QUEUE_NAME` environment variable (set to `dragen-queue` via `samconfig.yaml`)
- `jobDefinition`: hardcoded as `'dragen'` in `create_sample_command.py`

The workflow (`workflow.asl.json`) reads these from the `batchTemplate` object:
- `$batchTemplate.jobQueue` — used in "Run DRAGEN Pipeline Job" and all Joint Analysis steps
- `$batchTemplate.jobDefinition` — used in the same steps

---

## Rollout Plan

### Phase 1: Parameterize the Quickstart Resource Names

The core problem is that Batch resource names are hardcoded strings in `batch.template.yaml`. Deploying a second stack would cause CloudFormation name collisions. We need to make them version-aware.

#### 1.1 Add `DragenVersion` parameter to `batch.template.yaml`

Add a new parameter at the top of `batch.template.yaml`:

```yaml
Parameters:
  # ... existing parameters ...
  DragenVersion:
    Type: String
    Description: >-
      Version identifier used to make resource names unique (e.g., "44", "45", "45-dev").
      This allows multiple DRAGEN versions to coexist in the same AWS account/region.
    Default: "44"
    AllowedPattern: "[a-zA-Z0-9-]+"
```

#### 1.2 Update hardcoded resource names in `batch.template.yaml`

Replace the four hardcoded names:

| Resource | Current Name | New Name |
|---|---|---|
| `DragenComputeEnvironmentSpot` | `dragen-spot` | `!Sub dragen-${DragenVersion}-spot` |
| `DragenComputeEnvironmentOnDemand` | `dragen-ondemand` | `!Sub dragen-${DragenVersion}-ondemand` |
| `DragenJobQueue` | `dragen-queue` | `!Sub dragen-${DragenVersion}-queue` |
| `DragenJobDefinition` | `dragen` | `!Sub dragen-${DragenVersion}` |

Example diff for the job queue:

```yaml
# Before
DragenJobQueue:
  Type: AWS::Batch::JobQueue
  Properties:
    JobQueueName: dragen-queue

# After
DragenJobQueue:
  Type: AWS::Batch::JobQueue
  Properties:
    JobQueueName: !Sub dragen-${DragenVersion}-queue
```

#### 1.3 Update the library mount path in `batch.template.yaml`

The DRAGEN shared library version is hardcoded in the job definition's volume mount. Add a parameter for it:

```yaml
Parameters:
  # ... existing parameters ...
  DragenLibVersion:
    Type: String
    Description: >-
      DRAGEN shared library version string (e.g., "4.4.4", "4.5.3").
      Must match the library filename on the AMI at /usr/lib64/libdragen.so.<version>.
    Default: "4.4.4"
```

Then update the volume mount in `DragenJobDefinition`:

```yaml
# Before
- ContainerPath: "/usr/lib64/libdragen.so.4.4.4"
  ReadOnly: False
  SourceVolume: docker_usr_lib64
# ...
- Name: docker_usr_lib64
  Host:
    SourcePath: "/usr/lib64/libdragen.so.4.4.4"

# After
- ContainerPath: !Sub "/usr/lib64/libdragen.so.${DragenLibVersion}"
  ReadOnly: False
  SourceVolume: docker_usr_lib64
# ...
- Name: docker_usr_lib64
  Host:
    SourcePath: !Sub "/usr/lib64/libdragen.so.${DragenLibVersion}"
```

#### 1.4 Pass the new parameters through `dragen.template.yaml`

Add `DragenVersion` and `DragenLibVersion` as parameters in the parent template `dragen.template.yaml`, and pass them to the `Batch` nested stack:

```yaml
# In dragen.template.yaml Parameters section:
DragenVersion:
  Type: String
  Description: Version identifier for resource naming (e.g., "44", "45")
  Default: "44"
  AllowedPattern: "[a-zA-Z0-9-]+"
DragenLibVersion:
  Type: String
  Description: DRAGEN shared library version (e.g., "4.4.4", "4.5.3")
  Default: "4.4.4"

# In the Batch nested stack resource:
Batch:
  Type: AWS::CloudFormation::Stack
  Properties:
    # ... existing properties ...
    Parameters:
      # ... existing parameters ...
      DragenVersion: !Ref DragenVersion
      DragenLibVersion: !Ref DragenLibVersion
```

### Phase 2: Update the Existing 4.4 Stack

Before deploying 4.5, update the existing stack so its resources get the version-suffixed names. This is a one-time migration.

> **Warning:** Renaming Batch compute environments, job queues, and job definitions requires CloudFormation to replace them (delete old + create new). Make sure no jobs are running when you do this update.

#### 2.1 Upload updated templates to S3

The quickstart templates are served from S3 (the `QSS3BucketName` / `QSS3KeyPrefix` location). Upload the modified templates to the same S3 location.

```bash
# Package the updated templates
aws s3 sync templates/ s3://<QSS3BucketName>/<QSS3KeyPrefix>templates/ \
  --region <QSS3BucketRegion>
```

#### 2.2 Update the existing CloudFormation stack

```bash
aws cloudformation update-stack \
  --stack-name igm-dragen-batch \
  --template-url https://<QSS3BucketName>.s3.<region>.amazonaws.com/<QSS3KeyPrefix>templates/dragen.template.yaml \
  --parameters \
    ParameterKey=VPCID,UsePreviousValue=true \
    ParameterKey=PrivateSubnet1ID,UsePreviousValue=true \
    ParameterKey=PrivateSubnet2ID,UsePreviousValue=true \
    ParameterKey=KeyPairName,UsePreviousValue=true \
    ParameterKey=GenomicsS3Bucket,UsePreviousValue=true \
    ParameterKey=QSS3BucketName,UsePreviousValue=true \
    ParameterKey=QSS3KeyPrefix,UsePreviousValue=true \
    ParameterKey=MaxvCpus,UsePreviousValue=true \
    ParameterKey=InstanceType,UsePreviousValue=true \
    ParameterKey=DragenVersion,ParameterValue=44 \
    ParameterKey=DragenLibVersion,ParameterValue=4.4.4 \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

After this update, the resources will be named:
- `dragen-44-queue`
- `dragen-44-ondemand`
- `dragen-44-spot`
- `dragen-44` (job definition)

#### 2.3 Update igm-dragen samconfig.yaml to use the new queue name

In `igm-dragen/samconfig.yaml`, update `JobQueueName` for all environments:

```yaml
# Before
- JobQueueName=dragen-queue

# After
- JobQueueName=dragen-44-queue
```

### Phase 3: Find the DRAGEN 4.5 AMI IDs

Before deploying the 4.5 stack, you need the new AMI IDs.

```bash
# Find DRAGEN 4.5 marketplace AMIs in your region
aws ec2 describe-images \
  --filters "Name=product-code,Values=9uotaksivr7km6tn0wa0sy2fw" \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table \
  --region us-east-1
```

Look for AMIs with "4.5" or "4.5.x" in the name. Note the AMI ID for your region (likely `us-east-1` based on your samconfig).

You also need to determine the exact `libdragen.so` version on the new AMI. You can do this by launching a temporary instance from the AMI and checking:

```bash
ls /usr/lib64/libdragen.so.*
```

The filename will be something like `libdragen.so.4.5.3` — note this version string for the `DragenLibVersion` parameter.

### Phase 4: Deploy the DRAGEN 4.5 Stack

#### 4.1 Create a copy of `dragen.template.yaml` with 4.5 AMI IDs

You have two options:

**Option A (recommended):** Make the AMI an explicit parameter instead of a mapping, so you don't need to maintain separate template copies. Add to `dragen.template.yaml`:

```yaml
Parameters:
  # ... existing parameters ...
  DragenAmiOverride:
    Type: String
    Description: >-
      Optional AMI ID override. If provided, this is used instead of the AWSAMIRegionMap lookup.
      Use this when deploying a newer DRAGEN version with a different AMI.
    Default: ""

Conditions:
  # ... existing conditions ...
  HasAmiOverride: !Not [!Equals [!Ref DragenAmiOverride, ""]]
```

Then update the `Batch` nested stack's `ImageId` parameter:

```yaml
Batch:
  Properties:
    Parameters:
      # Before:
      # ImageId: !FindInMap [ AWSAMIRegionMap, !Ref "AWS::Region", DRAGEN ]
      # After:
      ImageId: !If
        - HasAmiOverride
        - !Ref DragenAmiOverride
        - !FindInMap [ AWSAMIRegionMap, !Ref "AWS::Region", DRAGEN ]
```

**Option B:** Update the `AWSAMIRegionMap` mapping directly with the 4.5 AMI IDs. This is simpler but means you need separate template files per version.

#### 4.2 Deploy the new stack

```bash
aws cloudformation create-stack \
  --stack-name igm-dragen-batch-45 \
  --template-url https://<QSS3BucketName>.s3.<region>.amazonaws.com/<QSS3KeyPrefix>templates/dragen.template.yaml \
  --parameters \
    ParameterKey=VPCID,ParameterValue=<your-vpc-id> \
    ParameterKey=PrivateSubnet1ID,ParameterValue=<your-subnet-1> \
    ParameterKey=PrivateSubnet2ID,ParameterValue=<your-subnet-2> \
    ParameterKey=KeyPairName,ParameterValue=<your-keypair> \
    ParameterKey=GenomicsS3Bucket,ParameterValue=<your-genomics-bucket> \
    ParameterKey=QSS3BucketName,ParameterValue=<your-qs-bucket> \
    ParameterKey=QSS3KeyPrefix,ParameterValue=<your-qs-prefix> \
    ParameterKey=MaxvCpus,ParameterValue=<your-max-vcpus> \
    ParameterKey=InstanceType,ParameterValue=f2.6xlarge \
    ParameterKey=DragenVersion,ParameterValue=45 \
    ParameterKey=DragenLibVersion,ParameterValue=<4.5.x from Phase 3> \
    ParameterKey=DragenAmiOverride,ParameterValue=<ami-id-from-phase-3> \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

This creates a fully independent stack with:
- `dragen-45-queue`
- `dragen-45-ondemand`
- `dragen-45-spot`
- `dragen-45` (job definition)
- Its own ECR repository and Docker image
- Its own CodePipeline that builds the container automatically

#### 4.3 Wait for the stack to complete

The stack takes ~15-20 minutes because CodePipeline needs to build the Docker image before the Batch resources are created (controlled by the `WaitCondition`).

```bash
aws cloudformation wait stack-create-complete --stack-name igm-dragen-batch-45
```

Verify the resources:

```bash
# Check the job queue exists
aws batch describe-job-queues --job-queues dragen-45-queue

# Check the job definition exists
aws batch describe-job-definitions --job-definition-name dragen-45
```

### Phase 5: Update igm-dragen for Version Routing

The goal is to let the caller choose which DRAGEN version to use when starting a Step Functions execution.

#### 5.1 Add optional `dragenVersion` fields to the input schema

In `igm-dragen/lambda_functions/validate_input/schema.json`, add:

```json
{
  "properties": {
    "jobQueueName": {
      "title": "jobQueueName",
      "type": "string",
      "description": "Override the Batch job queue name. Use this to target a specific DRAGEN version (e.g., 'dragen-45-queue')."
    },
    "jobDefinitionName": {
      "title": "jobDefinitionName",
      "type": "string",
      "description": "Override the Batch job definition name. Use this to target a specific DRAGEN version (e.g., 'dragen-45')."
    }
  }
}
```

Since the schema has `"additionalProperties": true`, these fields will pass through even before you formally add them, but adding them documents the contract.

#### 5.2 Update `create_sample_command.py` to accept overrides from input

In `igm-dragen/lambda_functions/create_sample_command/create_sample_command.py`, update `create_batch_job_template()`:

```python
def create_batch_job_template(event: dict, dragen_license_key: str) -> dict:
    # ... existing command building code ...

    return {
        "jobNamePrefix": event['caseName'],
        # Allow input to override the job queue and job definition
        "jobQueue": event.get('jobQueueName', os.environ.get('JOB_QUEUE_NAME', 'dragen-44-queue')),
        "jobDefinition": event.get('jobDefinitionName', os.environ.get('JOB_DEFINITION_NAME', 'dragen-44')),
        # ... rest of the template ...
    }
```

This gives you three levels of configuration:
1. **Input event** (highest priority) — per-execution override
2. **Environment variable** — per-deployment default (set in `template.yaml`)
3. **Hardcoded fallback** — safety net

#### 5.3 Update `template.yaml` environment variables

Add `JOB_DEFINITION_NAME` to the `CreateSampleCommandFunction` if it's not already there (it is already present via the `!If` condition). Make sure the default values match your 4.4 deployment:

```yaml
CreateSampleCommandFunction:
  Properties:
    Environment:
      Variables:
        JOB_QUEUE_NAME: !Ref JobQueueName
        JOB_DEFINITION_NAME: !If
          - HasAlias
          - !Sub dragen-${JobQueueVersion}
          - !Sub dragen-${JobQueueVersion}
```

Or more simply, just keep the existing env var pattern and ensure `samconfig.yaml` passes the right default queue name.

#### 5.4 Update `samconfig.yaml` with the default version

```yaml
# For all environments, set the default to 4.4:
parameter_overrides:
  - JobQueueName=dragen-44-queue
```

#### 5.5 Grant the Step Functions role access to both job queues

In `igm-dragen/template.yaml`, the `StateMachinePolicy` currently scopes `batch:SubmitJob` to a single job queue. Update it to allow both:

```yaml
StateMachinePolicy:
  Properties:
    PolicyDocument:
      Statement:
        - Effect: Allow
          Action:
            - batch:SubmitJob
            - batch:TagResource
            - batch:UntagResource
            - batch:ListTagsForResource
          Resource:
            - !Sub arn:aws:batch:${AWS::Region}:${AWS::AccountId}:job-definition/*
            - !Sub arn:aws:batch:${AWS::Region}:${AWS::AccountId}:job-queue/dragen-*
            # This wildcard covers dragen-44-queue, dragen-45-queue, etc.
```

### Phase 6: Test and Validate

#### 6.1 Test with DRAGEN 4.4 (default behavior)

Start a Step Functions execution without specifying version overrides. It should use `dragen-44-queue` and `dragen-44` job definition by default.

```json
{
  "caseName": "test-44",
  "reference": "s3://nch-igm-churchill-resources/dragen_resources/hg38-alt_masked.cnv.graph.hla.methyl_cg.rna-11-r5.0-1.tar.gz",
  "samples": [ ... ],
  "outputDirectory": "s3://your-bucket/test-44-output/",
  "tags": { "Pipeline": "dragen-wgs" }
}
```

#### 6.2 Test with DRAGEN 4.5 (explicit override)

Start a Step Functions execution with version overrides:

```json
{
  "caseName": "test-45",
  "reference": "s3://nch-igm-churchill-resources/dragen_resources/<4.5-compatible-reference>.tar.gz",
  "jobQueueName": "dragen-45-queue",
  "jobDefinitionName": "dragen-45",
  "samples": [ ... ],
  "outputDirectory": "s3://your-bucket/test-45-output/",
  "tags": { "Pipeline": "dragen-wgs" }
}
```

> **Important:** DRAGEN 4.5 may require regenerated reference hash tables. Check Illumina's release notes. If so, you'll need to generate new hash tables and use a different `reference` S3 path for 4.5 jobs.

#### 6.3 Compare outputs

Run the same sample through both versions and compare the outputs to validate 4.5 produces expected results before migrating production workloads.

### Phase 7: Deploying Additional Versions (Test/Dev)

The same pattern works for any number of versions. For example, to deploy a test environment:

```bash
aws cloudformation create-stack \
  --stack-name igm-dragen-batch-45-dev \
  --template-url https://<bucket>.s3.<region>.amazonaws.com/<prefix>templates/dragen.template.yaml \
  --parameters \
    ParameterKey=DragenVersion,ParameterValue=45-dev \
    ParameterKey=DragenLibVersion,ParameterValue=<version> \
    ParameterKey=DragenAmiOverride,ParameterValue=<ami-id> \
    # ... other params same as production ...
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM
```

This creates `dragen-45-dev-queue`, `dragen-45-dev` job definition, etc.

---

## Summary of All Changes

### dragen-aws-batch-quickstart changes

| File | Change |
|---|---|
| `templates/batch.template.yaml` | Add `DragenVersion` and `DragenLibVersion` parameters. Update 4 hardcoded resource names to use `!Sub` with version suffix. Update `libdragen.so` mount path to use parameter. |
| `templates/dragen.template.yaml` | Add `DragenVersion`, `DragenLibVersion`, and `DragenAmiOverride` parameters. Add `HasAmiOverride` condition. Pass new params to Batch nested stack. Use conditional AMI lookup. |

### igm-dragen changes

| File | Change |
|---|---|
| `lambda_functions/validate_input/schema.json` | Add optional `jobQueueName` and `jobDefinitionName` properties. |
| `lambda_functions/create_sample_command/create_sample_command.py` | Read `jobQueueName` and `jobDefinitionName` from input event with fallback to env vars. |
| `template.yaml` | Update `StateMachinePolicy` to allow `batch:SubmitJob` on `dragen-*` queues (wildcard). |
| `samconfig.yaml` | Update `JobQueueName` from `dragen-queue` to `dragen-44-queue` for all environments. |

### Deployment order

1. Modify quickstart templates (Phase 1)
2. Update existing 4.4 stack to use new names (Phase 2) — **requires no jobs running**
3. Find 4.5 AMI IDs and lib version (Phase 3)
4. Deploy 4.5 stack (Phase 4)
5. Update igm-dragen code and redeploy (Phase 5)
6. Test both versions (Phase 6)

### Risk considerations

| Risk | Mitigation |
|---|---|
| Renaming 4.4 resources causes downtime | Schedule during maintenance window with no running jobs |
| DRAGEN 4.5 requires new reference hash tables | Check Illumina release notes before deploying; generate new hash tables if needed |
| DRAGEN 4.5 CLI argument changes | Review Illumina changelog; test with a single sample before production |
| igm-dragen Step Functions role lacks permissions for new queue | Phase 5.5 updates the IAM policy with a wildcard pattern |
| Cost of running two Batch environments | Both use `MinvCpus: 0` by default, so idle cost is zero — you only pay when jobs run |
