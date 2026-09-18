---
title: "5.6.3. Amazon RDS for MySQL"
weight: 3
chapter: false
---

# Amazon RDS for MySQL

## Cấu hình database đã đối chiếu

`ClinicDatabase` là `AWS::RDS::DBInstance`. `DatabaseSubnetGroup` là `AWS::RDS::DBSubnetGroup` chứa cả hai private subnet.

| Thiết lập | Template hiện tại |
|---|---|
| Engine | `mysql`; không đặt `EngineVersion` tường minh. |
| Instance class | Parameter `DatabaseInstanceClass`, mặc định `db.t4g.micro`; có thể ghi đè. |
| Storage | 20 GiB, `gp3`, có mã hóa. |
| Availability | `MultiAZ: false` (Single-AZ); không cố định một AZ cụ thể. |
| Truy cập công khai | `PubliclyAccessible: false`. |
| Cổng / Security Group | TCP 3306 / `DatabaseSecurityGroup`. |
| Subnet group | `DatabaseSubnetGroup`: `PrivateSubnetA`, `PrivateSubnetB`. |
| Thông tin xác thực | `ManageMasterUserPassword: true`; RDS quản lý mật khẩu trong Secrets Manager. |
| Backup | `BackupRetentionPeriod: 1` ngày. |
| Bảo trì | `AutoMinorVersionUpgrade: true`. |
| Log export | `error` và `slowquery` đến CloudWatch. |
| Tag snapshot | `CopyTagsToSnapshot: true`. |
| Xóa / thay thế | `DeletionPolicy: Snapshot`, `UpdateReplacePolicy: Snapshot`. |
| Deletion protection | Không đặt tường minh; không khẳng định cấu hình đang chạy. |

Trích đoạn ngắn các thuộc tính DB:

```yaml
Engine: mysql
AllocatedStorage: '20'
DBInstanceClass:
  Ref: DatabaseInstanceClass
MultiAZ: false
PubliclyAccessible: false
StorageEncrypted: true
StorageType: gp3
Port: 3306
ManageMasterUserPassword: true
```

DB subnet group cung cấp vị trí trong private subnet trên hai AZ. Cấu hình này **không** tạo database thứ hai hoặc standby khi `MultiAZ` là false.

## Vì sao chọn MySQL được quản lý

Lịch làm việc, lịch hẹn, người dùng và bác sĩ tạo thành workload quan hệ với các thay đổi mang tính giao dịch. RDS phù hợp mô hình này và cung cấp vận hành database dưới dạng managed service, tránh phải tự vận hành MySQL trên EC2. Private routing và MySQL ingress chỉ từ Lambda bảo vệ ranh giới mạng của database.

Single-AZ và instance class mặc định nhỏ phù hợp môi trường học tập/demo. Đây không phải thiết kế high availability cho production. Yêu cầu availability của production có thể cần Multi-AZ và quyết định capacity riêng.

Migration schema ứng dụng thuộc giai đoạn triển khai sau. Trang này không kết nối database hoặc chạy migration.

## Chi phí và tác động của retention

DB instance đang chạy phát sinh phí instance; storage và các khoản áp dụng khác được xem xét riêng. Tham khảo [bảng giá Amazon RDS for MySQL](https://aws.amazon.com/rds/mysql/pricing/). Không khẳng định số tiền dự kiến hoặc kết quả billing thực tế tại đây.

Policy snapshot khi xóa/thay thế có thể để lại snapshot sau khi xóa stack hoặc thay DB. Snapshot được giữ lại có thể tiếp tục phát sinh phí lưu trữ và cần quyết định retention/cleanup ở bước sau.

{{< notice warning >}}
Đây là database học tập/demo, không phải thiết kế high availability cho production. RDS có thể phát sinh chi phí liên tục khi đang chạy. Chi phí chi tiết và bước cleanup được trình bày sau; không thực hiện cleanup tại đây.
{{< /notice >}}

📷 Ảnh cần bổ sung: Cấu hình RDS riêng tư thể hiện Single-AZ, mã hóa, subnet group và public-access setting. Che endpoint, thông tin xác thực và định danh riêng tư.
