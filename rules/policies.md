# Policies

### DenyCreateWithoutProjectTag

Chặn tạo resource nếu không có tag project.

```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyCreateWithoutProjectTag",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "rds:CreateDBInstance",
                "rds:CreateDBCluster",
                "s3:CreateBucket",
                "lambda:CreateFunction",
                "eks:CreateCluster",
                "ecs:CreateCluster",
                "elasticache:CreateCacheCluster",
                "dynamodb:CreateTable"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/project": "true",
                    "aws:ResourceTag/project": "true"
                },
                "StringEquals": {
                    "aws:ResourceAccount": "${aws:PrincipalAccount}"
                }
            }
        }
    ]
}
```

### DenyCreateWithoutEnvTag

Policy này dùng để bắt buộc người dùng phải gắn tag env khi tạo resource AWS, và giá trị env chỉ được phép là `dev`, `stage` hoặc `prod`.

```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyCreateWithoutRequiredTags",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateVolume",
                "ec2:CreateSnapshot",
                "rds:CreateDBInstance",
                "rds:CreateDBCluster",
                "s3:CreateBucket",
                "lambda:CreateFunction",
                "eks:CreateCluster",
                "ecs:CreateCluster",
                "elasticache:CreateCacheCluster",
                "dynamodb:CreateTable"
            ],
            "Resource": "*",
            "Condition": {
                "Null": {
                    "aws:RequestTag/env": "true"
                }
            }
        },
        {
            "Sid": "DenyInvalidEnvValue",
            "Effect": "Deny",
            "Action": [
                "ec2:RunInstances",
                "ec2:CreateVolume",
                "rds:CreateDBInstance",
                "rds:CreateDBCluster",
                "lambda:CreateFunction",
                "eks:CreateCluster",
                "ecs:CreateCluster",
                "elasticache:CreateCacheCluster",
                "dynamodb:CreateTable"
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestTag/env": [
                        "dev",
                        "prod"
                    ]
                }
            }
        }
    ]
}
```

### DenyDeleteResourcesEnvProd

Policy này dùng để bảo vệ resource production, cụ thể:
Policy này sẽ chặn các thao tác nguy hiểm nếu resource có tag `env=prod` hoặc `Env=Prod` cho service: EC2, RDS, S3, Lambda, Cloudfront, DynamoDB, ECS, ElastiCache, Tag, KMS, IAM, Secret Manager, Backup



