---
title: "5.2.1. Kiến trúc tổng thể"
weight: 1
chapter: false
---

# 5.2.1. Kiến trúc tổng thể

## Luồng xử lý yêu cầu chính

```text
Client
→ Amazon API Gateway
→ AWS Lambda
→ NestJS
→ Prisma ORM
→ Amazon RDS for MySQL
```

Amazon API Gateway cung cấp điểm tiếp nhận HTTP công khai. AWS Lambda chạy Backend Node.js; NestJS xử lý xác thực, RBAC và nghiệp vụ ứng dụng. Prisma ORM nằm trong Backend và chuyển thao tác truy cập dữ liệu thành truy vấn MySQL, không phải một dịch vụ AWS riêng biệt.

Backend sử dụng Node.js, TypeScript, NestJS, Prisma ORM và MySQL. JWT xác định người dùng đã xác thực, RBAC phân tách quyền PATIENT, DOCTOR và ADMIN, còn Swagger/OpenAPI mô tả API contract.

Lambda giúp tránh vận hành một máy chủ Backend EC2 thường trực. Amazon RDS for MySQL cung cấp cơ sở dữ liệu quan hệ được quản lý, phù hợp với dữ liệu giao dịch đặt lịch, gồm transaction và cơ chế khóa để ngăn đặt lịch trùng.

## Sơ đồ kiến trúc

Ranh giới VPC biểu diễn Amazon VPC. Lambda được gắn vào private application subnets, còn RDS nằm trong private database subnets. Các endpoint trong sơ đồ cung cấp đường truy cập đã thiết kế đến Amazon S3 và AWS Secrets Manager.

{{< mermaid >}}
graph TD
  Client["Client"] --> API["Amazon API Gateway"]
  API --> Backend
  subgraph VPC
    Backend["AWS Lambda / NestJS"]
    Database["Amazon RDS for MySQL"]
    S3Endpoint["S3 Gateway VPC Endpoint"]
    SecretsEndpoint["Secrets Manager Interface Endpoint"]
    Backend -->|Prisma ORM - TCP 3306| Database
    Backend --> S3Endpoint
    Backend --> SecretsEndpoint
  end
  S3Endpoint --> Storage["Amazon S3 - private bucket"]
  SecretsEndpoint --> Secrets["AWS Secrets Manager"]
  Backend --> Logs["Amazon CloudWatch"]
{{< /mermaid >}}

## Các đường truy cập dịch vụ hỗ trợ

| Đường truy cập | Vai trò |
| --- | --- |
| Lambda → AWS Secrets Manager | Đọc cấu hình runtime nhạy cảm, gồm thông tin đăng nhập cơ sở dữ liệu và cấu hình bí mật liên quan đến JWT, thay vì đưa secrets vào mã nguồn. |
| Lambda → Amazon S3 | Hỗ trợ tải lên tệp và hồ sơ riêng tư bằng presigned URL. Amazon S3 phù hợp với lưu trữ đối tượng và tệp. |
| Lambda → Amazon CloudWatch | Cung cấp log thực thi và giám sát để theo dõi hoạt động ứng dụng. |

AWS IAM kiểm soát quyền của Lambda và hoạt động triển khai. AWS SAM định nghĩa, build và triển khai hạ tầng Serverless qua AWS CloudFormation, giúp hạ tầng có thể được triển khai lặp lại.

## Mạng và quy tắc truy cập

- Lambda được gắn vào private VPC subnets.
- RDS sử dụng private database subnets và không được truy cập công khai.
- RDS Security Group chỉ nhận MySQL TCP 3306 từ Lambda Security Group.
- Bucket Amazon S3 được giữ riêng tư và bật Block Public Access.
- Lambda truy cập Amazon S3 qua S3 Gateway VPC Endpoint.
- Lambda truy cập AWS Secrets Manager qua Secrets Manager Interface Endpoint.
- Kiến trúc cuối cùng không có NAT Gateway.

RDS riêng tư và Security Group giới hạn truy cập giúp giảm khả năng cơ sở dữ liệu bị tiếp cận trực tiếp. Hai VPC Endpoints cung cấp đường truy cập dịch vụ theo thiết kế mà không cần thêm NAT Gateway.

## Cấu hình triển khai

| Thành phần | Cấu hình |
| --- | --- |
| Region | ap-southeast-1 — Asia Pacific (Singapore) |
| RDS | MySQL; db.t4g.micro; Single-AZ; dung lượng 20 GiB; riêng tư; PubliclyAccessible = false |
| Lambda | Node.js runtime; bộ nhớ 512 MiB; không khai báo reserved concurrency trong bản triển khai cuối cùng |
| Hạ tầng | AWS SAM và AWS CloudFormation |

Thiết kế RDS này dùng Single-AZ, không mô tả cơ sở dữ liệu được triển khai Multi-AZ.
