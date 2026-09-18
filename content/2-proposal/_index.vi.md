---
title: "Bản đề xuất"
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# BẢN ĐỀ XUẤT DỰ ÁN

Bản đề xuất trình bày Clinic Appointment Booking System, hệ thống Backend hỗ trợ tìm bác sĩ, xem lịch trống và quản lý lịch khám. Nội dung gồm mục tiêu nghiệp vụ, thiết kế kỹ thuật, các mốc triển khai, chi phí tham khảo cho môi trường demo và tiêu chí thành công khi triển khai trên AWS.

## 2.1. Tổng quan dự án

Clinic Appointment Booking System là hệ thống Backend phục vụ quy trình đặt lịch khám trực tuyến. Hệ thống giúp bệnh nhân tìm bác sĩ, xem các khung giờ còn trống, đặt lịch, hủy lịch và quản lý lịch khám của mình.

Bác sĩ có thể quản lý lịch làm việc và cập nhật trạng thái lịch khám. Quản trị viên hỗ trợ quản lý dữ liệu hệ thống theo quyền được cấp.

Dự án là bài toán thực hành kết hợp phát triển Backend, thiết kế cơ sở dữ liệu và triển khai trên AWS Cloud trong một hệ thống hoàn chỉnh. Backend sử dụng Node.js, TypeScript, NestJS, Prisma ORM và MySQL, cùng JWT authentication, phân quyền theo vai trò (RBAC) và tài liệu Swagger/OpenAPI.

## 2.2. Bài toán và mục tiêu

### Bài toán

- Quy trình đặt lịch thủ công có thể gây khó khăn khi kiểm tra lịch trống của bác sĩ.
- Các yêu cầu đồng thời có thể đặt cùng bác sĩ, ngày và giờ nếu Backend không xử lý concurrency.
- PATIENT, DOCTOR và ADMIN cần được phân quyền rõ ràng.
- Cơ sở dữ liệu và nơi lưu trữ tệp cần được triển khai an toàn trên Cloud.

### Mục tiêu chức năng

- Cung cấp chức năng xác thực.
- Hỗ trợ tìm bác sĩ và quản lý lịch làm việc.
- Tính toán các khung giờ khám còn trống.
- Hỗ trợ đặt lịch và hủy lịch khám.
- Quản lý vòng đời trạng thái lịch khám và ghi nhận lịch sử thay đổi trạng thái.
- Hỗ trợ tải lên tệp riêng tư.

### Mục tiêu kỹ thuật

- Sử dụng JWT authentication và RBAC để bảo vệ quyền truy cập ứng dụng.
- Ngăn đặt lịch trùng bằng MySQL transaction, cơ chế khóa và kiểm tra lịch khám đang hoạt động.
- Triển khai cơ sở dữ liệu ở chế độ riêng tư.
- Triển khai Backend trên AWS theo kiến trúc Serverless.
- Tập trung log thực thi.
- Định nghĩa hạ tầng bằng Infrastructure as Code.

### Tính nhất quán khi đặt lịch và lịch sử trạng thái

Lịch khám có trạng thái PENDING hoặc CONFIRMED được xem là đang hoạt động và chặn yêu cầu đặt thêm cùng bác sĩ, ngày và giờ. Lịch đã hủy không chặn việc đặt lại. Yêu cầu đặt trùng lịch đang hoạt động phải trả về HTTP 409 với mã lỗi SLOT_ALREADY_BOOKED.

Vòng đời lịch khám hỗ trợ:

- PENDING → CONFIRMED → COMPLETED
- PENDING → CANCELLED
- CONFIRMED → CANCELLED

Mỗi lần thay đổi trạng thái lịch khám đều được ghi vào appointment_status_history.

## 2.3. Đối tượng sử dụng và chức năng chính

| Vai trò | Chức năng chính |
| --- | --- |
| PATIENT | Đăng ký và đăng nhập; xem chuyên khoa; tìm kiếm và xem thông tin bác sĩ; xem khung giờ trống; tạo lịch khám; xem lịch khám của mình; hủy lịch khám. |
| DOCTOR | Đăng nhập; quản lý lịch làm việc; xem các lịch khám liên quan; cập nhật trạng thái lịch khám; yêu cầu presigned URL để tải lên hồ sơ hoặc tệp. |
| ADMIN | Quản lý chuyên khoa; kích hoạt hoặc vô hiệu hóa bác sĩ; thực hiện các thao tác quản trị theo quyền của hệ thống. |

Đăng ký công khai chỉ tạo tài khoản PATIENT, không cấp quyền DOCTOR hoặc ADMIN.

