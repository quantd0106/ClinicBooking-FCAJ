---
title: "5.7. Triển khai và kiểm thử trên AWS"
weight: 7
chapter: false
---

# 5.7. TRIỂN KHAI VÀ KIỂM THỬ TRÊN AWS

## Tổng quan

Sau khi hoàn thành ứng dụng và hạ tầng, Backend đã được triển khai lên AWS bằng AWS SAM template do AWS CloudFormation xử lý. Tài liệu triển khai và kiểm thử của Backend ghi nhận triển khai demo thành công tại **ap-southeast-1 ngày 09/09/2026**.

**Client → Amazon API Gateway → AWS Lambda → Amazon RDS for MySQL**

S3 hỗ trợ upload hồ sơ riêng tư, Secrets Manager cung cấp cấu hình runtime được bảo vệ, và CloudWatch nhận log ứng dụng cùng API. Thiết kế hạ tầng được trình bày ở [5.6. Hạ tầng AWS](../6-aws-infrastructure/).

{{< notice info >}}
Các kết quả bên dưới lấy từ bản ghi triển khai trước đây trong repository Backend. Bước viết tài liệu này không triển khai lại ứng dụng, gọi AWS API hoặc chạy lại kiểm thử live. Đây không phải xác nhận trạng thái hoạt động hiện tại của môi trường.
{{< /notice >}}

## Quy trình triển khai

Runbook thực tế và mã nguồn đóng gói/maintenance xác lập quy trình:

1. **Validate infrastructure template.** Review SAM template và chạy kiểm tra cấu trúc hạ tầng của repository.
2. **Build và đóng gói Lambda.** Build ứng dụng NestJS, generate Prisma client và đóng gói handler Node.js 24 cùng native Prisma runtime cần thiết và các migration đã được lưu trong mã nguồn.
3. **Triển khai qua CloudFormation change set đã review.** Quy trình được ghi lại dùng bootstrap template có cùng logical resource names/types và Lambda handler inline tối giản ở bước đầu. Sau khi private bucket thuộc stack được tạo, upload artifact ứng dụng vào deployment prefix của bucket và review change set thứ hai để cập nhật mã ứng dụng.
4. **Provision runtime và tài nguyên hỗ trợ.** CloudFormation xử lý SAM transform để tạo API Gateway, Lambda, RDS riêng tư, S3, networking, IAM, Secrets Manager, endpoint và logging.
5. **Áp dụng migration đã review lên RDS riêng tư.** Bản ghi demo dùng cách gọi trực tiếp Lambda hiện có với maintenance action không phải HTTP `deployReviewedMigrations`. Action áp dụng migration đã lưu trong mã nguồn với MySQL advisory lock và checksum tương thích Prisma. Nhánh này từ chối event có `requestContext` và không được mở qua API Gateway. Bản ghi cho biết cả ba migration đã review đều được áp dụng.
6. **Chạy smoke test live và kiểm tra log.** Kiểm tra ứng dụng qua API Gateway, kiểm thử vòng đời đặt lịch và truy cập file riêng tư, sau đó xem log Lambda/ứng dụng và API access log.

Hai lệnh chuẩn bị sau đã được đối chiếu với `package.json` và runbook triển khai. Lệnh chỉ dùng để tham khảo, **không được thực thi** trong bước viết tài liệu này:

```bash
npm run infra:validate
npm run build:lambda
```

Migration cần môi trường thực thi được kiểm soát với kết nối riêng đến RDS. Bản ghi triển khai giữ RDS riêng tư trong suốt quy trình. Trang này không thực hiện migration, đóng gói, triển khai hoặc seed.

## Kết quả triển khai

