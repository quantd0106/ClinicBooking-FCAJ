---
title: "5.5.3. Ngăn đặt trùng lịch"
weight: 3
chapter: false
---

# 5.5.3. Ngăn đặt trùng lịch

## 1. Hiểu tình huống race

Response khung giờ trống là dữ liệu đọc, không giữ chỗ. Nếu thiếu bảo vệ transaction, hai bệnh nhân có thể cùng thấy 09:00 còn trống và cùng tạo lịch khám:

```text
Patient A kiểm tra 09:00 → còn trống
Patient B kiểm tra 09:00 → còn trống
A ghi lịch khám
B ghi lịch khám
→ hai lịch khám đang hoạt động cho cùng khung
```

Backend phải kiểm tra lại lúc đặt lịch. Định danh xung đột là cùng bác sĩ, ngày phòng khám và giờ bắt đầu, với trạng thái PENDING hoặc CONFIRMED.

## 2. Theo dõi transaction thực tế

`AppointmentPersistenceService.bookAppointment()` thực hiện các bước sau trong một Prisma/MySQL transaction:

1. Gọi `lockBookingSchedule()` cho lịch làm việc đã chọn.
2. Kiểm tra trạng thái hoạt động của bác sĩ và tài khoản user bác sĩ qua join.
3. Yêu cầu bác sĩ/ngày của lịch khớp request và trạng thái lịch là OPEN.
4. Yêu cầu giờ bắt đầu nằm trong khoảng, giờ kết thúc tính được nằm trong lịch và độ lệch bắt đầu đúng ranh giới `slotDurationMinutes`.
5. Yêu cầu thời điểm bắt đầu chuyển từ phòng khám sang UTC phải sau hiện tại.
6. Query lịch khám đang hoạt động theo bác sĩ/ngày/giờ bắt đầu; ném `SlotAlreadyBookedError` nếu có.
7. Upsert hồ sơ bệnh nhân bằng user ID đã xác thực.
8. Tạo lịch khám PENDING và history ban đầu dạng nested write.
9. Chỉ commit nếu mọi bước thành công.

Giờ kết thúc được tính từ độ dài khung của lịch đã khóa. Client không được thay thế giá trị này.

Các mệnh đề SQL nguyên bản sau trích từ `lockBookingSchedule()`; đã lược bỏ các cột SELECT và lời gọi Prisma bao quanh:

```sql
FROM doctor_schedules
INNER JOIN doctors ON doctors.id = doctor_schedules.doctor_id
INNER JOIN users ON users.id = doctors.user_id
WHERE doctor_schedules.id = ${scheduleId}
FOR UPDATE
```

`${scheduleId}` là biểu thức SQL template của Prisma, không phải ID tài nguyên thật hay một lệnh SQL độc lập.

Locking read của lịch làm việc tuần tự hóa các lần đặt cạnh tranh trong cùng ngữ cảnh lịch. Sau khi transaction đầu tiên commit, request đang chờ tiếp tục và kiểm tra lịch đang hoạt động trước khi ghi.

## 3. Kiểm tra trạng thái chặn khung bằng current read

Các trạng thái đang hoạt động được định nghĩa chính xác:

```typescript
export const ACTIVE_APPOINTMENT_STATUSES: AppointmentStatus[] = [
  AppointmentStatus.PENDING,
  AppointmentStatus.CONFIRMED,
];
```

`isSlotActivelyBooked()` lọc theo ID bác sĩ, ngày khám, giờ bắt đầu và các trạng thái này. Query dùng chuyển đổi MySQL rõ ràng `DATE(...)` và `TIME(...)`. Khi gọi trong transaction đặt lịch, query thêm `FOR UPDATE`, nên luồng đặt lịch dùng locking/current read.

CANCELLED và COMPLETED không nằm trong query đang hoạt động. Bản ghi đã hủy cùng lịch sử kiểm tra vẫn được giữ; không cần xóa chúng để đặt lại.

Schema Prisma định nghĩa index thông thường trên bác sĩ/ngày/giờ bắt đầu, **không phải UNIQUE constraint** của khung. Kiểm tra xung đột bằng transaction áp dụng quy tắc lịch đang hoạt động và vẫn cho phép đặt lại khung đã hủy.

## 4. Trả đúng lỗi xung đột

Đoạn sau lấy từ `AppointmentsService.mapPersistenceError()`; các nhánh không liên quan được lược bỏ:

```typescript
if (error instanceof SlotAlreadyBookedError) {
  throw new ConflictException({ code: ApiErrorCode.SlotAlreadyBooked, message: error.message });
}
```

Exception filter toàn cục trả về:

```json
{
  "statusCode": 409,
  "code": "SLOT_ALREADY_BOOKED",
  "message": "The selected appointment slot is no longer available."
}
```

Exception ngăn transaction commit các thay đổi lịch khám/history chưa đầy đủ.

## 5. Thực hiện kịch bản Workshop local an toàn

Dùng tài khoản PATIENT local riêng và lịch OPEN tương lai có khung 09:00 đúng ranh giới. Theo DTO request ở [5.5.1](../1-create-appointment/).

| Bước | Thao tác | Kết quả dự kiến |
| --- | --- | --- |
| 1 | Đăng nhập Patient A; giữ token riêng tư. | Sẵn sàng xác thực request. |
| 2 | POST bác sĩ/lịch làm việc/ngày/09:00 đã chọn với `<patient-a-token>`. | 201 Created, lịch PENDING và một history ban đầu. |
| 3 | Đăng nhập Patient B. | Danh tính PATIENT đã xác thực khác. |
| 4 | POST cùng bác sĩ/lịch làm việc/ngày/09:00 với `<patient-b-token>`. | 409 `SLOT_ALREADY_BOOKED`. |
| 5 | Kiểm tra lịch khám tương ứng trong cơ sở dữ liệu local được phép truy cập. | Đúng một lịch PENDING/CONFIRMED cho bác sĩ/ngày/giờ bắt đầu đó. |
| 6 | Patient A hủy `<appointment-id>` ban đầu của mình. | 200, CANCELLED và history hủy mới. |
| 7 | Patient B thử đặt lại khi khung vẫn hợp lệ và còn thuộc tương lai. | 201; lịch đã hủy không còn chặn khung. |

Để thử đồng thời, gửi hai request đặt lịch cùng lúc. Bất kỳ bệnh nhân nào cũng có thể thành công trước; kỳ vọng một thành công và một 409, không cố định người thắng.

Integration test M4 đã kiểm tra dùng hai lời gọi `appointments.create()` đồng thời qua `Promise.allSettled()` tại 09:30. Test khẳng định một thành công, một 409 `SLOT_ALREADY_BOOKED`, một lịch khám đang hoạt động và một history ban đầu. Kịch bản 09:00 phía trên áp dụng cùng quy tắc với khung đúng ranh giới khác.

{{< notice info >}}
Khung giờ trống có thể thay đổi sau tra cứu. Đặt lịch phải validation và kiểm tra lại trong transaction. Hủy lịch chỉ giải phóng khung đang hoạt động để đặt lại khi các điều kiện bác sĩ, lịch làm việc và bắt đầu trong tương lai vẫn cho phép.
{{< /notice >}}

<!-- TODO_SCREENSHOT: đặt lịch thành công, xung đột đặt trùng và cơ sở dữ liệu local chỉ có một lịch đang hoạt động. Che token thật và định danh riêng tư. -->

