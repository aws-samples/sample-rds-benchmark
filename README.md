# AWS Graviton5 Benchmark Setup for Amazon RDS

This AWS CloudFormation template deploys a complete, automated benchmark environment to compare AWS Graviton5 (M9g), Graviton4 (M8g), Graviton3 (M7g), and Graviton2 (M6g) processors on Amazon RDS. The benchmark runs automatically with no manual commands required.

For the full analysis and results, see the blog post: [Boosting Amazon RDS with AWS Graviton5: Benchmarks](https://aws.amazon.com/blogs/database/) <!-- TODO: Update with final blog URL after publication -->

## Architecture

![Benchmark Architecture](assets/architecture-diagram.jpg)

The template deploys an Amazon EC2 instance running Sysbench connected to an Amazon RDS instance in the same Availability Zone within a private VPC. All connectivity uses VPC endpoints — no public IPs are required.

## Quick Start

**1. Deploy the stack**

```bash
aws cloudformation create-stack \
  --stack-name rds-bench-m9g \
  --template-body file://rds-graviton5-benchmark.yaml \
  --parameters ParameterKey=InstanceClass,ParameterValue=db.m9g.xlarge \
               ParameterKey=DatabaseEngine,ParameterValue=postgres \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**2. Wait for deployment (~15 minutes)**

```bash
aws cloudformation wait stack-create-complete --stack-name rds-bench-m9g --region us-east-1
```

**3. Connect to the EC2 instance**

Get the Session Manager URL from the stack outputs:

```bash
aws cloudformation describe-stacks --stack-name rds-bench-m9g --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='SessionManagerURL'].OutputValue" --output text
```

Open the URL in your browser, or connect via CLI:

```bash
aws ssm start-session --target <instance-id> --region us-east-1
```

**4. Run the benchmark**

```bash
/opt/run-benchmark.sh
```

The script runs the full benchmark automatically (approximately 35 minutes). Results are saved to `/opt/results/`.

**5. Cleanup**

```bash
aws cloudformation delete-stack --stack-name rds-bench-m9g --region us-east-1
```

## What the Stack Creates

| Resource | Description |
|----------|-------------|
| VPC | Private VPC with a subnet in a single Availability Zone |
| Amazon RDS DB instance | Running the selected engine and instance class (Single-AZ) |
| Amazon EC2 instance | m7g.4xlarge with Sysbench pre-installed and benchmark script |
| VPC endpoints | AWS Systems Manager Session Manager (private connectivity) |
| AWS Secrets Manager secret | Database credentials (auto-generated, no hardcoded passwords) |
| IAM roles | Least-privilege roles for Session Manager access |
| Parameter group | Enforces TLS connections (`rds.force_ssl` for PostgreSQL, `require_secure_transport` for MySQL/MariaDB) |
| Amazon EBS io2 volume | 200 GiB with 10,000 provisioned IOPS attached to the RDS instance |

## Parameters

| Parameter | Description | Default | Allowed Values |
|-----------|-------------|---------|----------------|
| `DatabaseEngine` | Database engine to benchmark | `postgres` | `postgres`, `mysql`, `mariadb` |
| `InstanceClass` | RDS instance class | `db.m9g.xlarge` | `db.m6g.xlarge`, `db.m6g.2xlarge`, `db.m6g.4xlarge`, `db.m7g.xlarge`, `db.m7g.2xlarge`, `db.m7g.4xlarge`, `db.m8g.xlarge`, `db.m8g.2xlarge`, `db.m8g.4xlarge`, `db.m9g.xlarge`, `db.m9g.2xlarge`, `db.m9g.4xlarge` |

## What the Benchmark Script Does

The `/opt/run-benchmark.sh` script automates the full Sysbench workflow:

1. **Prepare** — Creates 2 tables (`sbtest1`, `sbtest2`) with 2,000,000 rows each, generating the dataset for the OLTP workload
2. **Run** — Executes the `oltp_read_write` workload with 100 concurrent threads for 1,800 seconds (30 minutes), reporting metrics every 10 seconds
3. **Cleanup** — Drops the test tables after the run completes
4. **Save results** — Writes the full Sysbench output to `/opt/results/` with a timestamped filename including the engine and instance class

The script retrieves database credentials from Secrets Manager automatically and connects over TLS. No manual configuration is needed.

## Benchmark Configuration

| Parameter | Value |
|-----------|-------|
| Sysbench workload | OLTP read/write (`oltp_read_write.lua`) |
| Tables | 2 |
| Rows per table | 2,000,000 |
| Threads | 100 |
| Duration | 1,800 seconds (30 minutes) |
| Report interval | 10 seconds |
| TLS | Enforced |

## Supported Configurations

### Instance Classes

| Family | Processor | Supported Sizes |
|--------|-----------|-----------------|
| M6g | Graviton2 | xlarge, 2xlarge, 4xlarge |
| M7g | Graviton3 | xlarge, 2xlarge, 4xlarge |
| M8g | Graviton4 | xlarge, 2xlarge, 4xlarge |
| M9g | Graviton5 | xlarge, 2xlarge, 4xlarge |

### Database Engines

| Engine | Version |
|--------|---------|
| PostgreSQL | 17.10 |
| MySQL | 8.4 |
| MariaDB | 11.4 |

### Regions

As of this writing, M9g instances are available in:
- US East (N. Virginia) — `us-east-1`
- US East (Ohio) — `us-east-2`
- US West (Oregon) — `us-west-2`
- Europe (Frankfurt) — `eu-central-1`

M6g, M7g, and M8g instances are available in additional regions. See the [Amazon RDS documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.DBInstanceClass.Support.html) for full availability.

## Comparing All Four Generations

To run a side-by-side comparison, deploy four stacks (one per generation):

```bash
for INSTANCE in db.m6g.xlarge db.m7g.xlarge db.m8g.xlarge db.m9g.xlarge; do
  STACK_NAME="rds-bench-$(echo $INSTANCE | cut -d. -f2)"
  aws cloudformation create-stack \
    --stack-name $STACK_NAME \
    --template-body file://rds-graviton5-benchmark.yaml \
    --parameters ParameterKey=InstanceClass,ParameterValue=$INSTANCE \
                 ParameterKey=DatabaseEngine,ParameterValue=postgres \
    --capabilities CAPABILITY_NAMED_IAM \
    --region us-east-1
