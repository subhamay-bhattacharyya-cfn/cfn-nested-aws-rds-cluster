# CloudFormation RDS Cluster Template

<!-- Row 1: Status - Most Important -->
[![Release](https://img.shields.io/github/v/release/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster?label=Release)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/releases)&nbsp;[![Release Workflow](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/actions/workflows/release.yaml)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/badge/Languages-Python%20%7C%20YAML-blue)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazonaws&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-rds-cluster/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/31dcc0a54655348d6f57bfe2420f38de/raw/cfn-nested-aws-rds-cluster.json)](https://gist.github.com/subhamay-bhattacharyya/31dcc0a54655348d6f57bfe2420f38de)

This repository contains a nested CloudFormation template that creates the network-facing pieces of an Amazon RDS database: a DB subnet group, a database security group, a generated master-credentials secret, and (for Aurora and PostgreSQL) a DB cluster.

## Overview

The template is designed to be invoked from a parent stack. It can also be deployed on its own. It expects an existing VPC, existing subnets, and an existing application security group. It does not create the network.

## Template Files

- **`templates/template.yaml`** — the RDS template described in this README
- **`templates/parameters.json`** — parameter values for deploying the template

## What the Template Creates

| Logical resource | Type | Created when | Purpose |
| --- | --- | --- | --- |
| `DBSubnetGroup` | `AWS::RDS::DBSubnetGroup` | Always | Places the database in the subnets from `SubnetIds` |
| `DBSecurityGroup` | `AWS::EC2::SecurityGroup` | Always | Allows the database port from `AppSecurityGroupId` only |
| `DBSecret` | `AWS::SecretsManager::Secret` | Always | Holds the master username and a generated password as JSON |
| `DBCluster` | `AWS::RDS::DBCluster` | `DatabaseType` is `Aurora` or `PostgreSQL` | The database cluster |

`DBSecret` and `DBCluster` both depend on the secret. `DBCluster` reads the username and password from the secret at deploy time, so the password never appears in the template or in the parameters.

### Engine and port

| `DatabaseType` | Port | Cluster engine |
| --- | --- | --- |
| `MySQL` | 3306 | None (no cluster is created) |
| `Aurora` | 3306 | `aurora-mysql` |
| `PostgreSQL` | 5432 | `aurora-postgresql` |
| `Oracle` | 1521 | None (no cluster is created) |
| `SQLServer` | 1433 | None (no cluster is created) |

The security group allows TCP on the port for the selected engine, from `AppSecurityGroupId` only. No other inbound or outbound rules are added.

## Parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `ProjectName` | String | — | Prefix for resource names. 1–20 characters, lowercase letters, numbers, and hyphens (required) |
| `Environment` | String | `devl` | Deployment environment, used in resource names |
| `VpcId` | `AWS::EC2::VPC::Id` | — | Existing VPC that contains the subnets (required) |
| `SubnetIds` | CommaDelimitedList | — | Existing subnet IDs for the DB subnet group. Use at least two subnets in different Availability Zones (required) |
| `AppSecurityGroupId` | `AWS::EC2::SecurityGroup::Id` | — | Security group of the application that may connect to the database (required) |
| `DatabaseType` | String | `MySQL` | One of `MySQL`, `Aurora`, `PostgreSQL`, `Oracle`, `SQLServer` |
| `DatabaseVersion` | String | — | Engine version. Must be one of the allowed values for the selected `DatabaseType` (see [Engine versions](#engine-versions)) |
| `DatabaseUsername` | String | `dbadmin` | Master username. Starts with a letter; up to 16 letters, numbers, or underscores |
| `DatabaseName` | String | `appdb` | Name of the initial database. Starts with a letter; up to 63 letters, numbers, or underscores |
| `CiSuffix` | String | `""` | Optional suffix for CI deployments. When set, it is appended to resource names |

### Engine versions

`DatabaseVersion` has an `AllowedValues` list covering the recent versions of each engine, and a `Rules` block checks that the version belongs to the selected `DatabaseType`. The lists were taken from `aws rds describe-db-engine-versions` in `us-east-1`. RDS adds and retires versions over time, so the lists need updating when you need a version that isn't listed.

## Outputs

| Output | Value | Condition |
| --- | --- | --- |
| `DBSubnetGroupName` | Name of the DB subnet group | Always |
| `DBSubnetGroupArn` | ARN of the DB subnet group | Always |
| `DBSecurityGroupId` | ID of the database security group | Always |
| `DBSecurityGroupArn` | ARN of the database security group | Always |
| `DBSecretArn` | ARN of the credentials secret | Always |
| `DBClusterEndpoint` | Writer endpoint address | `Aurora` or `PostgreSQL` |
| `DBClusterArn` | ARN of the DB cluster | `Aurora` or `PostgreSQL` |

The outputs are not exported. Parent stacks read them with `!GetAtt <NestedStackName>.Outputs.<OutputName>`.

## Naming

Names follow the pattern `{ProjectName}-{Environment}-<resource>`:

| Resource | Name |
| --- | --- |
| Subnet group | `{ProjectName}-{Environment}-subnet-group` |
| Security group | `{ProjectName}-{Environment}-db-sg` |
| Cluster | `{ProjectName}-{Environment}-cluster` |
| Secret | `{ProjectName}-{Environment}-db-credentials` |

When `CiSuffix` is set, `-{CiSuffix}` is appended to each of these names. The subnet group and security group also get a `Name` tag with the same value.

Example with `ProjectName=myproject`, `Environment=devl`, `CiSuffix=ci42`: `myproject-devl-subnet-group-ci42`.

## Usage

### 1. Deploy on its own

Create `templates/parameters.json` with real values, then deploy:

```bash
aws cloudformation deploy \
  --template-file templates/template.yaml \
  --stack-name myproject-devl-rds \
  --region us-east-1 \
  --parameter-overrides \
    ProjectName=myproject \
    Environment=devl \
    VpcId=vpc-0123456789abcdef0 \
    SubnetIds=subnet-0aaaaaaaaaaaaaaa1,subnet-0bbbbbbbbbbbbbbb2 \
    AppSecurityGroupId=sg-0123456789abcdef0 \
    DatabaseType=Aurora \
    DatabaseVersion=8.0.mysql_aurora.3.13.0
```

`aws cloudformation deploy` does not read a parameters file, so pass the values with `--parameter-overrides` or use `create-stack --parameters` with the Key/Value array format.

### 2. Reference from a parent stack

Upload the template to S3, then reference it:

```bash
aws s3 cp templates/template.yaml s3://your-cfn-bucket/templates/rds-cluster.yaml
```

```yaml
RdsNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/rds-cluster.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      Environment: !Ref Environment
      VpcId: !Ref VpcId
      SubnetIds: !Join [",", !Ref SubnetIds]
      AppSecurityGroupId: !Ref AppSecurityGroupId
      DatabaseType: Aurora
      DatabaseVersion: 8.0.mysql_aurora.3.13.0
      CiSuffix: !Ref CiSuffix

Outputs:
  DBSecretArn:
    Value: !GetAtt RdsNestedStack.Outputs.DBSecretArn
```

### 3. CI deployments

Pass a suffix to keep resource names unique per pipeline run:

```bash
aws cloudformation deploy \
  --template-file templates/template.yaml \
  --stack-name myproject-devl-rds-ci \
  --parameter-overrides \
    ProjectName=myproject \
    VpcId=vpc-0123456789abcdef0 \
    SubnetIds=subnet-0aaaaaaaaaaaaaaa1,subnet-0bbbbbbbbbbbbbbb2 \
    AppSecurityGroupId=sg-0123456789abcdef0 \
    CiSuffix=$CI_PIPELINE_ID
```

## Limitations

- **No database instances.** `DBCluster` is created with no instances. Aurora and PostgreSQL clusters need at least one `AWS::RDS::DBInstance` before they accept connections. MySQL, Oracle, and SQL Server get no database resource at all; they need an `AWS::RDS::DBInstance`.
- **Cluster engines.** `DBCluster` supports only Aurora engines. `MySQL`, `Oracle`, and `SQLServer` do not create a cluster.
- **Application security group is required.** The database ingress rule uses `AppSecurityGroupId`. The template cannot run without an existing security group.
- **No customer-managed KMS key.** The secret and the cluster use the default AWS-managed encryption key.
- **No rotation.** The secret has no rotation schedule.
- **Deletion behavior.** The secret is deleted with the stack (`DeletionPolicy: Delete`). The cluster takes a final snapshot when the stack is deleted (`DeletionPolicy: Snapshot`).
- **Subnet count.** RDS requires subnets in at least two Availability Zones. The template does not check this, so a single-AZ list fails at deploy time.

## Best Practices Implemented

- ✅ Database is placed in the existing subnets you supply (use private subnets; the template does not check)
- ✅ Database port is open only to the application security group
- ✅ Master password is generated by Secrets Manager and never stored in the template
- ✅ Storage encryption enabled on the cluster
- ✅ Backup retention of 7 days on the cluster
- ✅ Cluster is protected by a snapshot on stack deletion
- ✅ Engine versions are validated against the selected engine
- ✅ Optional CI suffix for unique deployments

## License

MIT
