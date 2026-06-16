# Deny-Prod-Dangerous-Actions-Guardrail

- Name: Deny-Prod-Dangerous-Actions-Guardrail
- Description: Deny dangerous delete/update actions on production resources tagged Env=prod and protect critical AWS control-plane services.

```text
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyProdEC2DangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:StopInstances",
        "ec2:DeleteVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteSecurityGroup",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:DeleteVpc",
        "ec2:DeleteSubnet",
        "ec2:DeleteRouteTable",
        "ec2:DeleteInternetGateway",
        "ec2:DeleteNatGateway"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdRDSDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "rds:StopDBInstance",
        "rds:StopDBCluster",
        "rds:ModifyDBInstance",
        "rds:ModifyDBCluster",
        "rds:DeleteDBSnapshot",
        "rds:DeleteDBClusterSnapshot",
        "rds:DeleteDBParameterGroup",
        "rds:DeleteDBSubnetGroup"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdSecretsManagerDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "secretsmanager:DeleteSecret",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecret",
        "secretsmanager:RotateSecret",
        "secretsmanager:CancelRotateSecret",
        "secretsmanager:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdLambdaDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "lambda:DeleteFunction",
        "lambda:DeleteFunctionConcurrency",
        "lambda:DeleteFunctionUrlConfig",
        "lambda:DeleteLayerVersion",
        "lambda:DeleteEventSourceMapping",
        "lambda:UpdateFunctionCode",
        "lambda:UpdateFunctionConfiguration",
        "lambda:PutFunctionConcurrency",
        "lambda:RemovePermission",
        "lambda:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdSQSDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "sqs:DeleteQueue",
        "sqs:PurgeQueue",
        "sqs:SetQueueAttributes",
        "sqs:RemovePermission",
        "sqs:UntagQueue"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdSNSDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "sns:DeleteTopic",
        "sns:SetTopicAttributes",
        "sns:RemovePermission",
        "sns:Unsubscribe",
        "sns:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdDynamoDBDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "dynamodb:DeleteTable",
        "dynamodb:UpdateTable",
        "dynamodb:DeleteBackup",
        "dynamodb:DeleteTableReplica",
        "dynamodb:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdECSDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "ecs:DeleteCluster",
        "ecs:DeleteService",
        "ecs:UpdateService",
        "ecs:DeregisterTaskDefinition",
        "ecs:DeleteTaskDefinitions",
        "ecs:StopTask",
        "ecs:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdCloudWatchLogsDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "logs:DeleteLogGroup",
        "logs:DeleteLogStream",
        "logs:DeleteRetentionPolicy",
        "logs:PutRetentionPolicy",
        "logs:UntagLogGroup"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdEFSAndElastiCacheDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "elasticfilesystem:DeleteFileSystem",
        "elasticfilesystem:DeleteMountTarget",
        "elasticfilesystem:DeleteAccessPoint",
        "elasticfilesystem:UntagResource",
        "elasticache:DeleteCacheCluster",
        "elasticache:DeleteReplicationGroup",
        "elasticache:ModifyCacheCluster",
        "elasticache:ModifyReplicationGroup",
        "elasticache:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdBackupDangerousActionsByTag",
      "Effect": "Deny",
      "Action": [
        "backup:DeleteBackupVault",
        "backup:DeleteBackupVaultAccessPolicy",
        "backup:DeleteBackupVaultLockConfiguration",
        "backup:DeleteRecoveryPoint",
        "backup:DeleteBackupPlan",
        "backup:UntagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyProdS3BucketDeleteByBucketTag",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "s3:DeleteBucketPolicy",
        "s3:PutBucketPolicy",
        "s3:PutBucketAcl",
        "s3:PutBucketPublicAccessBlock",
        "s3:PutBucketVersioning",
        "s3:PutLifecycleConfiguration",
        "s3:DeleteBucketWebsite",
        "s3:UntagResource"
      ],
      "Resource": "arn:aws:s3:::*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        }
      }
    },
    {
      "Sid": "DenyS3ObjectDeleteOnKnownProductionBuckets",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteObject",
        "s3:DeleteObjectVersion",
        "s3:PutObjectAcl",
        "s3:PutObjectVersionAcl"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR-PROD-BUCKET-NAME-1/*",
        "arn:aws:s3:::YOUR-PROD-BUCKET-NAME-2/*"
      ]
    },
    {
      "Sid": "DenyVeryHighRiskControlPlaneActionsAlways",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteUser",
        "iam:DeleteRole",
        "iam:DeletePolicy",
        "iam:DeletePolicyVersion",
        "iam:DetachUserPolicy",
        "iam:DetachRolePolicy",
        "iam:PutUserPolicy",
        "iam:PutRolePolicy",
        "iam:CreateAccessKey",
        "iam:UpdateAccessKey",
        "iam:DeleteAccessKey",

        "kms:ScheduleKeyDeletion",
        "kms:DisableKey",
        "kms:PutKeyPolicy",
        "kms:DeleteAlias",

        "route53:ChangeResourceRecordSets",
        "route53:DeleteHostedZone",

        "cloudfront:DeleteDistribution",
        "cloudfront:UpdateDistribution",
        "cloudfront:DeleteFunction",
        "cloudfront:UpdateFunction",

        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail",

        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder",
        "config:DeleteDeliveryChannel",

        "organizations:LeaveOrganization",
        "organizations:DeleteOrganization",
        "organizations:DetachPolicy",

        "budgets:DeleteBudget",
        "ce:DeleteAnomalyMonitor",
        "ce:DeleteAnomalySubscription"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyCreateWithoutEnvTag",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "rds:CreateDBCluster",
        "secretsmanager:CreateSecret",
        "lambda:CreateFunction",
        "sqs:CreateQueue",
        "sns:CreateTopic",
        "dynamodb:CreateTable",
        "ecs:CreateCluster",
        "ecs:CreateService",
        "elasticfilesystem:CreateFileSystem",
        "elasticache:CreateCacheCluster",
        "elasticache:CreateReplicationGroup",
        "backup:CreateBackupPlan",
        "backup:CreateBackupVault"
      ],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/Env": "true"
        }
      }
    },
    {
      "Sid": "DenyCreateWithInvalidEnvTagValue",
      "Effect": "Deny",
      "Action": [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "rds:CreateDBCluster",
        "secretsmanager:CreateSecret",
        "lambda:CreateFunction",
        "sqs:CreateQueue",
        "sns:CreateTopic",
        "dynamodb:CreateTable",
        "ecs:CreateCluster",
        "ecs:CreateService",
        "elasticfilesystem:CreateFileSystem",
        "elasticache:CreateCacheCluster",
        "elasticache:CreateReplicationGroup",
        "backup:CreateBackupPlan",
        "backup:CreateBackupVault"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotEquals": {
          "aws:RequestTag/Env": [
            "prod",
            "staging",
            "dev"
          ]
        }
      }
    },
    {
      "Sid": "DenyRemovingOrChangingEnvTagOnProdResources",
      "Effect": "Deny",
      "Action": [
        "tag:UntagResources",
        "tag:TagResources",
        "ec2:DeleteTags",
        "ec2:CreateTags",
        "rds:RemoveTagsFromResource",
        "rds:AddTagsToResource",
        "secretsmanager:UntagResource",
        "secretsmanager:TagResource",
        "lambda:UntagResource",
        "lambda:TagResource",
        "sqs:UntagQueue",
        "sqs:TagQueue",
        "sns:UntagResource",
        "sns:TagResource",
        "dynamodb:UntagResource",
        "dynamodb:TagResource",
        "ecs:UntagResource",
        "ecs:TagResource"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Env": "prod"
        },
        "ForAnyValue:StringEquals": {
          "aws:TagKeys": [
            "Env"
          ]
        }
      }
    }
  ]
}
```

Trong statement này:

```
"DenyS3ObjectDeleteOnKnownProductionBuckets"
```

Bạn phải đổi:

```
YOUR-PROD-BUCKET-NAME-1
YOUR-PROD-BUCKET-NAME-2
```

thành bucket production thật

```
"Resource": [
  "arn:aws:s3:::housecan-prod-files/*",
  "arn:aws:s3:::videoapp-prod-media/*"
]
```



