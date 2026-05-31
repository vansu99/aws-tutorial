# Hướng dẫn cấu hình CloudWatch Agent

Hướng dẫn cài đặt và cấu hình Amazon CloudWatch Agent để thu thập metric RAM (memory) và Disk trên các hệ điều hành: CentOS 7, Amazon Linux 2, Amazon Linux 2023, Ubuntu.

## Yêu cầu

###  IAM Role gắn vào EC2

- `CloudWatchAgentServerPolicy` : cho phép agent push metric / log lên CloudWatch.
- `AmazonSSMManagedInstanceCore` :  (khuyến nghị) cho phép quản lý agent qua SSM Parameter Store và Run Command.

## Cài đặt CloudWatch Agent

### Amazon Linux 2 / Amazon Linux 2023

```
sudo yum install -y amazon-cloudwatch-agent
```

### CentOS 7

CentOS 7 không có sẵn package, tải trực tiếp từ S3:

```
sudo yum install -y wget
wget https://s3.amazonaws.com/amazoncloudwatch-agent/centos/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm
```

Kiểm tra cài đặt thành công:

```
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

## File cấu hình thu thập Memory & Disk

Tạo file cấu hình tại `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json` :

```
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "aggregation_dimensions": [
      ["InstanceId"],
      ["AutoScalingGroupName"]
    ],
    "metrics_collected": {
      "mem": {
        "measurement": [
          "mem_used_percent",
          "mem_available_percent"
        ],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "/"
        ],
        "drop_device": true,
        "ignore_file_system_types": [
          "sysfs",
          "devtmpfs",
          "tmpfs",
          "overlay",
          "squashfs",
          "nsfs",
          "proc"
        ]
      }
    }
  }
}
```

## Khởi động và đăng ký service

### Apply config và start agent

```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

Chạy lệnh tiếp theo:

```
sudo systemctl stop amazon-cloudwatch-agent
sudo systemctl start amazon-cloudwatch-agent
```

Nếu có chỉnh sửa file config thì chạy lệnh restart service:

```
sudo systemctl restart amazon-cloudwatch-agent
```

## Lệnh quản lý thường dùng

```
# Trạng thái
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status

# Stop
sudo systemctl stop amazon-cloudwatch-agent

# Start
sudo systemctl start amazon-cloudwatch-agent

# Restart
sudo systemctl restart amazon-cloudwatch-agent
```

## Kiểm tra metric trên CloudWatch

1. Mở CloudWatch Console → Metrics → All metrics.
2. Chọn namespace CWAgent.
3. Mở dimension group InstanceId.
4. Bạn sẽ thấy 2 metric:
  - mem_used_percent
  - disk_used_percent

Cả hai cùng dimension InstanceId, có thể chọn cùng lúc để vẽ trên một biểu đồ.


