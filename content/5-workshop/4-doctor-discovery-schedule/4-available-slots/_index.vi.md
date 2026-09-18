---
title: "5.4.4. Tính toán khung giờ trống"
weight: 4
chapter: false
---

# 5.4.4. Tính toán khung giờ trống

## 1. Yêu cầu khung giờ theo ngày tại phòng khám

Endpoint công khai là `GET /api/v1/doctors/:id/available-slots?date=YYYY-MM-DD`. Triển khai nằm trong `SchedulesController` và `SchedulesService.availableSlots()`; không cần JWT.

```http
GET /api/v1/doctors/<doctor-id>/available-slots?date=2026-09-10
```

Thay `<doctor-id>` bằng UUID bác sĩ hợp lệ từ tra cứu. Đây là ngày lịch sử được cung cấp cho Workshop. Khi ngày đã thuộc quá khứ, thuật toán loại tất cả thời điểm bắt đầu đã qua; dùng ngày tương lai tại phòng khám có lịch OPEN để xem khung giờ sắp tới.

`AvailableSlotsQueryDto` yêu cầu `date` theo `YYYY-MM-DD`. Time service kiểm tra cả ngày có thật trên lịch và định dạng.

## 2. Theo dõi cách tính thực tế

1. Tìm bác sĩ, yêu cầu cả bác sĩ và user liên kết đang hoạt động.
2. Parse ngày tại phòng khám được yêu cầu.
3. Tải lịch OPEN của bác sĩ trong ngày, sắp theo giờ bắt đầu rồi ID.
4. Song song, tải lịch khám của bác sĩ/ngày đó có trạng thái PENDING hoặc CONFIRMED.
5. Tạo tập hợp các giờ bắt đầu đã bị chiếm.
6. Chia từng khoảng làm việc theo `slotDurationMinutes`, chỉ giữ khung đầy đủ có giờ kết thúc nằm trong khoảng.
7. Chuyển ngày/giờ phòng khám của từng khung thành thời điểm UTC. Bỏ thời điểm bắt đầu trước hoặc bằng hiện tại và các giờ bắt đầu đã bị chiếm.
8. Trả các khung còn lại trong response ngoài.

Hằng dùng chung `ACTIVE_APPOINTMENT_STATUSES` chứa đúng PENDING và CONFIRMED. Lịch khám CANCELLED và COMPLETED không được đưa vào query giờ bắt đầu đã chiếm; do đó CANCELLED không ngăn đặt lại.

Đoạn nguyên bản sau trích từ `SchedulesService.availableSlots()`, đã lược bỏ vòng lặp sinh khung bao quanh:

```typescript
const startAt = this.clinicTime.toUtcInstant(date, displayStart);
if (startAt <= now || bookedStarts.has(`${displayStart}:00`)) {
  continue;
}
generated.push({
  startAt: startAt.toISOString(),
  endAt: this.clinicTime.toUtcInstant(date, displayEnd).toISOString(),
  displayStart,
  displayEnd,
  available: true as const,
});
```

Khung giờ trống được tính tại thời điểm query. Thao tác đặt lịch tiếp theo vẫn phải áp dụng quy tắc đặt lịch riêng; response đọc này không giữ chỗ.

## 3. Hiểu response và múi giờ

Response ngoài đã kiểm chứng gồm `doctorId`, `date`, `timezone` và `slots`. Mỗi khung có `startAt`, `endAt`, `displayStart`, `displayEnd` và `available: true`.

`ClinicTimeService` dùng múi giờ phòng khám được cấu hình, `Asia/Ho_Chi_Minh`, để chuyển các giá trị lịch local thành thời điểm UTC. `startAt` và `endAt` là chuỗi ISO-8601 UTC; giờ hiển thị giữ dạng `HH:mm` theo múi giờ phòng khám.

Ví dụ cụ thể có cơ sở từ source: unit test Schedules service cố định đồng hồ tại `2030-01-15T02:15:00.000Z` (09:15 giờ phòng khám). Lịch OPEN 09:00–11:00 dùng khung 30 phút, còn 09:30 đã bị chiếm. Kết quả loại giờ bắt đầu 09:00 đã qua và giờ 09:30 bị chiếm:

```json
{
  "doctorId": "<doctor-id>",
  "date": "2030-01-15",
  "timezone": "Asia/Ho_Chi_Minh",
  "slots": [
    {
      "startAt": "2030-01-15T03:00:00.000Z",
      "endAt": "2030-01-15T03:30:00.000Z",
      "displayStart": "10:00",
      "displayEnd": "10:30",
      "available": true
    },
    {
      "startAt": "2030-01-15T03:30:00.000Z",
      "endAt": "2030-01-15T04:00:00.000Z",
      "displayStart": "10:30",
      "displayEnd": "11:00",
      "available": true
    }
  ]
}
```

Đây là response dự kiến của test với ID bác sĩ giả trong test được thay bằng placeholder, không phải response Backend vừa chạy.

## 4. Phân biệt lỗi với không còn khung giờ trống

| Điều kiện | Kết quả thực tế |
| --- | --- |
| UUID bác sĩ sai định dạng | 400 `VALIDATION_ERROR` tại biên API. |
| UUID đúng định dạng nhưng không có bác sĩ, bác sĩ không hoạt động hoặc user liên kết không hoạt động | 404 `NOT_FOUND`. |
| Thiếu ngày, ngày sai định dạng hoặc ngày không có thật | 400 `VALIDATION_ERROR` khi bước validation ngày được thực hiện. |
| Không có lịch OPEN trong ngày | 200 với `slots: []`. |
| Mọi khung sinh ra đã bị chiếm hoặc thuộc quá khứ | 200 với `slots: []`. |

ID bác sĩ sai nhưng đúng định dạng trả về 404, sau đó dùng bác sĩ đang hoạt động hợp lệ trả về 200, phù hợp lần kiểm tra thủ công trước đây. Mảng khung giờ rỗng không phải lỗi bác sĩ không tồn tại.

## 5. Kiểm tra tính toán tại local

Trong Swagger, chọn bác sĩ đang hoạt động và ngày tương lai có lịch OPEN đã biết. Đối chiếu số lượng khung cùng giờ hiển thị với lịch đó. Unit test đã kiểm tra bao gồm loại khung đã qua, lịch khám đang hoạt động và đầu ra UTC; integration test persistence M3 còn kiểm tra CANCELLED không chặn khung giờ.

<!-- TODO_SCREENSHOT: response available-slots hiển thị timestamp UTC và giờ phòng khám, đã che định danh riêng tư. -->