JWT authentication xác định người dùng, còn RBAC xác định các chức năng người dùng được phép truy cập. Quyền truy cập lịch khám cá nhân hoặc lịch khám liên quan phải tuân theo vai trò và quyền của người dùng.

## 2.4. Kiến trúc giải pháp

### Luồng xử lý yêu cầu

Backend chạy ứng dụng NestJS trên AWS Lambda với Node.js runtime. Amazon API Gateway tiếp nhận yêu cầu từ Client. NestJS xử lý xác thực, phân quyền và nghiệp vụ; Prisma ORM truy cập Amazon RDS for MySQL.

```text
Client
  → Amazon API Gateway
  → AWS Lambda (NestJS / Node.js)
  → Prisma ORM
  → Amazon RDS for MySQL

AWS Lambda
  ├─ Amazon S3: đối tượng riêng tư và tạo presigned URL
  ├─ AWS Secrets Manager: cấu hình runtime nhạy cảm
  └─ Amazon CloudWatch: log thực thi và giám sát

Client
  → Amazon S3: tải lên tệp riêng tư bằng presigned URL
```

Prisma ORM là một phần của ứng dụng Lambda, không phải một dịch vụ AWS riêng biệt. Nghiệp vụ đặt lịch sử dụng MySQL transaction và cơ chế khóa để ngăn các yêu cầu đồng thời tạo lịch đang hoạt động cho cùng bác sĩ, ngày và giờ.

### Cô lập mạng và truy cập dịch vụ riêng tư

- Lambda chạy trong Amazon VPC.
- RDS sử dụng private subnets và không được truy cập công khai.
- Security Group của cơ sở dữ liệu chỉ cho phép cổng MySQL 3306 từ Lambda Security Group.
- Amazon S3 được giữ riêng tư và bật Block Public Access. Tải lên tệp sử dụng presigned URL.
- Lambda truy cập Amazon S3 qua S3 Gateway VPC Endpoint.
- Truy cập AWS Secrets Manager sử dụng Secrets Manager Interface VPC Endpoint.
- Kiến trúc cuối cùng không sử dụng NAT Gateway.

AWS IAM kiểm soát quyền của Lambda và hoạt động triển khai. AWS Secrets Manager lưu cấu hình runtime nhạy cảm, gồm thông tin đăng nhập cơ sở dữ liệu và cấu hình bí mật liên quan đến JWT. Amazon CloudWatch tiếp nhận log thực thi và hỗ trợ giám sát.

### Cấu hình triển khai

| Thành phần | Thiết kế |
| --- | --- |
| Region | ap-southeast-1 — Asia Pacific (Singapore) |
| Cơ sở dữ liệu | Amazon RDS for MySQL; db.t4g.micro; Single-AZ; dung lượng 20 GiB; riêng tư và không công khai |
| Lambda | Node.js runtime; bộ nhớ 512 MiB; không khai báo ReservedConcurrentExecutions trong bản triển khai cuối cùng |
| Hạ tầng | AWS SAM định nghĩa, build và triển khai hạ tầng Serverless thông qua AWS CloudFormation. |

## 2.5. Các dịch vụ AWS sử dụng

| Dịch vụ | Vai trò trong hệ thống | Lý do lựa chọn |
| --- | --- | --- |
| Amazon API Gateway | Cung cấp API của Backend và chuyển yêu cầu từ Client đến Lambda. | Tạo điểm tiếp nhận API được quản lý cho Backend Serverless. |
| AWS Lambda | Chạy Backend NestJS trên Node.js runtime. | Thực thi nghiệp vụ Backend mà không phải quản lý máy chủ EC2. |
| Amazon RDS for MySQL | Lưu người dùng, bác sĩ, lịch làm việc, lịch khám và lịch sử trạng thái. | Cung cấp cơ sở dữ liệu quan hệ được quản lý, phù hợp với giao dịch đặt lịch và cơ chế khóa. |
| Amazon S3 | Lưu tệp riêng tư và hồ sơ tải lên bằng presigned URL. | Cung cấp object storage phù hợp với tệp tải lên, đồng thời giữ bucket riêng tư. |
| Amazon VPC | Cô lập Lambda và cơ sở dữ liệu riêng tư; hỗ trợ các VPC Endpoints trong thiết kế. | Kiểm soát truy cập mạng đến tài nguyên riêng tư mà không cần NAT Gateway trong thiết kế cuối cùng. |
| AWS IAM | Kiểm soát quyền của Lambda và hoạt động triển khai. | Áp dụng quyền truy cập tối thiểu cần thiết đến tài nguyên AWS. |
| AWS Secrets Manager | Lưu thông tin đăng nhập cơ sở dữ liệu và cấu hình bí mật liên quan đến JWT. | Tách cấu hình runtime nhạy cảm khỏi mã nguồn ứng dụng. |
| Amazon CloudWatch | Thu thập log thực thi và hỗ trợ giám sát. | Tập trung log ứng dụng và thông tin phục vụ theo dõi vận hành. |
| AWS CloudFormation | Khởi tạo hạ tầng từ template. | Triển khai Infrastructure as Code có thể lặp lại và hỗ trợ rollback stack. |
| AWS SAM | Định nghĩa, build và triển khai ứng dụng Serverless. | Hỗ trợ quy trình phát triển Serverless với hạ tầng dựa trên CloudFormation. |

