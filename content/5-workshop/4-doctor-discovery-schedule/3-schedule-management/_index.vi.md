---
title: "5.4.3. Quản lý lịch làm việc bác sĩ"
weight: 3
chapter: false
---

# 5.4.3. Quản lý lịch làm việc bác sĩ

## 1. Xác định endpoint và ownership

Lịch làm việc xác định khi nào bác sĩ làm việc và cách chia khoảng thời gian đó thành các khung giờ khám. Triển khai nằm trong `src/schedules/schedules.controller.ts`, `schedules.service.ts`, `dto/schedule.dto.ts` và `src/database/schedule-persistence.service.ts`.

| Method | Endpoint | Thành công | Quyền truy cập |
| --- | --- | --- | --- |
| POST | `/api/v1/doctors/:id/schedules` | 201; response lịch làm việc. | Bearer JWT, DOCTOR, hồ sơ bác sĩ của chính mình. |
| PATCH | `/api/v1/schedules/:id` | 200; response lịch làm việc. | Bearer JWT, DOCTOR, lịch của chính mình. |
| DELETE | `/api/v1/schedules/:id` | 204; không có body. | Bearer JWT, DOCTOR, lịch của chính mình. |

Cả ba handler khai báo `@Roles(Role.DOCTOR)`. Không cấp quyền cho ADMIN. Service so sánh user ID của actor với `userId` của bác sĩ sở hữu lịch bằng `OwnershipService.assertOwner()`, không bật tùy chọn ADMIN override.

Thiếu/sai xác thực trả về 401. Vai trò không được phép trả về 403 `FORBIDDEN`; bác sĩ quản lý lịch của bác sĩ khác nhận 403 `OWNERSHIP_REQUIRED`.

## 2. Chuẩn bị request tạo lịch hợp lệ

Các trường `CreateScheduleDto` đã kiểm chứng:

| Trường | Validation và ý nghĩa |
| --- | --- |
| `workDate` | Ngày bắt buộc theo `YYYY-MM-DD`; time service còn kiểm tra ngày có thật trên lịch. |
| `startTime` | Giờ bắt đầu bắt buộc theo `HH:mm` 24 giờ. |
| `endTime` | Giờ kết thúc bắt buộc theo `HH:mm` 24 giờ; phải sau giờ bắt đầu. |
| `slotDurationMinutes` | Số nguyên bắt buộc, 1–1440 tại biên API; phải nằm trong độ dài khoảng làm việc. |
| `status` | Tùy chọn `OPEN` hoặc `CLOSED`; persistence mặc định OPEN. |

Dùng UUID bác sĩ từ dữ liệu local và token riêng của bác sĩ đó:

```http
POST /api/v1/doctors/<doctor-id>/schedules
Authorization: Bearer <token>
```

```json
{
  "workDate": "2030-01-15",
  "startTime": "09:00",
  "endTime": "12:00",
  "slotDurationMinutes": 30,
  "status": "OPEN"
}
```

Chọn ngày tương lai khi thực hiện ví dụ. Service yêu cầu thời điểm bắt đầu, diễn giải theo múi giờ phòng khám, phải sau thời điểm hiện tại.

Response lịch làm việc gồm `id`, `doctorId`, `workDate`, `startTime`, `endTime`, `slotDurationMinutes` và `status`. Chuỗi ngày/giờ request được chuyển qua `ClinicTimeService` sang các trường date/time MySQL trong schema; response định dạng lại ngày và giờ thành chuỗi.

## 3. Ngăn khoảng làm việc chồng lấn

Với cùng bác sĩ và ngày, lịch hiện có 09:00–12:00 chồng lấn lịch mới 10:00–13:00. Từ chối trường hợp này giúp tránh hai khoảng làm việc sinh ra các định nghĩa khung giờ xung đột.

`SchedulePersistenceService.createSchedule()` chuẩn hóa đầu vào, mở Prisma transaction, khóa dòng bác sĩ bằng `SELECT ... FOR UPDATE`, kiểm tra chồng lấn bằng locking/current read và chỉ tạo lịch khi không có xung đột.

