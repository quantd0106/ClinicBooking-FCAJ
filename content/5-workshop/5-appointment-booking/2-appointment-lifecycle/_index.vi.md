---
title: "5.5.2. Vòng đời lịch khám"
weight: 2
chapter: false
---

# 5.5.2. Vòng đời lịch khám

## 1. Theo dõi các trạng thái được phép

Lịch khám bắt đầu ở PENDING. Triển khai `assertTransition()` đã kiểm chứng chỉ cho phép các bước sau:

{{< mermaid >}}
graph TD
  PENDING --> CONFIRMED
  CONFIRMED --> COMPLETED
  PENDING --> CANCELLED
  CONFIRMED --> CANCELLED
{{< /mermaid >}}

COMPLETED và CANCELLED là trạng thái kết thúc trong triển khai này. Không được lặp lại trạng thái hoặc chuyển trực tiếp từ PENDING sang COMPLETED.

## 2. Áp dụng vai trò và ownership

Mọi endpoint lịch khám yêu cầu xác thực JWT. Endpoint chuyển trạng thái thành công trả về 200 với chi tiết lịch khám cùng history.

| Endpoint | Actor được phép | Trạng thái đích |
| --- | --- | --- |
| `PATCH /api/v1/appointments/:id/cancel` | PATIENT sở hữu lịch, DOCTOR được phân công hoặc ADMIN override ownership. | CANCELLED từ PENDING hoặc CONFIRMED. |
| `PATCH /api/v1/appointments/:id/status` | DOCTOR được phân công hoặc ADMIN override phân công. | CONFIRMED từ PENDING; COMPLETED từ CONFIRMED. |

ADMIN override ownership/phân công, không bỏ qua tính hợp lệ của vòng đời. PATIENT không được dùng endpoint status; bác sĩ không liên quan không được chuyển trạng thái lịch của bác sĩ khác.

## 3. Hủy lịch với lý do tùy chọn

```http
PATCH /api/v1/appointments/<appointment-id>/cancel
Authorization: Bearer <patient-token>
```

```json
{
  "reason": "Schedule conflict"
}
```

`CancelAppointmentDto` cho phép chuỗi `reason` tùy chọn tối đa 500 ký tự; không bắt buộc có body. Hủy lịch ghi history CANCELLED cùng actor và lý do đã gửi.

CANCELLED không còn tham gia kiểm tra xung đột khung giờ đang hoạt động. Đặt lại có thể thành công nếu lịch làm việc vẫn OPEN, bác sĩ/tài khoản hoạt động, khung đúng ranh giới và còn thuộc tương lai.

## 4. Xác nhận rồi hoàn thành

Dùng token riêng của bác sĩ được phân công hoặc token ADMIN với endpoint status:

```http
PATCH /api/v1/appointments/<appointment-id>/status
Authorization: Bearer <doctor-token>
```

Đầu tiên, với lịch PENDING:

```json
{
  "status": "CONFIRMED"
}
```

Sau đó, với lịch CONFIRMED:

```json
{
  "status": "COMPLETED"
}
```

`UpdateAppointmentStatusDto` chỉ nhận CONFIRMED hoặc COMPLETED. DTO này không có trường reason. Hủy lịch dùng endpoint cancel riêng.

## 5. Từ chối thay đổi không hợp lệ hoặc xung đột

Phương thức nguyên bản sau lấy từ `src/database/appointment-persistence.service.ts`; đã lược bỏ class bao quanh:

```typescript
private assertTransition(oldStatus: AppointmentStatus, newStatus: AppointmentStatus): void {
  const allowed =
    (oldStatus === AppointmentStatus.PENDING &&
      (newStatus === AppointmentStatus.CONFIRMED || newStatus === AppointmentStatus.CANCELLED)) ||
    (oldStatus === AppointmentStatus.CONFIRMED &&
      (newStatus === AppointmentStatus.COMPLETED || newStatus === AppointmentStatus.CANCELLED));
  if (!allowed) {
    throw new InvalidAppointmentTransitionError();
  }
}
```

Tầng persistence đọc trạng thái quan sát được và kiểm tra chuyển trạng thái trước. Trong transaction, tầng này khóa lịch khám bằng `FOR UPDATE`, kiểm tra trạng thái vẫn khớp lần đọc trước, kiểm tra quyền actor, kiểm tra chuyển trạng thái lần nữa, cập nhật trạng thái và thêm history nguyên tử.

| Lỗi | Response |
| --- | --- |
| Trạng thái DTO không hỗ trợ, như PENDING/CANCELLED gửi đến endpoint status | 400 `VALIDATION_ERROR`. |
| Trạng thái đích đúng DTO nhưng không được phép từ trạng thái hiện tại, hoặc hủy không hợp lệ | 400 `INVALID_APPOINTMENT_TRANSITION`. |
| Trạng thái đã đổi giữa lần quan sát và lần đọc có khóa | 409 `APPOINTMENT_CHANGED`. |
| Actor không có ownership/phân công cho thao tác được phép | 403 `OWNERSHIP_REQUIRED`. |
| Không tồn tại lịch khám | 404 `NOT_FOUND`. |

Lỗi guard xác thực/vai trò vẫn là 401/403. Kiểm tra chuyển trạng thái độc lập với quyền theo vai trò; ADMIN cũng nhận lỗi khi yêu cầu chuyển trạng thái không được hỗ trợ.

<!-- TODO_SCREENSHOT: chuyển trạng thái lịch khám hợp lệ và history mới, đã che định danh riêng tư. -->

