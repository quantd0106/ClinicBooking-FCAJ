---
title: "5.6.4. Secrets Manager và VPC Endpoint"
weight: 4
chapter: false
---

# Secrets Manager và VPC Endpoint

## Quyền quản lý secret và các tham chiếu

Cấu hình runtime nhạy cảm không được hard-code trong ứng dụng hoặc nhúng plaintext vào cấu hình biến môi trường Lambda thông thường. Template dùng hai nguồn secret:

| Nguồn | Quyền quản lý | Tham chiếu runtime |
|---|---|---|
| Thông tin xác thực database | RDS quản lý qua `ManageMasterUserPassword: true`; không khai báo resource DB secret riêng. | `ClinicDatabase.MasterUserSecret.SecretArn`, truyền qua `DATABASE_SECRET_ARN`. |
| Cấu hình ký JWT | `JwtSecret`, một `AWS::SecretsManager::Secret`. | Tham chiếu tượng trưng `Ref: JwtSecret`, truyền qua `JWT_SECRET_ARN`. |

Giá trị được bảo vệ ở mức khái niệm là `<database-credentials>` và `<jwt-secret>`. Không cần giá trị secret thực để giải thích thiết kế này.

`DB_HOST`, `DB_NAME` và `DB_PORT` cung cấp metadata cấu hình database; không chép lại giá trị triển khai thực tại đây. Cấu hình environment thông thường trong template chứa tham chiếu secret, không chứa connection string database hoặc secret ký JWT dưới dạng plaintext.

## Hành vi khởi động Lambda

`src/lambda.ts` gọi `resolveLambdaRuntimeConfiguration` trước khi import và khởi tạo ứng dụng NestJS. Trong `src/aws/runtime-secrets.ts`, resolver đọc secret được tham chiếu qua Secrets Manager client khi chưa có giá trị runtime tương ứng.

Resolver tạo cấu hình kết nối Prisma đã encode trong bộ nhớ của process từ metadata DB và thông tin xác thực được bảo vệ, với connection limit là 2 và pool/connect timeout là 5 giây. Cấu hình JWT cũng được cung cấp trong bộ nhớ trước khi khởi tạo ứng dụng. Promise khởi tạo và Lambda handler được cache để tái sử dụng trong execution environment hiện có.

Đây là resolve lúc khởi động, không phải đọc secret cho mỗi API request. Mã nguồn không triển khai refresh cache cho mỗi request sau khi rotate secret; không giả định có hành vi này.

`RuntimeSecrets` trong execution role cấp `secretsmanager:GetSecretValue` chỉ cho hai tham chiếu secret tượng trưng:

```yaml
Effect: Allow
Action:
  - secretsmanager:GetSecretValue
Resource:
  - Fn::GetAtt:
      - ClinicDatabase
      - MasterUserSecret.SecretArn
  - Ref: JwtSecret
```

Policy của interface endpoint cũng giới hạn action này cho hai secret đó. `Principal: '*'` không tự cấp quyền độc lập cho caller: IAM role và endpoint policy đều phải cho phép truy cập.

## Hai loại endpoint

| Endpoint | Loại / Cơ chế | Vị trí thực tế | Ranh giới truy cập |
|---|---|---|---|
| `S3GatewayEndpoint` | Gateway; đường truy cập dịch vụ S3 dựa trên route table. | Liên kết với `PrivateRouteTable`. | Endpoint policy cho phép `s3:GetObject` và `s3:PutObject` đối với object `doctors/*` trong bucket; vẫn cần IAM. |
| `SecretsManagerEndpoint` | Interface; network interface riêng tư của PrivateLink với private DNS. | ENI tại `PrivateSubnetA/B`. | Endpoint SG nhận TCP 443 từ Lambda SG; endpoint policy giới hạn đọc secret. |

Trích đoạn ngắn resource interface endpoint dưới đây; không chép policy trong trích đoạn này:

```yaml
SecretsManagerEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    PrivateDnsEnabled: true
    SecurityGroupIds:
      - Ref: AwsServiceEndpointSecurityGroup
    ServiceName:
      Fn::Sub: com.amazonaws.${AWS::Region}.secretsmanager
    SubnetIds:
      - Ref: PrivateSubnetA
      - Ref: PrivateSubnetB
    VpcEndpointType: Interface
    VpcId:
      Ref: ClinicVpc
```

Private DNS giúp tên dịch vụ Secrets Manager được SDK sử dụng resolve qua endpoint. Khi không có NAT và private default route Internet, đây là đường truy cập được thiết kế cho việc đọc secret của Lambda. S3 gateway dùng routing thay vì Security Group của interface endpoint.

## Chi phí và lưu ý retention

Interface endpoint được tính phí theo thời gian provisioned tại mỗi AZ, cộng phí xử lý dữ liệu áp dụng; chi phí không chỉ phụ thuộc hoạt động API request. Tham khảo [bảng giá AWS PrivateLink](https://aws.amazon.com/privatelink/pricing/). Không ước tính số tiền tại đây.

`JwtSecret` có `DeletionPolicy: Retain` và `UpdateReplacePolicy: Retain`. Cần xét retention khi lập kế hoạch cleanup sau. Thông tin xác thực database do RDS quản lý; trang này không truy xuất hoặc thao tác với hai secret.

{{< notice warning >}}
Interface endpoint Secrets Manager có thể phát sinh chi phí liên tục khi được provisioned. Demo dùng endpoint thay NAT cho các đường truy cập AWS cần thiết. Hướng dẫn chi phí chi tiết và cleanup được trình bày sau.
{{< /notice >}}

<!-- TODO_SCREENSHOT: Loại Secrets Manager VPC Endpoint, private DNS, subnet và SG được gắn. Không hiển thị giá trị secret. -->
