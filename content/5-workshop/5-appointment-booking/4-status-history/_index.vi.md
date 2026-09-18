---
title: "5.5.4. Lịch sử trạng thái lịch khám"
weight: 4
chapter: false
---

# 5.5.4. Lịch sử trạng thái lịch khám

## 1. Giữ dấu vết kiểm tra

`appointment_status_history` ghi lại cách một lịch khám đi đến trạng thái hiện tại. Chỉ lưu `appointments.status` sẽ mất chuỗi xác nhận, hoàn thành hoặc hủy, khiến việc theo dõi sự kiện nghiệp vụ và debug khó hơn.

Model Prisma là `AppointmentStatusHistory` trong `prisma/schema.prisma`. Các thao tác ghi khi đặt lịch và chuyển trạng thái được triển khai bởi `AppointmentPersistenceService`.

## 2. Đọc các trường lưu trữ đã kiểm chứng

| Trường Prisma | Ý nghĩa lưu trữ |
| --- | --- |
| `id` | UUID history, được sinh mặc định. |
| `appointmentId` | Tham chiếu lịch khám; ánh xạ sang `appointment_id`. |
| `oldStatus` | Trạng thái lịch khám trước đó, có thể null; ánh xạ sang `old_status`. |
| `newStatus` | Trạng thái lịch khám mới; ánh xạ sang `new_status`. |
| `changedBy` | User ID của actor, liên kết với `User`; ánh xạ sang `changed_by`. |
| `changedAt` | Timestamp mặc định `now()`; ánh xạ sang `changed_at`. |
| `reason` | Chuỗi có thể null, tối đa 500 ký tự. |

Model ánh xạ sang `appointment_status_history`. `changedBy` tham chiếu tài khoản user, không phải ID hồ sơ bệnh nhân hay bác sĩ.

## 3. Tạo history PENDING ban đầu nguyên tử

Transaction đặt lịch tạo lịch PENDING kèm nested history. Đoạn thuộc tính object nguyên bản sau trích từ `bookAppointment()`; đã lược bỏ các thuộc tính lịch khám khác và lời gọi create bao quanh:

```typescript
history: {
  create: {
    oldStatus: null,
    newStatus: AppointmentStatus.PENDING,
    changedBy: input.patientUserId,
  },
},
```

Khi tạo chưa có trạng thái trước, nên `oldStatus` là null. Actor là user ID của bệnh nhân đã xác thực. Schema cung cấp ID history và timestamp; reason ban đầu là null.

Tạo lịch khám và history ban đầu đều phải thành công trước khi transaction commit. Request đặt lịch bị từ chối không tạo lịch chưa đầy đủ hay history khẳng định tạo lịch thành công.

## 4. Thêm history cho mỗi chuyển trạng thái hợp lệ

Trong `transitionAppointment()`, cập nhật trạng thái lịch khám và lời gọi tạo history nguyên bản sau dùng cùng transaction:

```typescript
await transaction.appointmentStatusHistory.create({
  data: {
    appointmentId,
    oldStatus: appointment.status,
    newStatus,
    changedBy: actor.id,
    reason,
  },
});
```

History phản ánh trạng thái trước của lịch đã khóa và actor đã xác thực. Xác nhận/hoàn thành dùng actor DOCTOR được phân công hoặc ADMIN được phép; hủy dùng PATIENT sở hữu lịch, DOCTOR được phân công hoặc ADMIN theo [5.5.2](../2-appointment-lifecycle/).

`CancelAppointmentDto` hỗ trợ chuỗi lý do tùy chọn tối đa 500 ký tự. DTO cập nhật status không nhận trường reason. Khi không cung cấp lý do, trường nullable giữ giá trị null.

| Sự kiện | History trước → sau |
| --- | --- |
| Tạo lịch khám | null → PENDING |
| Bác sĩ/Admin xác nhận | PENDING → CONFIRMED |
| Bác sĩ/Admin hoàn thành | CONFIRMED → COMPLETED |
| Hủy được phép | PENDING → CANCELLED hoặc CONFIRMED → CANCELLED |

Ứng dụng thêm dòng history mới thay vì ghi đè các dòng trước. Nếu chuyển trạng thái bị từ chối hoặc transaction thất bại, không commit bản ghi kiểm tra khẳng định thay đổi đó đã thành công.

## 5. Kiểm tra history qua chi tiết đã phân quyền

`GET /api/v1/appointments/:id` và response create/cancel/status thành công có `history`, sắp theo `changedAt` tăng dần. Mỗi item history API gồm `id`, `oldStatus`, `newStatus`, `changedBy`, `changedAt` và `reason`; response chi tiết không lặp lại `appointmentId` trong từng item.

Response phân trang `/appointments/me` không chứa history. Dùng quyền xem chi tiết phù hợp để đọc timeline: lịch của mình với PATIENT, lịch được phân công với DOCTOR, mọi lịch với ADMIN.

Kiểm tra tạo lịch thành công và chuyển trạng thái được phép, rồi đối chiếu timeline trả về với các dòng history trong cơ sở dữ liệu local. Kiểm tra hủy lịch giữ các entry PENDING/CONFIRMED trước đó và thêm CANCELLED cùng actor/lý do. Đặt lại tạo lịch khám mới với history ban đầu riêng, giữ timeline của lịch đã hủy.

Các test M4 đã kiểm tra bao gồm history ban đầu, history vòng đời, actor/lý do hủy và một dòng ban đầu cho request thắng khi đặt đồng thời. Đây là kết quả dự kiến có cơ sở từ source để bạn kiểm tra tại local.

> 📷 Ảnh cần bổ sung: các bản ghi appointment_status_history và timeline chi tiết được phép truy cập tương ứng. Che user ID thật và thông tin cá nhân trong reason.

