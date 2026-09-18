---
title: "5.2.2. Thiết kế cơ sở dữ liệu"
weight: 2
chapter: false
---

# 5.2.2. Thiết kế cơ sở dữ liệu

## Cách truy cập dữ liệu

Backend truy cập MySQL thông qua Prisma ORM. Mô hình dữ liệu cốt lõi tách biệt danh tính xác thực, hồ sơ bệnh nhân và bác sĩ, chuyên khoa, lịch làm việc, lịch khám và lịch sử trạng thái lịch khám.

## Các thực thể và quan hệ chính

| Thực thể | Mục đích | Quan hệ chính |
| --- | --- | --- |
| users | Danh tính xác thực với vai trò PATIENT, DOCTOR hoặc ADMIN. | Hồ sơ bệnh nhân và bác sĩ được liên kết với danh tính người dùng tương ứng. |
| patients | Hồ sơ bệnh nhân. | Liên kết với users và các lịch khám của bệnh nhân. |
| doctors | Hồ sơ bác sĩ. | Liên kết với users; có quan hệ với chuyên khoa, lịch làm việc và lịch khám. |
| specialties | Chuyên khoa y tế. | Liên kết với doctors thông qua doctor_specialties. |
| doctor_specialties | Bảng liên kết bác sĩ và chuyên khoa. | Hỗ trợ quan hệ nhiều-nhiều giữa doctors và specialties. |
| doctor_schedules | Lịch làm việc và cấu hình khung giờ của bác sĩ. | Gắn với bác sĩ và được dùng để xác định khung giờ khám còn trống. |
| appointments | Lịch bệnh nhân đặt với bác sĩ, gồm ngày, giờ và trạng thái. | Gắn với bệnh nhân và bác sĩ; có ghi nhận các lần thay đổi trạng thái. |
| appointment_status_history | Lịch sử kiểm tra cho mọi lần chuyển trạng thái lịch khám. | Gắn với lịch khám có trạng thái thay đổi. |

## Sơ đồ quan hệ logic

ERD logic này thể hiện mối liên hệ giữa danh tính người dùng, hồ sơ, chuyên khoa, lịch làm việc, lịch khám và lịch sử trạng thái.

{{< mermaid >}}
graph LR
  Users["users"] -->|patient profile| Patients["patients"]
  Users -->|doctor profile| Doctors["doctors"]
  Doctors -->|working schedule| Schedules["doctor_schedules"]
  Doctors -->|mapping| DoctorSpecialties["doctor_specialties"]
  Specialties["specialties"] -->|mapping| DoctorSpecialties
  Patients -->|patient booking| Appointments["appointments"]
  Doctors -->|doctor booking| Appointments
  Appointments -->|status transitions| History["appointment_status_history"]
{{< /mermaid >}}

## Trạng thái lịch khám

| Trạng thái | Ý nghĩa |
| --- | --- |
| PENDING | Lịch khám đã được tạo và đang chờ xác nhận. Lịch đang hoạt động và chặn khung giờ. |
| CONFIRMED | Lịch khám đã được xác nhận. Lịch vẫn đang hoạt động và chặn khung giờ. |
| COMPLETED | Lịch khám đã xác nhận được hoàn thành. |
| CANCELLED | Lịch khám đã bị hủy và không còn chặn việc đặt lại. |

Vòng đời được phép:

```text
PENDING → CONFIRMED → COMPLETED
PENDING → CANCELLED
CONFIRMED → CANCELLED
```

Mọi lần thay đổi trạng thái lịch khám được ghi vào appointment_status_history.

## Tính nhất quán khi đặt lịch

Việc tính khung giờ trống kết hợp lịch làm việc và cấu hình khung giờ của bác sĩ với các lịch khám đang hoạt động. PENDING và CONFIRMED là các trạng thái đang hoạt động được kiểm tra khi phát hiện xung đột cùng bác sĩ, ngày và giờ.

Đặt lịch sử dụng MySQL transaction và cơ chế khóa. Yêu cầu đặt trùng lịch đang hoạt động trả về HTTP 409 với mã SLOT_ALREADY_BOOKED. Lịch CANCELLED được loại khỏi kiểm tra lịch đang hoạt động, vì vậy không chặn việc đặt lại cùng khung giờ.
