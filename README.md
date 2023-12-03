# RDS Instance Generation Upgrade

## Summary
This is a reference solution for keeping the DB instance generation (e.g. the **7** part in **db.m7g.large**) of [Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) up to latest.  User only needs to set the specified tag (default: `'upgrade-rds-instance-generation':'true'`) on the DB instance, and the upgrade to the latest instance generation will be performed automatically.

## What is the motivation for adopting a new instance type?
Newer instance types have improved performance and are more cost-effective (newer generations are often cheaper but there are some exceptions, for example db.m6g is cheaper than db.m7g).  Also, there is a pressing requirement of migrating to new instance type.  Because certain instance type will be deprecated someday.  For example, **db.m4**, **db.r4**, and **db.t2** retire in April 2024, for PostgreSQL, MySQL, and MariaDB.

## Is there any consideration to adopt new instance type?
Compared to [Amazon Elastic Compute Cloud (EC2)](https://aws.amazon.com/ec2/), RDS has higher validity in adopting new instance type.  RDS is deployed on EC2, and EC2 has instance families with different CPU architectures.  For instance, **m6i** is powered by Intel processor, and **m6g** is powered by Arm-based AWS Graviton2 processor.  When upgrading EC2 instance to a new instance type with a different architecture (e.g. **m5** to **m6g**), it is necessary to verify that the application runs on the target processor architecture.  On the other hand, in RDS where the application (database) has already been installed and is confirmed to function, there is no need to worry about the difference of architecture.

## What makes you hesitate to upgrade instance type?
Even if you understand the validity of adopting a new instance type, it takes a lot of effort to actually implement it.  Careful planning is required, especially in production environments.  DB instances in non-production environments are less difficult to upgrade, but the response tends to be slower due to their lower priority.  If you set the specified tag to the DB instances, this solution will automatically migrate them to the latest instance types.

## Installation
This reference solution can be deployed via [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template.  Instance type modification only happens for RDS DB instance which has specified tag.  Default is `'upgrade-rds-instance-generation':'true'`.  You can change these tag key/value as CloudFormation parameter.  Following parameters can be set through CloudFormation.

![CloudFormation_parameter_crop](https://github.com/sk8393/s3-intelligent-tiering-lifecycle-automation/assets/13175031/de69f6b1-1186-40a4-8cfe-b865f03cff32)

## Usage
This reference solution is a combination of [AWS Config](https://aws.amazon.com/config/) and [AWS Lambda](https://aws.amazon.com/pm/lambda/).  Config Custom Lambda Rule is executed every 24 hours.  List of RDS DB instances appears on AWS Management Console like below.

![Config_resource_in_scope_crop](https://github.com/sk8393/s3-intelligent-tiering-lifecycle-automation/assets/13175031/e3f65b31-d4dc-46f9-994e-f9ab060b6016)

We can check from above screenshot that: **1/** *Current instance type is* **db.m5.large**, **2/** **db.m7g.large** *is available as newer instance type*, and **3/** *Instance type will be modified by setting tag* `'upgrade-rds-instance-generation':'true'`.  Available instance type is checked by [AWS Price List API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/price-changes.html), so user does not need to maintain mapping (e.g. **db.m5.large** can be upgraded to **db.m7g.large** for RDS PostgreSQL in Frankfurt region).  Upgrade will be made in maintenance window by default (this can be changed to "Apply Immediately" through CloudFormation parameter).  Once upgrade is scheduled in the next maintenance window, it appears on AWS Management Console as below.

![RDS_pending_modification_crop](https://github.com/sk8393/s3-intelligent-tiering-lifecycle-automation/assets/13175031/60728db4-1480-40aa-88c2-1c62cc911e4b)

If the upgrade failed, the exception will be recorded in [Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) of Lambda function.  Exception during [ModifyDBInstance](https://docs.aws.amazon.com/AmazonRDS/latest/APIReference/API_ModifyDBInstance.html) API will be recorded as below.

![CloudWatch_Logs_crop](https://github.com/sk8393/s3-intelligent-tiering-lifecycle-automation/assets/13175031/ba3b701e-581d-463a-a1fa-33afa42622ca)

## How upgrade happens?
According to an offering in December 2023, this reference solution upgrades instance type as below.

|Engine|Current Instance Type|Target Instance Type|Note|
|:-:|:-:|:-:|:-:|
|Aurora PostgreSQL|db.t3.medium|db.t4g.medium|Aurora can be also upgraded.|
|MySQL|db.r5.large|db.r7g.large||
|MySQL|db.r5.24xlarge|db.r6i.24xlarge|24xlarge is not available for m7g.|
|SQL Server Web Edition|db.m5.2xlarge|db.m6i.2xlarge|There is no Graviton instance for SQL Server.|

## Support
This sample was tested that the expected results were obtained in an actual AWS account.  It is implemented so that changes are made when only specified tag is set to RDS DB instance, but we recommend that you try it once in a test environment when using it.
