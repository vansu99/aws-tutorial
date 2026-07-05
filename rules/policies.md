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

Protects production resources from accidental deletion, termination, stopping, or other destructive actions based on the env/Env tag with values prod or production. It also prevents modification or removal of protected tags such as env and project on production resources.

```text
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDestructiveActionsOnProdResources",
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

        "rds:StopDBInstance",
        "rds:StopDBCluster",
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:DeleteDBClusterSnapshot",

        "lambda:DeleteFunction",
        "lambda:DeleteFunctionUrlConfig",

        "cloudfront:DeleteDistribution",
        "cloudfront:DeleteFunction",

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

        "kms:ScheduleKeyDeletion",
        "kms:DisableKey",
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

        "logs:DeleteLogGroup",

        "backup:DeleteBackupVault",
        "backup:DeleteBackupPlan",
        "backup:DeleteRecoveryPoint",

        "acm:DeleteCertificate",

        "sqs:DeleteQueue",
        "sqs:PurgeQueue",

        "sns:DeleteTopic",

        "ecr:DeleteRepository",
        "ecr:BatchDeleteImage"
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
      "Sid": "DenyChangingProtectedTagsOnProdResources",
      "Effect": "Deny",
      "Action": [
        "ec2:CreateTags",
        "ec2:DeleteTags",

        "rds:AddTagsToResource",
        "rds:RemoveTagsFromResource",

        "lambda:TagResource",
        "lambda:UntagResource",

        "eks:TagResource",
        "eks:UntagResource",

        "ecs:TagResource",
        "ecs:UntagResource",

        "elasticache:AddTagsToResource",
        "elasticache:RemoveTagsFromResource",

        "dynamodb:TagResource",
        "dynamodb:UntagResource",

        "cloudfront:TagResource",
        "cloudfront:UntagResource",

        "kms:TagResource",
        "kms:UntagResource",

        "secretsmanager:TagResource",
        "secretsmanager:UntagResource",

        "ssm:AddTagsToResource",
        "ssm:RemoveTagsFromResource",

        "iam:TagRole",
        "iam:UntagRole",
        "iam:TagUser",
        "iam:UntagUser",
        "iam:TagPolicy",
        "iam:UntagPolicy",

        "elasticloadbalancing:AddTags",
        "elasticloadbalancing:RemoveTags",

        "autoscaling:CreateOrUpdateTags",
        "autoscaling:DeleteTags",

        "logs:TagResource",
        "logs:UntagResource",

        "backup:TagResource",
        "backup:UntagResource",

        "acm:AddTagsToCertificate",
        "acm:RemoveTagsFromCertificate",

        "sqs:TagQueue",
        "sqs:UntagQueue",

        "sns:TagResource",
        "sns:UntagResource",

        "ecr:TagResource",
        "ecr:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEqualsIgnoreCase": {
          "aws:ResourceTag/env": [
            "prod",
            "production"
          ]
        },
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



