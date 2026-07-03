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
Policy này sẽ chặn các thao tác nguy hiểm nếu resource có tag `env=prod` hoặc `Env=Prod` cho service: EC2, RDS, S3, Lambda, Cloudfront, DynamoDB, ECS, ElastiCache, Tag

```text
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyStopTerminateDeleteEC2Prod",
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
        "ec2:DeleteKeyPair"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteRDSProd",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:DeleteDBClusterSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteS3Prod",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteObject",
        "s3:DeleteObjectVersion"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteLambdaProd",
      "Effect": "Deny",
      "Action": [
        "lambda:DeleteFunction",
        "lambda:DeleteFunctionUrlConfig",
        "lambda:DeleteLayerVersion"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteCloudFrontProd",
      "Effect": "Deny",
      "Action": [
        "cloudfront:DeleteDistribution",
        "cloudfront:DeleteFunction",
        "cloudfront:DeleteKeyGroup",
        "cloudfront:DeletePublicKey",
        "cloudfront:DeleteOriginAccessControl",
        "cloudfront:DeleteCloudFrontOriginAccessIdentity"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteEKSProd",
      "Effect": "Deny",
      "Action": [
        "eks:DeleteCluster",
        "eks:DeleteNodegroup",
        "eks:DeleteFargateProfile"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteECSProd",
      "Effect": "Deny",
      "Action": [
        "ecs:DeleteCluster",
        "ecs:DeleteService",
        "ecs:DeleteTaskDefinitions",
        "ecs:DeleteCapacityProvider"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteElastiCacheProd",
      "Effect": "Deny",
      "Action": [
        "elasticache:DeleteCacheCluster",
        "elasticache:DeleteReplicationGroup",
        "elasticache:DeleteCacheSubnetGroup",
        "elasticache:DeleteSnapshot"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyDeleteDynamoDBProd",
      "Effect": "Deny",
      "Action": [
        "dynamodb:DeleteTable",
        "dynamodb:DeleteBackup"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
          ]
        }
      }
    },
    {
      "Sid": "DenyRemoveEnvProjectTagFromProd",
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
        "tag:UntagResources"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/env": [
            "prod",
            "Prod"
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