```text
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDeleteProdResources-key-env",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:StopInstances",
        "ec2:DeleteVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSubnet",
        "ec2:DeleteVpc",
        "ec2:DeleteNetworkInterface",
        "ec2:DeleteKeyPair",
        "ec2:DeleteNatGateway",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteRouteTable",
        "ec2:ReleaseAddress",
        "ec2:DeleteVpcPeeringConnection",
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:DeleteDBClusterSnapshot",
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "lambda:DeleteFunction",
        "lambda:DeleteFunctionUrlConfig",
        "lambda:DeleteLayerVersion",
        "cloudfront:DeleteDistribution",
        "cloudfront:DeleteFunction",
        "cloudfront:DeleteKeyGroup",
        "cloudfront:DeletePublicKey",
        "cloudfront:DeleteOriginAccessControl",
        "cloudfront:DeleteCloudFrontOriginAccessIdentity",
        "eks:DeleteCluster",
        "eks:DeleteNodegroup",
        "eks:DeleteFargateProfile",
        "ecs:DeleteCluster",
        "ecs:DeleteService",
        "ecs:DeleteTaskDefinitions",
        "ecs:DeleteCapacityProvider",
        "elasticache:DeleteCacheCluster",
        "elasticache:DeleteReplicationGroup",
        "elasticache:DeleteCacheSubnetGroup",
        "elasticache:DeleteSnapshot",
        "dynamodb:DeleteTable",
        "dynamodb:DeleteBackup",
        "kms:ScheduleKeyDeletion",
        "kms:DisableKey",
        "kms:DeleteAlias",
        "kms:DeleteImportedKeyMaterial",
        "secretsmanager:DeleteSecret",
        "ssm:DeleteParameter",
        "ssm:DeleteParameters",
        "iam:DeleteRole",
        "iam:DeleteUser",
        "iam:DeletePolicy",
        "iam:DeleteAccessKey",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:DeleteTargetGroup",
        "autoscaling:DeleteAutoScalingGroup",
        "route53:DeleteHostedZone",
        "logs:DeleteLogGroup",
        "backup:DeleteBackupVault",
        "backup:DeleteBackupPlan",
        "acm:DeleteCertificate",
        "sqs:DeleteQueue",
        "sns:DeleteTopic",
        "ecr:DeleteRepository"
      ],
      "Resource": "*",
      "Condition": {
        "StringEqualsIgnoreCase": {
          "aws:ResourceTag/env": [
            "prod",
            "production"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteProdResources-key-Env",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:StopInstances",
        "ec2:DeleteVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteSubnet",
        "ec2:DeleteVpc",
        "ec2:DeleteNetworkInterface",
        "ec2:DeleteKeyPair",
        "ec2:DeleteNatGateway",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteRouteTable",
        "ec2:ReleaseAddress",
        "ec2:DeleteVpcPeeringConnection",
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:DeleteDBClusterSnapshot",
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "lambda:DeleteFunction",
        "lambda:DeleteFunctionUrlConfig",
        "lambda:DeleteLayerVersion",
        "cloudfront:DeleteDistribution",
        "cloudfront:DeleteFunction",
        "cloudfront:DeleteKeyGroup",
        "cloudfront:DeletePublicKey",
        "cloudfront:DeleteOriginAccessControl",
        "cloudfront:DeleteCloudFrontOriginAccessIdentity",
        "eks:DeleteCluster",
        "eks:DeleteNodegroup",
        "eks:DeleteFargateProfile",
        "ecs:DeleteCluster",
        "ecs:DeleteService",
        "ecs:DeleteTaskDefinitions",
        "ecs:DeleteCapacityProvider",
        "elasticache:DeleteCacheCluster",
        "elasticache:DeleteReplicationGroup",
        "elasticache:DeleteCacheSubnetGroup",
        "elasticache:DeleteSnapshot",
        "dynamodb:DeleteTable",
        "dynamodb:DeleteBackup",
        "kms:ScheduleKeyDeletion",
        "kms:DisableKey",
        "kms:DeleteAlias",
        "kms:DeleteImportedKeyMaterial",
        "secretsmanager:DeleteSecret",
        "ssm:DeleteParameter",
        "ssm:DeleteParameters",
        "iam:DeleteRole",
        "iam:DeleteUser",
        "iam:DeletePolicy",
        "iam:DeleteAccessKey",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:DeleteTargetGroup",
        "autoscaling:DeleteAutoScalingGroup",
        "route53:DeleteHostedZone",
        "logs:DeleteLogGroup",
        "backup:DeleteBackupVault",
        "backup:DeleteBackupPlan",
        "acm:DeleteCertificate",
        "sqs:DeleteQueue",
        "sns:DeleteTopic",
        "ecr:DeleteRepository"
      ],
      "Resource": "*",
      "Condition": {
        "StringEqualsIgnoreCase": {
          "aws:ResourceTag/Env": [
            "prod",
            "production"
          ]
        }
      }
    },
    {
      "Sid": "DenyRemoveEnvTagFromProd",
      "Effect": "Deny",
      "Action": [
        "ec2:DeleteTags",
        "rds:RemoveTagsFromResource",
        "s3:DeleteBucketTagging",
        "lambda:UntagResource",
        "eks:UntagResource",
        "ecs:UntagResource",
        "elasticache:RemoveTagsFromResource",
        "dynamodb:UntagResource",
        "cloudfront:UntagResource",
        "kms:UntagResource",
        "secretsmanager:UntagResource",
        "ssm:RemoveTagsFromResource",
        "iam:UntagRole",
        "iam:UntagUser",
        "iam:UntagPolicy",
        "elasticloadbalancing:RemoveTags",
        "autoscaling:DeleteTags",
        "route53:ChangeTagsForResource",
        "logs:UntagLogGroup",
        "backup:UntagResource",
        "acm:RemoveTagsFromCertificate",
        "sqs:UntagQueue",
        "sns:UntagResource",
        "ecr:UntagResource",
        "tag:UntagResources"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringEqualsIgnoreCase": {
          "aws:TagKeys": [
            "env",
            "project"
          ]
        }
      }
    }
  ]
}
```

### Cloudwatch Log (Write/Put)

Name: CloudWatchLogsWritePolicy

```text
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents",
                "logs:DescribeLogStreams",
                "logs:DescribeLogGroups"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
```