| Thành phần | Kết quả trước đây đã được ghi nhận |
|---|---|
| CloudFormation | Bản ghi triển khai ghi nhận `UPDATE_COMPLETE` ngày 09/09/2026. |
| API Gateway → Lambda | API Gateway phục vụ API M1–M5 và tài liệu Swagger qua Lambda runtime. |
| Lambda → RDS | Lambda áp dụng migration đã review và phục vụ các luồng dùng database trên RDS riêng tư. |
| Secrets Manager | Runtime được mô tả resolve cấu hình được bảo vệ qua Secrets Manager interface endpoint. |
| S3 riêng tư | Luồng presigned upload được cấp quyền cho DOCTOR thành công; truy cập object ẩn danh trả HTTP 403. |
| Ranh giới mạng RDS | Bản ghi cho biết kết nối riêng tư; template đặt `PubliclyAccessible: false`. |
| Ranh giới public access S3 | Template bật cả bốn tùy chọn Block Public Access; truy cập ẩn danh được ghi nhận là thất bại. |
| Logging | Log Lambda/ứng dụng và API access log có trong CloudWatch. |

Các kết quả này mô tả lần demo được ghi lại, không phải kiểm tra mới đối với tài nguyên đang chạy.

<!-- TODO_SCREENSHOT: CloudFormation stack triển khai thành công. Che định danh stack/tài nguyên. -->

<!-- TODO_SCREENSHOT: Cấu hình RDS riêng tư. Che endpoint và các định danh. -->

## Kiểm thử API live

Bảng phân biệt **hành vi API mong đợi** và **bằng chứng live đã được ghi nhận**. Mã HTTP mong đợi dựa trên tài liệu API/controller. Luồng thành công được ghi nhận không chứng minh mã HTTP chính xác nếu bản tóm tắt trước đây không lưu mã đó.

| Kiểm thử | Kết quả mong đợi | Kết quả |
|---|---|---|
| PATIENT login: `POST /api/v1/auth/login` | HTTP 200. | Patient login đã pass trong smoke test được ghi lại; bản tóm tắt không lưu mã HTTP chính xác. |
| GET specialties: `GET /api/v1/specialties` | HTTP 200. | Discovery đã pass ở mức luồng tổng hợp; không ghi mã phản hồi riêng. |
| GET doctors: `GET /api/v1/doctors` | HTTP 200. | Discovery đã pass ở mức luồng tổng hợp; không ghi mã phản hồi riêng. |
| GET available slots: `GET /api/v1/doctors/<doctor-id>/available-slots?date=<future-date>` | HTTP 200. | Tra cứu slot đã pass; bản tóm tắt không lưu mã HTTP chính xác. |
| Tạo appointment: `POST /api/v1/appointments` | HTTP 201. | Đặt lịch đã pass; các live check được ghi lại nêu HTTP 201. |
| Đặt lịch trùng | HTTP 409, `SLOT_ALREADY_BOOKED`. | Ghi nhận HTTP 409 với `SLOT_ALREADY_BOOKED`. |
| Danh sách lịch hẹn của caller: `GET /api/v1/appointments/me` | Lịch hẹn mới xuất hiện trong danh sách mà caller được phép xem. | Endpoint được mô tả/triển khai; chưa ghi nhận riêng kết quả live này. |
| Hủy appointment: `PATCH /api/v1/appointments/<appointment-id>/cancel` | HTTP 200, trạng thái `CANCELLED`. | Hủy lịch đã pass; bản tóm tắt không lưu mã phản hồi hoặc response body. |
| Kiểm tra lại/đặt lại slot đã giải phóng | Slot có thể tái sử dụng sau hủy, theo availability và quyền truy cập. | Tái sử dụng slot đã pass; không ghi mã phản hồi riêng. |
| DOCTOR login: `POST /api/v1/auth/login` | HTTP 200. | Endpoint login đã triển khai; chưa ghi nhận riêng kết quả DOCTOR login. |
| Request presigned upload: `POST /api/v1/files/presigned-upload` | Request DOCTOR hợp lệ thành công; HTTP 201 theo contract của POST controller. | Presigned upload của DOCTOR thành công; không ghi mã phản hồi request chính xác. |
| GET object S3 riêng tư trực tiếp, không xác thực | Từ chối truy cập. | Ghi nhận HTTP 403. |

Triển khai upload trả về **presigned POST URL cùng form fields** để upload trực tiếp đến S3 riêng tư. Luồng upload thành công không đồng nghĩa object được phép đọc công khai.

