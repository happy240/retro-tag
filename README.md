# Retro Tag

[![Software License](https://img.shields.io/github/license/gorillastack/retro-tag.svg?style=for-the-badge)](/LICENSE)
![GitHub last commit](https://img.shields.io/github/last-commit/gorillastack/retro-tag.svg?style=for-the-badge)
[![Powered By: GorillaStack](https://img.shields.io/badge/powered%20by-GorillaStack-green.svg?style=for-the-badge)](https://www.gorillastack.com)

Retro Tag helps you retrospectively tag resources with the ARN of the user that created them and the creation date/time using the [Auto Tag](https://github.com/GorillaStack/auto-tag) engine, and supports tagging across multiple regions and across AWS accounts.

This is designed to solve the problem of [Auto Tagging](https://github.com/GorillaStack/auto-tag) existing resources in your environments. 

## About

Retro Tag uses the log data in your AWS CloudTrail S3 bucket logs to gather information about the "who" and "when" for each of your AWS existing resources. Using this information, engineers can determine which resources are required, which are not, and can cleanup the resources, or improve their tagging.

## Installation

The installation consists of a single `Main` CloudFormation Stack with the AutoTag Lambda function in the same account as the CloudTrail S3 Bucket, and a `Role` CloudFormation Stack deployed to each account where tagging will be applied.

### Query CloudTrail logs using AWS Athena

Use AWS Athena to scan your history of CloudTrail logs in S3 and retrospectively tag existing AWS resources.

WARNING: You are charged for AWS Athena based on the amount the data that is scanned.

#### Table Query

Login to the AWS account and region where your CloudTrail logs S3 bucket is located and bring up the Athena service. You'll probably need to create a unique table for each AWS account in the S3 bucket.

Update the table name, S3 bucket, S3 path including the AWS account ID to query.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS my_table_name (
eventversion STRING,
userIdentity STRUCT<
               type:STRING,
               principalid:STRING,
               arn:STRING,
               accountid:STRING,
               invokedby:STRING,
               accesskeyid:STRING,
               userName:STRING,
sessioncontext:STRUCT<
attributes:STRUCT<
               mfaauthenticated:STRING,
               creationdate:STRING>,
sessionIssuer:STRUCT<  
               type:STRING,
               principalId:STRING,
               arn:STRING, 
               accountId:STRING,
               userName:STRING>>>,
eventTime STRING,
eventSource STRING,
eventName STRING,
awsRegion STRING,
sourceIpAddress STRING,
userAgent STRING,
errorCode STRING,
errorMessage STRING,
requestParameters STRING,
responseElements STRING,
additionalEventData STRING,
requestId STRING,
eventId STRING,
resources ARRAY<STRUCT<
               ARN:STRING,
               accountId:STRING,
               type:STRING>>,
eventType STRING,
apiVersion STRING,
readOnly STRING,
recipientAccountId STRING,
serviceEventDetails STRING,
sharedEventID STRING,
vpcEndpointId STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-bucket/dev/AWSLogs/11111111111/'
```

#### Data Query

Update the table name, run the Athena query, and save the output to a CSV file.

NOTE: You can request a longer Athena query timeout limit from AWS if the default 30 minutes is not enough. 

```sql
SELECT eventTime, eventSource, eventName, awsRegion, userIdentity.accountId as "userIdentity.accountId", recipientAccountId, "$path" as key, requestParameters, responseElements
FROM my_table_name
WHERE
eventName in (
    'AllocateAddress',
    'CloneStack',
    'CopyImage',
    'CopySnapshot',
    'CreateAutoScalingGroup',
    'CreateBucket',
    'CreateDBInstance',
    'CreateImage',
    'CreateInternetGateway',
    'CreateLoadBalancer',
    'CreateNatGateway',
    'CreateNetworkAcl',
    'CreateNetworkInterface',
    'CreatePipeline',
    'CreateRole',
    'CreateRouteTable',
    'CreateSecurityGroup',
    'CreateSnapshot',
    'CreateStack',
    'CreateSubnet',
    'CreateTable',
    'CreateUser',
    'CreateVolume',
    'CreateVpc',
    'CreateVpnConnection',
    'CreateVpcPeeringConnection',
    'ImportImage',
    'ImportSnapshot',
    'RegisterImage',
    'RunInstances',
    'RunJobFlow'
)
and eventSource in (
    'autoscaling.amazonaws.com',
    'datapipeline.amazonaws.com',
    'dynamodb.amazonaws.com',
    'ec2.amazonaws.com',
    'elasticloadbalancing.amazonaws.com',
    'elasticmapreduce.amazonaws.com',
    'iam.amazonaws.com',
    'opsworks.amazonaws.com',
    'rds.amazonaws.com',
    's3.amazonaws.com'
)
and errorcode is null
```

### Deploy the Main CloudFormation template

In the same account as your CloudTrail S3 bucket deploy this Main CloudFormation template in a single region. This one CloudFormation stack, in combination with the IAM Role CloudFormation stack, will have the ability to tag all regions and more than one AWS account.

```bash
export REGION=ap-southeast-2 # set this to the region you plan to deploy to

wget https://raw.githubusercontent.com/GorillaStack/retro-tag/master/cloud_formation/autotag_retro_main-template.json

aws cloudformation create-stack \
	--template-body file://autotag_retro_main-template.json \
	--stack-name AutoTagRetro \
	--parameters ParameterKey=CloudTrailBucketName,ParameterValue=my-cloudtrail-bucket \
	   ParameterKey=CodeS3Bucket,ParameterValue=gorillastack-autotag-releases-$REGION \
      ParameterKey=CodeS3Path,ParameterValue=autotag-0.5.0.zip \
      ParameterKey=AutoTagDebugLogging,ParameterValue=Disabled \
      ParameterKey=AutoTagTagsCreateTime,ParameterValue=Enabled \
      ParameterKey=AutoTagTagsInvokedBy,ParameterValue=Enabled	--capabilities CAPABILITY_NAMED_IAM \
	--region $REGION
```

### Deploy the IAM Role CloudFormation template

In each AWS accounts where tagging will be applied deploy this IAM Role CloudFormation template in a single region. 

```bash
export REGION=ap-southeast-2               # set this to the region you plan to deploy to
export MAIN_AWS_ACCOUNT_NUMBER=11111111111 # set this to the AWS account number where we deployed the Main CloudFormation template

wget https://raw.githubusercontent.com/GorillaStack/retro-tag/master/cloud_formation/autotag_retro_role-template.json

aws cloudformation create-stack \
	--template-body file://autotag_retro_role-template.json \
	--stack-name AutoTagRetro \
	--parameters ParameterKey=MainAwsAccountNumber,ParameterValue=$MAIN_AWS_ACCOUNT_NUMBER \
	   ParameterKey=MainStackName,ParameterValue=AutoTagRetro \
	--capabilities CAPABILITY_NAMED_IAM \
	--region $REGION
```

### Tag Existing Resources

Use the `retro_tagging/retro_tag.rb` script to scan your environment for resources and then apply tagging to any resources that still exist.

The script will start by scanning each region for the AWS resources that exist then it will run the AutoTag lambda function against each CloudTrail log in S3 that includes at least one of the existing AWS resources.

```bash
$ bundle install # run this once to grab the ruby gem dependencies
Bundle complete! 6 Gemfile dependencies, 210 gems now installed.

export CSV_PATH="~/Downloads/MyAwsAccount_10292019.csv" # set this to the CSV exported from Athena
export BUCKET=my-cloudtrail-bucket  # set this to the name of the CloudTrail S3 bucket
export BUCKET_REGION=ap-southeast-2 # set this to the region of the CloudTrail S3 bucket
export SCAN_PROFILE=development     # set this to a profile of the account where tagging will be applied, this should match the data in the CSV
export LAMBDA_PROFILE=development   # set this to a profile of the account where the Main CloudFormation template was deployed
export LAMBDA_REGION=ap-southeast-2 # set this to the region where the Main CloudFormation template was deployed

./retro_tag.rb \
	--csv "$CSV_PATH" \
	--bucket $BUCKET \
	--bucket-region $BUCKET_REGION \
	--scan-profile "$SCAN_PROFILE" \
	--lambda-profile "$LAMBDA_PROFILE" \
	--lambda-region $LAMBDA_REGION
```

## Audit AutoTags

Use the `retro_tagging/audit_tag.rb` script to scan all supported resources for Auto Tags to view the overall coverage of the Retro Tag process.

The script will start by scanning each region for the AWS resources that exist and show a report. 

`./audit_all_tags.rb --profile development-readonly`

`./audit_all_tags.rb --access_key_id XXX --secret-access-key XXXXXX`

Each resource's tags are inspected for the existence of the `AutoTag_Creator` and `AutoTag_CreateTime` required tags. For each AWS resource a point is added to either the `Passed` or `Failed` column based on each of those required tags existence.

Example Output:

```bash
+---------------------------+--------+--------+----------+
|                 Auto-Tag Audit Summary                 |
+---------------------------+--------+--------+----------+
| Service                   | Passed | Failed | Coverage |
+---------------------------+--------+--------+----------+
| AutoScaling Groups        |     64 |      6 |   91.43% |
+---------------------------+--------+--------+----------+
| Data Pipelines            |     84 |     14 |   85.71% |
+---------------------------+--------+--------+----------+
| DynamoDB Tables           |    578 |    152 |   79.18% |
+---------------------------+--------+--------+----------+
| EC2 AMIs                  |    138 |     70 |   66.35% |
+---------------------------+--------+--------+----------+
| EC2 EIPs                  |     56 |    124 |   31.11% |
+---------------------------+--------+--------+----------+
| EC2 Instances             |    294 |     48 |   85.96% |
+---------------------------+--------+--------+----------+
| EC2 Snapshots             |     78 |    272 |   22.29% |
+---------------------------+--------+--------+----------+
| EC2 Volumes               |    470 |     58 |   89.02% |
+---------------------------+--------+--------+----------+
| EMR Clusters              |      2 |      0 |   100.0% |
+---------------------------+--------+--------+----------+
| Elastic Load Balancing    |    100 |     38 |   72.46% |
+---------------------------+--------+--------+----------+
| Elastic Load Balancing V2 |      2 |      0 |   100.0% |
+---------------------------+--------+--------+----------+
| IAM Roles                 |    340 |     88 |   79.44% |
+---------------------------+--------+--------+----------+
| IAM Users                 |    292 |     46 |   86.39% |
+---------------------------+--------+--------+----------+
| OpsWorks Stacks           |     17 |      4 |   80.95% |
+---------------------------+--------+--------+----------+
| RDS Instances             |     27 |     12 |   69.23% |
+---------------------------+--------+--------+----------+
| S3 Buckets                |    160 |    170 |   48.48% |
+---------------------------+--------+--------+----------+
| Security Groups           |  1,036 |    520 |   66.58% |
+---------------------------+--------+--------+----------+
| VPC ENIs                  |    618 |    114 |   84.43% |
+---------------------------+--------+--------+----------+
| VPC Internet Gateways     |     62 |     20 |   75.61% |
+---------------------------+--------+--------+----------+
| VPC NAT Gateways          |     26 |      4 |   86.67% |
+---------------------------+--------+--------+----------+
| VPC Network ACLs          |     14 |     90 |   13.46% |
+---------------------------+--------+--------+----------+
| VPC Peering Connections   |     54 |      6 |    90.0% |
+---------------------------+--------+--------+----------+
| VPC Route Tables          |    170 |    122 |   58.22% |
+---------------------------+--------+--------+----------+
| VPC Subnets               |    386 |     84 |   82.13% |
+---------------------------+--------+--------+----------+
| VPCs                      |     68 |     18 |   79.07% |
+---------------------------+--------+--------+----------+
| VPNs                      |     10 |      2 |   83.33% |
+---------------------------+--------+--------+----------+
```

## Contributing

If you have questions, feature requests or bugs to report, please do so on [the issues section of our github repository](https://github.com/GorillaStack/retro-tag/issues).

If you are interested in contributing, please get started by forking our GitHub repository and submit a pull-request.

