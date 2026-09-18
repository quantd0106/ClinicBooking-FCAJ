---
title: "5.4.2. Tra cứu bác sĩ"
weight: 2
chapter: false
---

# 5.4.2. Tra cứu bác sĩ

## 1. Xác định thao tác công khai và quản trị

Triển khai nằm trong `src/doctors/doctors.controller.ts`, `doctors.service.ts` và `dto/doctor.dto.ts`.

| Method | Endpoint | Mục đích | Quyền truy cập |
| --- | --- | --- | --- |
| GET | `/api/v1/doctors` | Liệt kê/tìm bác sĩ đang hoạt động; 200. | Công khai. |
| GET | `/api/v1/doctors/:id` | Lấy thông tin bác sĩ đang hoạt động; 200. | Công khai. |
| PATCH | `/api/v1/doctors/:id` | Bật/tắt trạng thái hoạt động bác sĩ; 200. | Bearer JWT, ADMIN. |

Thao tác activation dùng PATCH ngay trên tài nguyên bác sĩ. Controller khai báo `@Patch(':id')`; không khai báo hậu tố `/activation`.

## 2. Tra cứu và lọc bác sĩ

```text
Client
→ GET /api/v1/doctors
→ DoctorsService tạo query tra cứu
→ Prisma tải bác sĩ và liên kết chuyên khoa đang hoạt động
→ response
```

Các query parameter đã kiểm chứng:

| Parameter | Hành vi |
| --- | --- |
| `search` | Tùy chọn, được trim, tối đa 150 ký tự; tìm `displayName` bằng `contains`. |
| `specialtyId` | UUID tùy chọn; yêu cầu liên kết với chuyên khoa đang hoạt động đó. |
| `page` | Số nguyên, tối thiểu 1; mặc định 1. |
| `pageSize` | Số nguyên, 1–100; mặc định 20. |

```http
GET /api/v1/doctors?search=Nguyen&page=1&pageSize=20
GET /api/v1/doctors?specialtyId=<specialty-id>&page=1&pageSize=20
```

Thay `<specialty-id>` bằng UUID từ response chuyên khoa local. Tra cứu yêu cầu cả `doctor.isActive` và `isActive` của user liên kết. Chỉ chuyên khoa đang hoạt động được đưa vào response, sắp theo tên chuyên khoa; danh sách bác sĩ sắp theo tên hiển thị rồi ID.

Response ngoài gồm `items`, `page`, `pageSize`, `totalItems` và `totalPages`. Mỗi bác sĩ có `id`, `name` (ánh xạ từ `displayName`), `bio`, `isActive` và `specialties`; mỗi chuyên khoa trong response có `id` và `name`.

## 3. Lấy chi tiết bác sĩ

```http
GET /api/v1/doctors/<doctor-id>
```

Dùng ID bác sĩ lấy từ danh sách tra cứu, không dùng ID tài khoản user. Query nguyên bản sau trích từ `DoctorsService.detail()`; đã bỏ phần xử lý lỗi và ánh xạ response:

```typescript
const doctor = await this.prisma.doctor.findFirst({
  where: { id, isActive: true, user: { isActive: true } },
  include: doctorInclude,
});
```

Response chi tiết có cùng các trường bác sĩ như một item trong danh sách. Bác sĩ không tồn tại, không hoạt động hoặc có user liên kết không hoạt động trả về 404 `NOT_FOUND`. Route UUID sai định dạng trả về 400. Danh sách rỗng vẫn là response phân trang thành công.

## 4. Điều khiển activation bằng ADMIN

Activation quyết định bác sĩ có được tra cứu công khai hay không, nên đây là thao tác quản trị. Handler dùng `JwtAuthGuard`, `RolesGuard` và `@Roles(Role.ADMIN)`.

DTO request đã kiểm chứng chỉ có boolean `isActive` bắt buộc:

```http
PATCH /api/v1/doctors/<doctor-id>
Authorization: Bearer <token>
```

```json
{
  "isActive": false
}
```

Response gồm `id`, `isActive` và `updatedAt`. Đặt `isActive: true` bật lại bản ghi bác sĩ, nhưng tra cứu công khai vẫn yêu cầu tài khoản user liên kết hoạt động. Bác sĩ không tồn tại trả về 404; thiếu/sai xác thực trả về 401, vai trò không được phép trả về 403.

## 5. Kiểm tra tra cứu

Trong Swagger local, đối chiếu danh sách, chi tiết và bộ lọc chuyên khoa. Với tài khoản ADMIN riêng, ngừng hoạt động một bác sĩ test và kiểm tra bác sĩ không còn trong tra cứu công khai, endpoint chi tiết công khai trả về 404. Test Doctors service kiểm tra điều kiện user hoạt động và phân trang ổn định; các test API M3 kiểm tra giới hạn vai trò quản trị.

<!-- TODO_SCREENSHOT: tra cứu bác sĩ trong Swagger và response activation ADMIN đã che dữ liệu nhạy cảm. -->

