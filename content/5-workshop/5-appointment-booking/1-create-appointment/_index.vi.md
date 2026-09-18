---
title: "5.5.1. Tạo lịch khám"
weight: 1
chapter: false
---

# 5.5.1. Tạo lịch khám

## 1. Xác định endpoint và danh tính bệnh nhân

`POST /api/v1/appointments` yêu cầu Bearer JWT và vai trò PATIENT. Controller dùng `@Roles(Role.PATIENT)`, đồng thời `AppointmentsService.create()` kiểm tra lại vai trò.

DTO request đã kiểm chứng có đúng bốn trường bắt buộc:

| Trường | Validation tại API |
| --- | --- |
| `doctorId` | UUID bác sĩ. |
| `scheduleId` | UUID lịch làm việc. |
| `appointmentDate` | Ngày tại phòng khám theo `YYYY-MM-DD`; time service còn kiểm tra ngày có thật. |
| `startTime` | Giờ phòng khám theo `HH:mm` 24 giờ. |

Không có trường `patientId`, `patientUserId`, `endTime` hoặc `status` do client gửi. Validation Pipe toàn cục từ chối trường ngoài DTO. Server lấy danh tính bệnh nhân từ actor đã xác thực, tính giờ kết thúc từ độ dài khung của lịch làm việc và tạo trạng thái PENDING.

Đoạn gọi nguyên bản sau trích từ `src/appointments/appointments.service.ts`, đã lược bỏ kiểm tra vai trò, xử lý lỗi bao quanh và ánh xạ response:

```typescript
const appointment = await this.persistence.bookAppointment({
  patientUserId: actor.id,
  doctorId: dto.doctorId,
  scheduleId: dto.scheduleId,
  appointmentDate: dto.appointmentDate,
  startTime: dto.startTime,
  slotInstant: this.clinicTime.toUtcInstant(dto.appointmentDate, dto.startTime),
});
```

## 2. Chuẩn bị request an toàn

Lấy ID bác sĩ từ tra cứu và ID lịch làm việc từ response tạo lịch đã được phân quyền. Chọn giờ bắt đầu đúng ranh giới khung, thuộc tương lai trong lịch OPEN đó.

```http
POST /api/v1/appointments
Authorization: Bearer <patient-token>
```

```json
{
  "doctorId": "<doctor-id>",
  "scheduleId": "<schedule-id>",
  "appointmentDate": "2036-01-15",
  "startTime": "09:00"
}
```

Thay định danh bằng UUID hợp lệ trong lần kiểm tra local riêng. Dùng ngày tương lai khớp với lịch thực tế; ngày minh họa theo fixture unit test hiện có.

## 3. Theo dõi luồng đặt lịch đã kiểm chứng

```text
PATIENT JWT và validation DTO
→ actor ID đã xác thực và thời điểm khung giờ chuyển từ phòng khám sang UTC
→ MySQL transaction
→ khóa lịch làm việc, kiểm tra bác sĩ/tài khoản, ngày, khoảng giờ, ranh giới khung, bắt đầu trong tương lai
→ kiểm tra xung đột lịch khám đang hoạt động
→ upsert hồ sơ bệnh nhân bằng user ID đã xác thực
→ tạo lịch PENDING cùng lịch sử ban đầu
→ commit và trả response chi tiết
```

Hồ sơ bệnh nhân được xác định bằng `patient.upsert()` trong transaction, sau kiểm tra lịch làm việc và xung đột. Thứ tự này theo code persistence thực tế. Hồ sơ còn thiếu có thể được tạo chỉ với danh tính liên kết; không tự thêm thông tin cá nhân.

## 4. Hiểu response thành công

Controller khai báo 201 Created với `AppointmentDetailResponseDto`. Ví dụ đã làm sạch sau theo fixture lịch khám lưu trữ và ánh xạ response trong unit test service; UUID được thay bằng placeholder. Đây không phải response vừa thực thi.

```json
{
  "id": "<appointment-id>",
  "patientId": "<patient-id>",
  "doctorId": "<doctor-id>",
  "scheduleId": "<schedule-id>",
  "appointmentDate": "2036-01-15",
  "startTime": "09:00",
  "endTime": "09:30",
  "status": "PENDING",
  "createdAt": "2030-01-01T00:00:00.000Z",
  "updatedAt": "2030-01-01T00:00:00.000Z",
  "history": [
    {
      "id": "<history-id>",
      "oldStatus": null,
      "newStatus": "PENDING",
      "changedBy": "<patient-user-id>",
      "changedAt": "2030-01-01T00:00:00.000Z",
      "reason": null
    }
  ]
}
```

`patientId` của lịch khám là định danh hồ sơ bệnh nhân, còn `changedBy` trong history là định danh tài khoản user của actor. Các trường lịch giữ ngày/giờ phòng khám; timestamp kiểm tra là chuỗi ISO-8601 UTC.

| Lỗi | Response thực tế |
| --- | --- |
| DTO sai định dạng hoặc ngày không có thật | 400 `VALIDATION_ERROR`. |
| Không có lịch làm việc trong query join đặt lịch | 404 `NOT_FOUND`. |
| Bác sĩ hoặc tài khoản bác sĩ liên kết không hoạt động | 400 `DOCTOR_INACTIVE`. |
| Sai bác sĩ/ngày so với lịch đã chọn, lịch CLOSED, bắt đầu lệch khung/ngoài khoảng/đã qua | 400 `INVALID_APPOINTMENT_SLOT`. |
| Khung đã có lịch khám đang hoạt động | 409 `SLOT_ALREADY_BOOKED`. |

Thiếu/sai xác thực trả về 401; vai trò không phải PATIENT trả về 403.

## 5. Xem lịch khám theo phạm vi server kiểm soát

| Endpoint | Phạm vi đã kiểm chứng | Response |
| --- | --- | --- |
| `GET /api/v1/appointments/me` | PATIENT: lịch khám của mình; DOCTOR: lịch khám gắn với hồ sơ bác sĩ của mình; ADMIN: toàn bộ lịch khám. | 200 danh sách phân trang. |
| `GET /api/v1/appointments/:id` | PATIENT: lịch của mình; DOCTOR: lịch được phân công; ADMIN: mọi lịch khám. | 200 chi tiết gồm history. |

Danh sách nhận `status` và `date` phòng khám tùy chọn, cùng `page` (mặc định 1) và `pageSize` (mặc định 20, tối đa 100). Sắp theo ngày khám giảm dần, giờ bắt đầu giảm dần, rồi ID tăng dần. Response ngoài có `items`, `page`, `pageSize`, `totalItems` và `totalPages`; item danh sách không chứa history.

Khi xem danh sách, tài khoản PATIENT/DOCTOR thiếu hồ sơ cần thiết nhận 403 `PROFILE_REQUIRED`. Bản ghi chi tiết không tồn tại trả về 404; bản ghi tồn tại nhưng nằm ngoài ownership/phân công trả về 403 `OWNERSHIP_REQUIRED`.

<!-- TODO_SCREENSHOT: tạo lịch khám thành công, HTTP 201, đã che token thật và định danh cá nhân. -->