<!-- TODO_SCREENSHOT: Phản hồi đặt lịch live, HTTP 201. Che token và định danh riêng tư. -->

<!-- TODO_SCREENSHOT: Đặt lịch trùng, HTTP 409 `SLOT_ALREADY_BOOKED`. Che định danh riêng tư. -->

## Kiểm tra logging và monitoring

Bằng chứng M5 trước đây ghi nhận cả log Lambda/ứng dụng và API access log trong CloudWatch. Log được kiểm tra về hành vi runtime/ứng dụng; bản ghi cho biết không phát hiện pattern của thông tin xác thực, connection URL database, JWT hoặc chữ ký presigned.

Template cấu hình API access logging và đặt **retention 14 ngày** cho cả hai log group được khai báo. Không để giá trị nhạy cảm xuất hiện trong log và ảnh chụp. Không khẳng định đã triển khai CloudWatch Alarm.

<!-- TODO_SCREENSHOT: CloudWatch Lambda logs đã che giá trị nhạy cảm và định danh riêng tư. -->

## Sự cố trong quá trình triển khai

Điều chỉnh liên quan quota đã được đối chiếu là **reserved concurrency của Lambda**. Tài liệu hạ tầng giải thích quota của tài khoản đã giới hạn concurrency và yêu cầu giữ unreserved pool khiến không thể đặt thêm reservation riêng cho function.

Template xử lý bằng `LambdaReservedConcurrency`: khi giá trị là `0`, condition chọn `AWS::NoValue` và bỏ `ReservedConcurrentExecutions`. Bản ghi triển khai xác nhận demo cuối cùng không đặt reservation riêng cho function và dùng **bộ nhớ 512 MiB**.

Bằng chứng đã kiểm tra không xác lập lỗi memory quota trước đó hoặc chuỗi rollback của CloudFormation. Vì vậy không trình bày các sự cố này như lịch sử đã được xác minh.

## Kết quả cuối cùng

Lần demo được ghi lại đã triển khai Backend thành công và kiểm thử API cốt lõi M1–M5 qua API Gateway và Lambda. Các luồng dùng database truy cập RDS riêng tư; upload S3 riêng tư và việc từ chối truy cập ẩn danh đã được ghi nhận. Bảo vệ đặt lịch trùng trả `409 SLOT_ALREADY_BOOKED`, và hủy lịch/tái sử dụng slot đã pass. Có logging trên CloudWatch.

SAM/CloudFormation template và runbook của repository cung cấp quy trình hạ tầng/triển khai có thể tái lập. Đây vẫn là **môi trường học tập/demo**; các kiểm tra này không xác lập production readiness.

Nguồn bằng chứng trong repository Backend: `docs/09-deployment.md` (trạng thái/runbook triển khai), `docs/08-test-plan.md` (bằng chứng M5 được ghi lại), `docs/06-aws-infrastructure.md` (hạ tầng/concurrency) và `docs/05-api-design.md` (API contract), được đối chiếu với template hiện tại, script đóng gói, controller, Lambda entry và mã nguồn maintenance.

## Lưu ý chi phí và dọn dẹp

Tài nguyên AWS đang chạy có thể phát sinh chi phí. RDS instance được tính phí khi đang chạy, và Secrets Manager interface endpoint được provisioned phát sinh phí endpoint-hour cùng phí xử lý dữ liệu áp dụng. Tham khảo [bảng giá RDS](https://aws.amazon.com/rds/mysql/pricing/) và [AWS PrivateLink](https://aws.amazon.com/privatelink/pricing/).

Dùng [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) để theo dõi chi phí và cảnh báo theo budget phù hợp cho demo; không khẳng định đã có cấu hình budget. Khi không còn cần demo, lập kế hoạch cleanup và rà soát bucket, object, snapshot và secret được giữ lại.

{{< notice warning >}}
Trang này không cung cấp lệnh cleanup mang tính phá hủy và không thực hiện cleanup. Tài nguyên được giữ lại có thể tiếp tục phát sinh chi phí sau khi xóa stack.
{{< /notice >}}