## 2.6. Kế hoạch triển khai

| Mốc | Phạm vi | Sản phẩm dự kiến |
| --- | --- | --- |
| M1 — Foundation / Auth / API Contract | Thiết lập nền tảng Node.js, TypeScript và NestJS; triển khai JWT authentication và RBAC; xác định API contract. | Nền tảng Backend, luồng xác thực và phân quyền, cùng API contract trên Swagger/OpenAPI. |
| M2 — Core Data Model + MySQL | Thiết kế mô hình dữ liệu cốt lõi bằng Prisma ORM và MySQL, gồm lịch sử trạng thái lịch khám. | Schema cơ sở dữ liệu và nền tảng truy cập dữ liệu. |
| M3 — Doctor Discovery + Schedule | Triển khai tra cứu chuyên khoa và bác sĩ, lịch làm việc và tính toán khung giờ trống. | API tìm bác sĩ và kiểm tra lịch trống. |
| M4 — Appointment Booking Core | Triển khai đặt lịch, hủy lịch, vòng đời trạng thái và ghi lịch sử; áp dụng transaction và cơ chế khóa. | Nghiệp vụ đặt lịch có xử lý xung đột lịch đang hoạt động và cho phép đặt lại sau khi hủy. |
| M5 — AWS Runtime + Infrastructure | Triển khai Lambda runtime, API Gateway, RDS riêng tư, S3 riêng tư, VPC Endpoints, quyền IAM, secrets và logging bằng SAM/CloudFormation. | Hạ tầng Serverless và cấu hình runtime trên AWS. |
| M6 — QA + Documentation + Demo | Thực hiện QA cuối cùng, hoàn thiện tài liệu và chuẩn bị demo. | Kết quả QA, tài liệu dự án và nội dung chuẩn bị demo. |

M6 tập trung vào QA cuối cùng, tài liệu và chuẩn bị demo; đây không phải điều kiện bắt buộc để hệ thống kỹ thuật hoạt động. Các mốc trên mô tả kế hoạch triển khai, không phải báo cáo kiểm chứng trực tiếp đã hoàn thành.

## 2.7. Chi phí dự kiến

### Giả định chi phí cho môi trường demo

Dự án phục vụ học tập và demo, không hướng tới vận hành production liên tục. Chi phí phụ thuộc vào lưu lượng, thời gian duy trì tài nguyên, dung lượng lưu trữ và mức sử dụng.

Ước tính tham khảo cho môi trường demo chạy liên tục: khoảng USD 35–40/tháng. Đây là khoảng chi phí sơ bộ để lập kế hoạch cho thiết kế được cung cấp tại ap-southeast-1, không phải hóa đơn AWS cố định được bảo đảm hoặc phép tính giá đã kiểm chứng. Chi phí thực tế có thể thay đổi.

| Thành phần chi phí | Yếu tố cần lưu ý |
| --- | --- |
| Amazon RDS for MySQL | Instance db.t4g.micro Single-AZ và dung lượng 20 GiB là nguồn chi phí duy trì chính khi tài nguyên còn được khởi tạo. |
| Secrets Manager Interface VPC Endpoint | Số giờ duy trì endpoint và dữ liệu xử lý tạo chi phí thường xuyên; chi phí còn phụ thuộc số Availability Zones dùng cho endpoint. |
| AWS Lambda và Amazon API Gateway | Với lưu lượng demo, chi phí yêu cầu và thực thi dự kiến tương đối thấp. |
| Amazon S3 | Lưu trữ tệp tải lên và các yêu cầu tạo chi phí theo mức sử dụng. |
| Amazon CloudWatch | Thu nhận log, thời gian lưu log và mức sử dụng giám sát tạo chi phí. |
| AWS Secrets Manager | Số secrets được lưu và các yêu cầu API tạo chi phí. |

