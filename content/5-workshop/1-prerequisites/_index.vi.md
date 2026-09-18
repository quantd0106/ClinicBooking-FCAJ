---
title: "5.1. Điều kiện tiên quyết"
weight: 1
chapter: false
---

# 5.1. ĐIỀU KIỆN TIÊN QUYẾT

## Mục tiêu

Trước khi thực hiện Workshop, cần chuẩn bị môi trường phát triển local, tài khoản AWS, công cụ triển khai và các quyền AWS cơ bản cần thiết. Việc chuẩn bị này giúp phát triển Backend tại local và triển khai hạ tầng Serverless bằng AWS SAM và AWS CloudFormation.

## Yêu cầu kiến thức

Chỉ cần hiểu cơ bản các nội dung sau, không yêu cầu kiến thức ở mức chuyên gia:

- REST API.
- Node.js và TypeScript.
- Cơ sở dữ liệu quan hệ và MySQL.
- Git.
- Các khái niệm AWS cơ bản.
- Kiến thức nền tảng về Amazon VPC và Security Group.

## Công cụ cần chuẩn bị

| Công cụ | Vai trò |
| --- | --- |
| Visual Studio Code | Chỉnh sửa mã nguồn Backend và các tệp hạ tầng. |
| Node.js | Chạy Backend tại local và hỗ trợ quy trình phát triển Node.js. |
| npm | Cài đặt và quản lý các dependency của dự án. |
| Git | Clone mã nguồn Backend và theo dõi thay đổi. |
| GitHub | Lưu trữ và cộng tác trên mã nguồn. |
| MySQL | Cung cấp cơ sở dữ liệu quan hệ cho môi trường phát triển local. |
| Prisma CLI | Làm việc với Prisma schema, migrations và database client được sinh từ schema. |
| Postman hoặc Swagger UI | Gửi yêu cầu API và kiểm tra API contract. |
| AWS CLI v2 | Kiểm tra danh tính AWS, region và thao tác với tài nguyên AWS. |
| AWS SAM CLI | Định nghĩa, build và triển khai ứng dụng Serverless. |
| Hugo Extended | Build website Workshop FCAJ này. |

Hugo Extended chỉ phục vụ website báo cáo và Workshop FCAJ, không phải thành phần runtime của Backend Clinic Appointment Booking System.

## Tài khoản AWS

1. Chuẩn bị tài khoản AWS để thực hiện Workshop.
2. Tránh sử dụng thông tin đăng nhập root cho hoạt động triển khai thường ngày.
3. Sử dụng danh tính AWS IAM cho CLI và triển khai, với quyền phù hợp cho các tài nguyên được định nghĩa bằng SAM/CloudFormation.
4. Cấu hình AWS CLI profile cho danh tính đó.
5. Sử dụng ap-southeast-1 — Asia Pacific (Singapore) trong Workshop.

Áp dụng quyền tối thiểu cần thiết và không đưa mã định danh tài khoản hoặc thông tin đăng nhập thực tế vào tài liệu Workshop.

## Kiểm tra môi trường

Kiểm tra các công cụ phát triển và triển khai đã được cài đặt:

```text
node --version
npm --version
git --version
aws --version
sam --version
```

Kiểm tra danh tính AWS bằng profile đã cấu hình:

```text
aws sts get-caller-identity --profile <your-profile>
```

Kiểm tra region của profile:

```text
aws configure get region --profile <your-profile>
```

Thay `<your-profile>` bằng tên AWS CLI profile tại máy local. Chỉ thực hiện kiểm tra danh tính tại local; kết quả chứa thông tin tài khoản và danh tính không nên công bố. Kết quả kiểm tra region cần là ap-southeast-1.

## Chuẩn bị mã nguồn

Clone hoặc tải mã nguồn Backend Clinic Appointment Booking System từ nơi lưu trữ mã nguồn hiện có của dự án. Mở thư mục dự án Backend và cài đặt dependency:

```text
npm install
```

Chạy lệnh trong dự án Backend, không chạy trong repository báo cáo Hugo. Workshop không giả định một URL repository Backend công khai.

## Lưu ý bảo mật

{{< notice warning >}}
Không commit các tệp .env. Không hard-code AWS access keys. Không công bố thông tin đăng nhập cơ sở dữ liệu. Sử dụng AWS Secrets Manager cho cấu hình runtime nhạy cảm, gồm thông tin đăng nhập cơ sở dữ liệu và các bí mật liên quan đến JWT. Không chuyển Amazon RDS sang chế độ công khai chỉ để thuận tiện thao tác.
{{< /notice >}}
