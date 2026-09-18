---
title: "5.2.3. Các luồng chính của hệ thống"
weight: 3
chapter: false
---

# 5.2.3. Các luồng chính của hệ thống

## 1. Xác thực

```text
Client
→ POST /api/v1/auth/login
→ Amazon API Gateway
→ AWS Lambda
→ NestJS Auth
→ Trả JWT về Client
```

Luồng xác thực xác định người dùng và trả về JWT. Các yêu cầu được bảo vệ sử dụng JWT authentication và RBAC để thực thi quyền PATIENT, DOCTOR và ADMIN.

Đăng ký công khai chỉ tạo tài khoản PATIENT. Client công khai không được chọn vai trò hoặc tự đăng ký thành DOCTOR hay ADMIN.

## 2. Tìm bác sĩ và xem khung giờ trống

```text
Patient
→ GET specialties / doctors
→ Chọn bác sĩ
→ GET /doctors/{id}/available-slots?date=YYYY-MM-DD
→ Backend kết hợp lịch làm việc và các lịch khám đang hoạt động
→ Trả về các khung giờ còn trống
```

Lịch làm việc và cấu hình khung giờ của bác sĩ xác định các khung giờ khám có thể đặt. Các lịch PENDING và CONFIRMED đã tồn tại loại khung giờ bị chiếm khỏi kết quả còn trống. Lịch CANCELLED không chặn việc đặt lại.

Kết quả lịch trống không phải một lần giữ chỗ: yêu cầu khác có thể đặt khung giờ trước khi bệnh nhân gửi yêu cầu đặt lịch. Vì vậy, transaction đặt lịch phải kiểm tra lại khả năng đặt.

## 3. Đặt lịch khám

```text
Patient
→ POST /appointments
→ Kiểm tra danh tính và vai trò
→ Kiểm tra lịch làm việc
→ Mở MySQL transaction
→ Khóa lịch làm việc liên quan
→ Kiểm tra lịch đang hoạt động cho cùng bác sĩ/ngày/giờ
→ Tạo lịch PENDING
→ Tạo lịch sử trạng thái
→ Commit
```

PENDING và CONFIRMED là các trạng thái lịch khám đang hoạt động. Transaction và cơ chế khóa bảo vệ trước các yêu cầu đặt lịch đồng thời cho cùng bác sĩ, ngày và giờ.

Nếu đã có lịch đang hoạt động chiếm khung giờ:

```text
HTTP 409
SLOT_ALREADY_BOOKED
```

Backend từ chối yêu cầu đặt trùng thay vì tạo thêm một lịch đang hoạt động cho khung giờ đã bị chiếm.

## 4. Hủy lịch và giải phóng khung giờ

```text
Patient hủy lịch khám
→ Trạng thái trở thành CANCELLED
→ Ghi lịch sử trạng thái
→ Lịch đã hủy không còn chặn khung giờ
→ Khung giờ có thể được đặt lại
```

Hủy lịch tuân theo quyền truy cập lịch khám của bệnh nhân. Lịch PENDING và CONFIRMED có thể chuyển sang CANCELLED. Lịch đã hủy vẫn thuộc vòng đời lịch khám và lịch sử trạng thái, trong khi kiểm tra lịch đang hoạt động cho phép đặt lại khung giờ.

## 5. Bác sĩ chuyển trạng thái lịch khám

```text
Doctor cập nhật lịch khám liên quan
→ PENDING → CONFIRMED → COMPLETED
→ Mỗi lần thay đổi được ghi vào appointment_status_history
```

Bác sĩ xem các lịch khám liên quan và cập nhật trạng thái trong phạm vi quyền của hệ thống. Luồng này tuân theo vòng đời được phép, không bổ sung chuyển trạng thái hoặc hành vi quản trị chưa được xác nhận.