Đoạn SQL nguyên bản sau trích từ `assertNoOverlap()`, đã lược bỏ query bao quanh và phần loại trừ lịch đang cập nhật:

```sql
WHERE doctor_id = ${input.doctorId}
  AND work_date = DATE(${input.workDate})
  AND start_time < TIME(${input.endTime})
  AND end_time > TIME(${input.startTime})
```

Đây là các biểu thức trong SQL template của Prisma, không phải giá trị để dán vào SQL console. Hai phép so sánh đều nghiêm ngặt, nên các khoảng liền kề có thể tiếp giáp ở ranh giới. Query chồng lấn kiểm tra mọi trạng thái lịch, kể cả CLOSED.

Locking/current read tránh bỏ sót lịch vừa được transaction khác commit do snapshot transaction cũ. Update cũng khóa dòng bác sĩ và lịch, đồng thời loại trừ chính lịch đang cập nhật khỏi kiểm tra chồng lấn.

Xung đột trả về:

```json
{
  "statusCode": 409,
  "code": "SCHEDULE_OVERLAP",
  "message": "The doctor already has an overlapping schedule on this date."
}
```

Ví dụ lịch sử ngày 10/09/2026 đã cung cấp—09:00–12:00, khung 30 phút trả về 201, sau đó request chồng lấn 10:00–13:00 trả về 409—phù hợp quy tắc này. Đây không phải lần chạy test mới. Khi tạo lại sau ngày đó, cần dùng ngày tương lai để đáp ứng validation thời điểm bắt đầu.

## 4. Cập nhật hoặc xóa mà không ảnh hưởng lịch khám

`UpdateScheduleDto` là phiên bản partial của DTO tạo lịch. Gửi ít nhất một trường. Service gộp các trường gửi lên với lịch hiện có và kiểm tra thời điểm bắt đầu sau khi gộp vẫn thuộc tương lai.

Trước update hoặc delete, persistence kiểm tra lịch khám đang hoạt động. Nếu có, trả về 409 `SCHEDULE_HAS_ACTIVE_BOOKINGS`. Kiểm tra update áp dụng cho mọi cập nhật lịch, không chỉ thay đổi thời gian.

Delete chạy trong transaction:

- Có lịch khám đang hoạt động: từ chối với 409.
- Còn lịch sử lịch khám nhưng không còn lịch khám đang hoạt động: đặt lịch làm việc thành CLOSED.
- Không có lịch khám tham chiếu: xóa vật lý lịch làm việc.

UUID sai định dạng, khoảng thời gian/độ dài khung không hợp lệ, PATCH body rỗng hoặc thời điểm bắt đầu không thuộc tương lai trả về 400. Bác sĩ hoặc lịch làm việc không tồn tại trả về 404 `NOT_FOUND`.

{{< notice info >}}
Ngăn chồng lấn lịch làm việc bảo đảm các khoảng làm việc của cùng bác sĩ không giao nhau. Ngăn đặt trùng lịch khám bảo đảm hai lịch khám đang hoạt động không chiếm cùng bác sĩ/ngày/giờ bắt đầu. Đây là hai quy tắc riêng; bảo vệ đồng thời khi đặt lịch được trình bày ở mục 5.5.
{{< /notice >}}

## 5. Kiểm tra trong Swagger local

Với tài khoản DOCTOR riêng, tạo lịch tương lai, thử khoảng thời gian chồng lấn và kiểm tra ownership khi nhắm đến lịch của bác sĩ khác. Các test M3 liên quan kiểm tra giới hạn vai trò, ownership, chồng lấn và bảo vệ lịch làm việc có lịch khám đang hoạt động.

<!-- TODO_SCREENSHOT: tạo lịch thành công và response 409 SCHEDULE_OVERLAP, đã che token và định danh cá nhân. -->

