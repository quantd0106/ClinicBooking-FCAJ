---
title: "5.6.5. Amazon S3 riêng tư và IAM"
weight: 5
chapter: false
---

# Amazon S3 riêng tư và IAM

## Bảo vệ bucket

`ProfileFilesBucket` là `AWS::S3::Bucket` với các kiểm soát được định nghĩa trong mã nguồn:

| Thiết lập | Template hiện tại |
|---|---|
| Public access | Bật cả bốn tùy chọn Block Public Access. |
| Mã hóa | Server-side encryption `AES256` (SSE-S3). |
| Object ownership | `BucketOwnerEnforced`. |
| Bucket policy | Từ chối `s3:*` trên bucket và object khi `aws:SecureTransport` là false. |
| Multipart lifecycle | Hủy multipart upload chưa hoàn thành sau 1 ngày; không phải rule xóa file hồ sơ đã upload xong. |
| Xóa / thay thế | `DeletionPolicy: Retain`, `UpdateReplacePolicy: Retain`. |

Trích đoạn public-access thực tế:

```yaml
PublicAccessBlockConfiguration:
  BlockPublicAcls: true
  BlockPublicPolicy: true
  IgnorePublicAcls: true
  RestrictPublicBuckets: true
```

Bucket policy không có Allow công khai. GET object trực tiếp không xác thực không được làm lộ object riêng tư. Policy cũng **không** giới hạn mọi request vào một VPC endpoint; bucket riêng tư và truy cập chỉ trong VPC là hai kiểm soát khác nhau. Vì vậy, client được cấp quyền có thể dùng presigned POST qua HTTPS từ bên ngoài VPC.

## Upload có xác thực và giới hạn

Triển khai dùng **presigned POST**, trả về upload URL cùng form fields thay vì presigned PUT URL.

1. Bác sĩ gửi request có xác thực `POST /api/v1/files/presigned-upload`.
2. JWT và role guard chỉ cho phép **DOCTOR**. Backend kiểm tra caller có hồ sơ bác sĩ và validate `fileName`, `contentType`.
3. Server sinh object key và ký POST policy với giới hạn content-type và kích thước.
4. API trả về `objectKey`, `uploadUrl`, `formFields`, `expiresInSeconds` và `maxFileSizeBytes`.
5. Client upload file trực tiếp đến S3 bằng multipart POST form được trả về; S3 thực thi policy đã ký.

| Giới hạn | Hành vi đã đối chiếu |
|---|---|
| JPEG | `image/jpeg`, phần mở rộng `.jpg` hoặc `.jpeg`. |
| PNG | `image/png`, phần mở rộng `.png`. |
| WEBP | `image/webp`, phần mở rộng `.webp`. |
| Tên file | Tên file đơn được trim, tối đa 255 ký tự; không cho phép ký tự phân tách đường dẫn. |
| Khớp loại file | MIME type khai báo phải khớp phần mở rộng của tên file. |
| Kích thước upload thực | POST policy đã ký cho phép từ 1 byte đến **5 MiB** (`5 * 1024 * 1024` byte). |
| Hết hạn | Có thể cấu hình; mặc định của template và ứng dụng là 300 giây. |
| Key do server sinh | Mẫu `doctors/<doctor-id>/avatar/<random-uuid><extension>`; client không chọn toàn bộ key. |

API request không có trường kích thước và không mang binary của file. Backend đặt giới hạn kích thước trong policy đã ký; **S3 kiểm tra kích thước file thực tế được upload**. Kiểm tra MIME/phần mở rộng không đồng nghĩa kiểm tra binary signature của file.

Trích đoạn ngắn từ `src/files/s3-presigner.service.ts`:

```typescript
return createPresignedPost(this.client, {
  Bucket: bucket,
  Key: objectKey,
  Expires: expiresInSeconds,
  Fields: { 'Content-Type': contentType },
  Conditions: [
    ['eq', '$Content-Type', contentType],
    ['content-length-range', 1, maximumBytes],
  ],
});
```

Upload trực tiếp giúp binary không đi qua request API/Lambda trong use case này. Việc cấp quyền vẫn bắt đầu tại API có xác thực, và form được ký tạm thời bị giới hạn vào object key do server sinh.

## Quyền thực tế của Lambda execution role

`LambdaExecutionRole` trust `lambda.amazonaws.com`. Các inline policy:

| Policy | Quyền đã đối chiếu | Phạm vi / Giới hạn |
|---|---|---|
| `RestrictedCloudWatchLogs` | `logs:CreateLogStream`, `logs:PutLogEvents`. | Phạm vi log group của function; template tạo log group riêng. |
| `LambdaVpcNetworking` | Các thao tác EC2 tạo/mô tả/xóa network interface và gán/bỏ gán private IP. | `Resource: '*'`; quyền networking rộng hơn, không phải least privilege hoàn hảo. |
| `RuntimeSecrets` | `secretsmanager:GetSecretValue`. | Chỉ tham chiếu secret RDS được quản lý và `JwtSecret`. |
| `DoctorProfileUploads` | `s3:PutObject`. | Chỉ object `doctors/*` trong bucket; không cấp GetObject, ListBucket, DeleteObject hoặc S3 full access tại đây. |

`S3GatewayEndpoint` cho phép GetObject và PutObject cho prefix đó, nhưng endpoint policy không cấp GetObject cho Lambda role. IAM và quyền endpoint đều phải cho phép request. Hai CloudWatch log group được khai báo dùng retention 14 ngày.

Role đã giới hạn secrets, upload và logging. Hardening trong tương lai có thể gồm rà soát condition được hỗ trợ cho quyền EC2 networking rộng và xem lại SG egress rộng hơn ở 5.6.2. Đây là các cân nhắc thiết kế riêng; không thay đổi IAM hoặc SG trong bước tài liệu này.

{{< notice warning >}}
Giữ bucket riêng tư và coi giá trị presigned form là quyền truy cập tạm thời. Retention của bucket có thể giữ lại object sau khi xóa stack, nên kế hoạch cleanup sau cần xét phần lưu trữ được giữ lại. Không cleanup hoặc kiểm tra truy cập trên AWS tại đây.
{{< /notice >}}

<!-- TODO_SCREENSHOT: Cấu hình S3 Block Public Access và mã hóa, đã che tên bucket/định danh riêng tư. -->
