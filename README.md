# quickstart-illumina-dragen
## DRAGEN on the AWS Cloud


This Quick Start deploys Dynamic Read Analysis for GENomics Complete Suite (DRAGEN CS), a data analysis platform by Illumina, on the AWS Cloud in about 15 minutes.

DRAGEN CS enables ultra-rapid analysis of next-generation sequencing (NGS) data, significantly reduces the time required to analyze genomic data, and improves accuracy. It includes bioinformatics pipelines that provide highly optimized algorithms for mapping, aligning, sorting, duplicate marking, and haplotype variant calling. These pipelines include DRAGEN Germline V2, DRAGEN Somatic V2 (Tumor and Tumor/Normal), DRAGEN Virtual Long Read Detection (VLRD), DRAGEN RNA Gene Fusion, DRAGEN Joint Genotyping, and GATK Best Practices.

The Quick Start builds an AWS environment that spans two Availability Zones for high availability, and provisions two AWS Batch compute environments for Spot Instances and On-Demand Instances. These environments include DRAGEN F1 instances that are connected to field-programmable gate arrays (FPGAs) for hardware acceleration.

The Quick Start offers two deployment options:

- Deploying DRAGEN into a new virtual private cloud (VPC) on AWS
- Deploying DRAGEN into an existing VPC on AWS

You can also use the AWS CloudFormation templates as a starting point for your own implementation.

![Quick Start architecture for DRAGEN on AWS](https://d0.awsstatic.com/partner-network/QuickStart/datasheets/quickstart-architecture-for-dragen-on-aws.png)

For architectural details, best practices, step-by-step instructions, and customization options, see the 
[deployment guide](https://fwd.aws/YqKNQ).

To post feedback, submit feature ideas, or report bugs, use the **Issues** section of this GitHub repo.
If you'd like to submit code for this Quick Start, please review the [AWS Quick Start Contributor's Kit](https://aws-quickstart.github.io/). 

## Known Issues & Important Notes

### Security Group Egress Rules (Critical)

The Batch compute environment security group in `templates/batch.template.yaml` **must** include egress rules for HTTP, DNS, and NTP in addition to HTTPS. Without these, containers will experience intermittent or consistent exit code 137 (SIGKILL) failures due to:

- **TCP 80**: Required for the ECS task credential endpoint (`169.254.170.2:80`). Without this, containers lose the ability to refresh IAM credentials mid-run, causing S3 streaming failures during long-running jobs.
- **UDP 53**: Required for DNS resolution. Without this, containers may fail to resolve S3 endpoints or the DRAGEN license server.
- **UDP 123**: Required for NTP time synchronization.

The original `igm-dragen-batch` stack had these rules added manually at deploy time but they were never committed to this repository. The upstream Illumina quickstart template only includes TCP 443, which is insufficient for production DRAGEN workloads.

**Symptoms of missing rules**: Exit code 137 errors that appear to be OOM kills but are actually caused by network timeouts. Jobs may succeed intermittently (when credentials are still cached) and fail on longer-running samples.

### DRAGEN AMI and f2.12xlarge Compatibility

The DRAGEN 4.5.4 private AMI (`ami-0de1a56afe1c3b0b8`) does **not** properly support `f2.12xlarge` instances (dual FPGA boards). Board1 fails to initialize with `HWAL error: Timeout during last indirect operation -- indirect control register 0xf000 = 0xdeadbeef`. Only use `f2.6xlarge` with this AMI until Illumina provides a validated multi-board image.