done
```

Then connect to each EC2 instance and run `/opt/run-benchmark.sh`.

## Costs

⚠️ **This template provisions resources that incur charges.** Estimated cost per stack:

| Resource | Approximate Cost |
|----------|-----------------|
| RDS db.m9g.xlarge (Single-AZ) | ~$0.37/hour |
| EC2 m7g.4xlarge | ~$0.58/hour |
| EBS io2 (200 GiB, 10K IOPS) | ~$0.17/hour |
| VPC endpoints (3x) | ~$0.03/hour |
| **Total per stack** | **~$1.15/hour** |

A full four-generation comparison costs approximately **$4.60/hour**. Each benchmark takes ~35 minutes. Delete all stacks when testing is complete to stop charges.

## Cleanup

Delete a single stack:

```bash
aws cloudformation delete-stack --stack-name rds-bench-m9g --region us-east-1
```

Delete all benchmark stacks:

```bash
for STACK in rds-bench-m6g rds-bench-m7g rds-bench-m8g rds-bench-m9g; do
  aws cloudformation delete-stack --stack-name $STACK --region us-east-1
done
```

Or navigate to AWS CloudFormation in the [AWS Management Console](https://console.aws.amazon.com/cloudformation/), select each stack, and choose **Delete**.

## Security

This template follows AWS security best practices:

- **No public IPs** — All connectivity through VPC endpoints (Systems Manager Session Manager)
- **TLS enforced** — Database connections require encryption via parameter group settings
- **No hardcoded credentials** — Database passwords auto-generated and stored in AWS Secrets Manager
- **Least-privilege IAM** — EC2 instance role has only the permissions needed for Session Manager and Secrets Manager access
- **Private subnet** — RDS instance is not accessible from the internet

## Prerequisites

- An active AWS account
- Sufficient IAM permissions to create CloudFormation stacks, VPCs, EC2 instances, RDS instances, and IAM roles
- AWS CLI configured (for CLI deployment) or access to the AWS Management Console

## Related

- Blog post: [Boosting Amazon RDS with AWS Graviton5: Benchmarks](https://aws.amazon.com/blogs/database/) <!-- TODO: Update URL -->
- [Amazon RDS pricing](https://aws.amazon.com/rds/pricing/)
- [Amazon RDS instance types](https://aws.amazon.com/rds/instance-types/)
- [AWS Graviton](https://aws.amazon.com/ec2/graviton/)

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