RDS và Secrets Manager Interface VPC Endpoint là các tài nguyên có chi phí duy trì liên tục chính trong thiết kế này. Các chi phí sử dụng và lưu trữ khác trong bảng vẫn cần được theo dõi.

### Kiểm soát chi phí

- Cấu hình AWS Budget để theo dõi chi phí.
- Dọn dẹp tài nguyên demo sau buổi trình diễn khi không còn nhu cầu sử dụng.
- Theo dõi mức bộ nhớ Lambda 512 MiB đã thiết kế dựa trên tải thực tế.
- Tránh NAT Gateway không cần thiết; kiến trúc cuối cùng sử dụng các VPC Endpoints đã thiết kế, một phần nhằm giảm chi phí duy trì.

## 2.8. Rủi ro và biện pháp xử lý

| Rủi ro | Ảnh hưởng | Biện pháp |
| --- | --- | --- |
| Đặt lịch trùng | Hai lịch đang hoạt động có thể xung đột cho cùng bác sĩ, ngày và giờ. | Sử dụng MySQL transaction, cơ chế khóa và kiểm tra lịch PENDING/CONFIRMED; trả về HTTP 409 SLOT_ALREADY_BOOKED khi đặt trùng lịch đang hoạt động. |
| Truy cập trái phép | Người dùng có thể truy cập chức năng hoặc dữ liệu ngoài phạm vi quyền. | Áp dụng JWT authentication, RBAC và quyền AWS IAM tối thiểu cần thiết. |
| Cơ sở dữ liệu bị công khai | Dữ liệu lịch khám và người dùng có thể bị lộ qua truy cập mạng công khai. | Giữ RDS riêng tư và chỉ cho phép cổng MySQL 3306 từ Lambda Security Group. |
| Rò rỉ bí mật | Thông tin đăng nhập cơ sở dữ liệu hoặc bí mật liên quan đến JWT có thể bị lộ. | Lưu cấu hình nhạy cảm trong AWS Secrets Manager và không hard-code thông tin đăng nhập. |
| Chi phí tài nguyên Cloud | Tài nguyên còn chạy có thể tạo chi phí không cần thiết. | Cấu hình AWS Budget, dọn dẹp sau demo và tránh NAT Gateway không cần thiết. |
| Giới hạn quota tài khoản AWS | Quota hiện có có thể giới hạn triển khai hoặc hoạt động runtime. | Kiểm tra quota và chọn bộ nhớ, cấu hình concurrency phù hợp cho Lambda; bản triển khai cuối cùng dùng bộ nhớ 512 MiB và không khai báo ReservedConcurrentExecutions. |
| Triển khai thất bại | Triển khai chưa hoàn tất có thể khiến môi trường không khả dụng hoặc thiếu nhất quán. | Sử dụng rollback của AWS CloudFormation, kiểm tra log và xác thực cấu hình trước khi triển khai lại. |

## 2.9. Kết quả mong đợi và tiêu chí thành công

### Kết quả mong đợi

Kết quả mong đợi là Backend Serverless của Clinic Appointment Booking System hoạt động và được triển khai trên AWS.

### Tiêu chí thành công của đề xuất

- Xác thực người dùng hoạt động và quyền truy cập tuân theo vai trò được cấp.
- Bệnh nhân có thể tìm bác sĩ và xem các khung giờ khám còn trống.
- Đặt lịch thành công với khung giờ còn trống.
- Yêu cầu đặt trùng lịch đang hoạt động trả về HTTP 409 với mã SLOT_ALREADY_BOOKED.
- Lịch đã hủy giải phóng khung giờ để có thể đặt lại.
- Vòng đời trạng thái hỗ trợ PENDING → CONFIRMED → COMPLETED, PENDING → CANCELLED và CONFIRMED → CANCELLED.
- Mỗi lần thay đổi trạng thái lịch khám được ghi vào appointment_status_history.
- Amazon RDS for MySQL không được truy cập công khai.
- Bucket Amazon S3 được giữ riêng tư.
- Lambda truy cập được các tài nguyên riêng tư cần thiết qua cấu hình mạng đã thiết kế.
- Amazon CloudWatch tiếp nhận log ứng dụng.
- Hạ tầng có thể được triển khai bằng AWS SAM và AWS CloudFormation.

Đây là các tiêu chí nghiệm thu được đề xuất, không phải báo cáo kiểm thử trực tiếp. Việc kiểm chứng cuối cùng và bằng chứng chi tiết sẽ được trình bày sau trong phần Workshop.
