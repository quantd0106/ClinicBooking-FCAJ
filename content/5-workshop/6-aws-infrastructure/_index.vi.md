---
title: "5.6. Hạ tầng AWS"
weight: 6
chapter: false
---

# Hạ tầng AWS

Giai đoạn này chuyển yêu cầu của ứng dụng thành các tài nguyên mạng, cơ sở dữ liệu, lưu trữ, secrets và IAM trên AWS trước khi triển khai. Nguồn đối chiếu là `infrastructure/template.yaml` trong repository Backend, sử dụng transform `AWS::Serverless-2016-10-31`:

**AWS SAM template → CloudFormation → tài nguyên AWS**

Mã nguồn hiện tại khai báo **31 tài nguyên thuộc 19 loại trước SAM transform**. SAM có thể sinh thêm tài nguyên; đây là số lượng trong mã nguồn, không phải số lượng tài nguyên AWS đang chạy. Region dự kiến của workshop là **ap-southeast-1 (Asia Pacific — Singapore)**. Template sử dụng region của stack thay vì cố định Singapore.

| Thành phần | Mục đích |
|---|---|
| VPC / Subnets | Kết nối riêng tư cho Lambda và RDS trên hai Availability Zones. |
| Security Groups | Kiểm soát lưu lượng MySQL và dịch vụ AWS giữa các workload. |
| RDS MySQL | Lưu trữ quan hệ được quản lý cho lịch làm việc, lịch hẹn và dữ liệu liên quan. |
| Secrets Manager | Bảo vệ thông tin xác thực database và cấu hình ký JWT. |
| VPC Endpoints | Truy cập S3 và Secrets Manager mà không cần NAT Gateway. |
| S3 | Lưu file hồ sơ bác sĩ riêng tư và hỗ trợ upload trực tiếp có giới hạn. |
| IAM | Cấp quyền ghi log, đọc secret, upload và hỗ trợ kết nối VPC. |

## Nội dung hạ tầng

1. [5.6.1. Thiết kế VPC và mạng](1-vpc-network/)
2. [5.6.2. Security Groups](2-security-groups/)
3. [5.6.3. Amazon RDS for MySQL](3-rds-mysql/)
4. [5.6.4. Secrets Manager và VPC Endpoint](4-secrets-endpoints/)
5. [5.6.5. Amazon S3 riêng tư và IAM](5-s3-iam/)

{{< notice info >}}
Phần này chỉ mô tả thiết kế hạ tầng của môi trường học tập/demo. Triển khai, đóng gói, migration database, smoke test trên AWS và cleanup được trình bày sau. Không tạo hoặc kiểm tra tài nguyên AWS đang chạy trong phần này.
{{< /notice >}}

📷 Ảnh cần bổ sung: Tổng quan tài nguyên CloudFormation/VPC, đã che các định danh nhạy cảm.
