---
title: "5.6.1. Thiết kế VPC và mạng"
weight: 1
chapter: false
---

# Thiết kế VPC và mạng

## Tài nguyên và vị trí subnet

`ClinicVpc` bật DNS support và DNS hostnames. CIDR của VPC lấy từ parameter trong template. Template chọn Availability Zone có chỉ số **0 và 1** bằng `Fn::GetAZs` và `Fn::Select`; tên AZ phụ thuộc region của stack.

| Subnet | Lựa chọn AZ | Route table | Vị trí workload |
|---|---|---|---|
| `PublicSubnetA` | Chỉ số 0 | `PublicRouteTable` | Không khai báo đặt Lambda hoặc RDS tại đây. |
| `PublicSubnetB` | Chỉ số 1 | `PublicRouteTable` | Không khai báo đặt Lambda hoặc RDS tại đây. |
| `PrivateSubnetA` | Chỉ số 0 | `PrivateRouteTable` | Lambda, DB subnet group của RDS, ENI của endpoint Secrets Manager. |
| `PrivateSubnetB` | Chỉ số 1 | `PrivateRouteTable` | Lambda, DB subnet group của RDS, ENI của endpoint Secrets Manager. |

Cả bốn subnet đều đặt `MapPublicIpOnLaunch: false`. Lambda và RDS **dùng chung hai private subnet**; template không tạo hai cặp subnet riêng cho ứng dụng và database.

`InternetGatewayAttachment` gắn `InternetGateway` vào VPC. `PublicDefaultRoute` định tuyến `0.0.0.0/0` qua gateway này. `PublicSubnetARouteTableAssociation` và `PublicSubnetBRouteTableAssociation` liên kết public subnet với `PublicRouteTable`. Hai resource `PrivateSubnetARouteTableAssociation` và `PrivateSubnetBRouteTableAssociation` liên kết private subnet với `PrivateRouteTable`.

Private route table **không có default route ra Internet và không có NAT Gateway**. Đường truy cập dịch vụ S3 được cung cấp bởi `S3GatewayEndpoint`. `SecretsManagerEndpoint` tạo network interface riêng tư tại cả hai private subnet và bật private DNS.

## Luồng truy cập ứng dụng

{{< mermaid >}}
graph TD
  Client["Internet / Client"] --> API["API Gateway HTTP API"]
  API --> Lambda
  subgraph VPC
    Lambda["Lambda - PrivateSubnetA/B"]
    RDS["RDS MySQL - private DB subnet group"]
    S3EP["S3 Gateway Endpoint - PrivateRouteTable"]
    SecretsEP["Secrets Manager Interface Endpoint - private ENIs"]
    Lambda -->|TCP 3306| RDS
    Lambda -->|HTTPS| S3EP
    Lambda -->|HTTPS 443| SecretsEP
  end
  S3EP --> S3["Private S3 bucket"]
  SecretsEP --> Secrets["Secrets Manager"]
{{< /mermaid >}}

API Gateway gọi Lambda qua tích hợp dịch vụ AWS. Luồng này không yêu cầu inbound HTTP rule trên Security Group VPC của Lambda. Kết nối VPC cho phép mã ứng dụng trong Lambda truy cập MySQL và các service endpoint theo thiết kế bằng mạng riêng.

Cấu hình VPC thực tế của `ClinicApiFunction`:

```yaml
VpcConfig:
  SecurityGroupIds:
    - Ref: LambdaSecurityGroup
  SubnetIds:
    - Ref: PrivateSubnetA
    - Ref: PrivateSubnetB
```

`DatabaseSubnetGroup` cũng tham chiếu `PrivateSubnetA` và `PrivateSubnetB`. DB subnet group trải trên hai AZ không đồng nghĩa database chạy Multi-AZ: DB instance được cấu hình rõ ràng là Single-AZ.

Function dùng `nodejs24.x`, bộ nhớ 512 MiB, timeout 30 giây và `x86_64`. Thuộc tính `ReservedConcurrentExecutions` có điều kiện: parameter `LambdaReservedConcurrency` mặc định bằng 0, khi đó chọn `AWS::NoValue` và bỏ thuộc tính này. Parameter dương bật giới hạn tường minh. Đây là hành vi của template, không phải xác minh cấu hình concurrency đã triển khai.

## Lý do lựa chọn

RDS đặt trong private subnet để dữ liệu ứng dụng được truy cập qua kết nối nội bộ có kiểm soát thay vì endpoint database công khai. Lambda dùng cùng VPC để kết nối RDS qua TCP 3306. S3 và Secrets Manager là các đường truy cập dịch vụ AWS trong runtime được hai endpoint hỗ trợ.

Bỏ NAT giúp không phải bổ sung NAT Gateway vào thiết kế demo. Đổi lại, runtime bình thường không được phụ thuộc vào các dịch vụ Internet bất kỳ ở bên ngoài. Rule Security Group cho phép HTTPS không tạo ra route Internet còn thiếu. Nếu bổ sung dependency bên ngoài, cần quyết định thiết kế mạng riêng.

Log thực thi Lambda được chuyển qua dịch vụ logging của Lambda; template không khai báo thêm CloudWatch Logs VPC Endpoint.

{{< notice info >}}
Template không có NAT Gateway. Internet Gateway và public route không cung cấp Internet tổng quát cho private subnet của Lambda.
{{< /notice >}}

📷 Ảnh cần bổ sung: Các subnet VPC, liên kết route table và tổng quan endpoint, đã che các định danh.
