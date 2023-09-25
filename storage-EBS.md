## Elastic Block Storage

EBS là dịch vụ lưu trữ dạng block. EBS sẽ chỉ định dung lượng ổ địa cho người dùng (volume), và dung lượng này có thể được mã hoá bằng KMS. 

Với máy chủ EC2, khi nhìn thấy khối thiết bị (block device) sẽ tạo hệ thống tập tin trên thiết bị đó. 

Bộ lưu trữ sẽ được tạo trong 1 AZ => AZ resilient

EBS được đính kèm với 1 máy chủ (instance) qua storage network (mạng lưới lưu trữ)

EBS => persistent - nghĩa là nó có thể được gỡ đính kèm với 1 máy chủ này và chuyển qua đính kèm với máy chủ khác. 

Có thể đổi dữ liệu từ AZ này qua AZ khác bằng cách Snapshot vào S3. 

### EBS Volume Types

#### General Purpose SSD - GP2
- Dung lượng có thể bắt đầu từ **1GB -> 16TB**
- IO 'Credit' Bucket Capacity 5.4 million IO credits 
- Baseline performance ** 3 IO credits per second**
- GP2 thường được dùng là dung lượng bắt đầu, dánh cho ứng dụng tương tác có độ trễ thấp. 

1. GP2:

2. GP3:
- Tốc độ cơ sở (baseline rate) là 3000 IOPS và 125 MiB/s


#### Provisioned IOPS SSD
- IOPS có thể được điều chỉnh không tuỳ thuộc vào kích thước
1. IO1
2. IO2

#### HDD-Based

### Instance Store Volumes

Block storage devices - khối thiết bị lưu trữ có sẵn cho mỗi instance, và là nền tảng cho các hệ thống file

Volume này được kết nối vật lý tới 1 máy chủ EC2 và chỉ instance trên máy chủ đó mới truy cập được. 

Phí của volume này được tính cùng với mỗi EC2 instance. 

**Instance Store Volume phải được đính kèm với máy khi khởi động (launch)**

Instance Store Volume là bộ lưu trữ tạm thời. Nếu máy cần được bảo trì có thể được chuyển qua máy chủ khác thì lúc đó dữ liệu lưu trong Volume ở máy chủ cũ sẽ mất đi.

Lợi ích khi dùng Instance Store Volume:
 - Tốc độ xử lý nhanh hơn - hiệu suất cao

#### Instance Store vs EBS

- Cần lưu trữ lâu dài => EBS
- Cần bền => EBS
- Cần lưu trữ dữ liệu lâu hơn vòng đời của máy (instance) => EBS
- Cần hiệu suất cao => Tuỳ
- Giá cả thấp => Instance Store

- Nếu cần giá thấp = ST1 hoặc SC1
- Cần thông lượng (throughput) hoặc trực tiếp (streaming) = ST1

- GP2/2 - tới 16,000 IOPS
- IO1/2 - tới 64,000 IOPS
- RAID0 + EBS - tới 260,000 IOPS
- Instance Store - nhiều hơn 260,000 IOPS

### EBS Snapshot
EBS Snapshot là cách để backup dữ liệu, và cũng là cách để copy dữ liệu từ AZ này qua AZ khác dùng S3 để làm cầu nối. 

Snapshots được lưu tại S3, nghĩa là snapshot sẽ là regional resilient

Snapshots sẽ copy lượng dữ liêụ **đã** được dùng trong EBS. Sau snapshot đầu tiên thì các snapshot sau chỉ copy những dữ liệu mới, chưa được cập nhật. 

Snapshots có thể dùng để tạo volumes mới có cùng dữ liệu đã được lưu. Nhưng sẽ tốn thời gian, bởi vì dữ liệu sẽ được truyền từ từ. Hoặc có thể dùng Fast Snapshot Restore để truyền dữ liệu liền khi được restore vào máy mới.

Có thể có **50** snapshots trong một khu vực

Snapshots có thể được dùng để tạo bản copy ở một region (khu vực) khác.

Snapshot được tính phí theo Gigabyte mỗi tháng và chỉ tính với dung lượng đã được sử dụng. 


Sample Code demo:

```
# listing all block storage in instance
lsblk

# checking if there is a file system mounted
sudo file -s /dev/xvdf

# creating a file system on block device '/dev/xvdf' with type 'xfs'
sudo mkfs -t xfs /dev/xvdf

# creating a folder in file system
sudo mkdir /ebstest

# mounting the volume to the file system
sudo mount /dev/xvdf /ebstest
cd /ebstest
sudo nano amazingtestfile.txt
add a message
save and exit
ls -la

f63ed318-601a-4463-9b38-da2c11cd0a14

```

### EBS Encryption

EBS Encryption được dùng để mã hoá dữ liệu tại chỗ.

KMS sẽ được dùng để mã hoá dữ liệu trên EBS bằng cách dùng CMK để tạo ra DEK và DEK sẽ được mã hoá và lưu tại ổ cứng của EBS. Chỉ khi cần lưu dữ liệu thì DEK sẽ được giải mã và được lưu tạm thời tại EC2 Host. 

Khi snapshot được tạo từ EBS, snapshot cũng sẽ được mã hoá bằng DEK đã được dùng cho volume đó và volume mới được tạo từ snapshot cũng sẽ dùng cùng DEK để mã hoá. 

OS (hệ điều hành) sẽ không biết dữ liệu đã được mã hoá.
