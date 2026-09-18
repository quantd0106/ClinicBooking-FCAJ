---
title: "5.4.1. Quản lý chuyên khoa"
weight: 1
chapter: false
---

# 5.4.1. Quản lý chuyên khoa

## 1. Hiểu danh mục chuyên khoa

Chuyên khoa biểu diễn một lĩnh vực khám và điều trị. Tách chuyên khoa khỏi hồ sơ bác sĩ giúp tránh lặp dữ liệu danh mục và hỗ trợ quan hệ nhiều-nhiều qua `doctor_specialties`: một bác sĩ có thể thuộc nhiều chuyên khoa, một chuyên khoa có thể liên kết nhiều bác sĩ.

Triển khai nằm trong `src/specialties/specialties.controller.ts`, `specialties.service.ts` và `dto/specialty.dto.ts`.

## 2. Xác định endpoint và quyền truy cập

| Method | Endpoint | Mục đích | Quyền truy cập |
| --- | --- | --- | --- |
| GET | `/api/v1/specialties` | Liệt kê/tìm chuyên khoa đang hoạt động; 200. | Công khai; không cần JWT. |
| POST | `/api/v1/specialties` | Tạo chuyên khoa; 201. | Bearer JWT, ADMIN. |
| PATCH | `/api/v1/specialties/:id` | Cập nhật các trường chuyên khoa; 200. | Bearer JWT, ADMIN. |
| DELETE | `/api/v1/specialties/:id` | Ngừng hoạt động chuyên khoa; 204, không có response body. | Bearer JWT, ADMIN. |

Các handler ghi dữ liệu dùng `JwtAuthGuard`, `RolesGuard` và `@Roles(Role.ADMIN)`. DELETE đặt `isActive: false`; không xóa vật lý chuyên khoa hoặc liên kết với bác sĩ.

## 3. Tìm kiếm và phân trang

`SpecialtyQueryDto` nhận query parameter `search` tùy chọn, được trim và giới hạn 100 ký tự. DTO phân trang dùng chung cung cấp `page` (mặc định 1, tối thiểu 1) và `pageSize` (mặc định 20, khoảng 1–100).

```http
GET /api/v1/specialties?search=cardio&page=1&pageSize=20
```

Tìm kiếm dùng điều kiện `contains` trên tên chuyên khoa. Kết quả luôn yêu cầu `isActive: true`, sắp xếp theo tên rồi ID. Service lấy trang kết quả và tổng số trong một Prisma transaction.

Đoạn nguyên bản sau trích từ `SpecialtiesService.list()`, đã lược bỏ phần query và response bao quanh:

```typescript
const where: Prisma.SpecialtyWhereInput = {
  isActive: true,
  ...(query.search ? { name: { contains: query.search } } : {}),
};
```

Response ngoài gồm `items`, `page`, `pageSize`, `totalItems` và `totalPages`. Mỗi item có `id`, `name`, `description` và `isActive`. Không khẳng định thêm về phân biệt chữ hoa/thường ngoài hành vi query cơ sở dữ liệu thực tế.

## 4. Tạo và cập nhật theo DTO đã kiểm chứng

Ví dụ request body tạo chuyên khoa an toàn:

```json
{
  "name": "Cardiology",
  "description": "Heart and circulatory system care."
}
```

`CreateSpecialtyDto` nhận `name` được trim, dài 2–100 ký tự và `description` tùy chọn tối đa 2000 ký tự. Description có thể là null; service lưu description rỗng thành null.

`UpdateSpecialtyDto` chuyển các trường này thành tùy chọn và thêm boolean `isActive` tùy chọn. Cần gửi ít nhất một trường; PATCH body rỗng trả về 400 `VALIDATION_ERROR`.

Tên trùng trả về 409 `SPECIALTY_ALREADY_EXISTS`. UUID đúng định dạng nhưng không có chuyên khoa tương ứng trả về 404 `NOT_FOUND`; route ID sai định dạng và DTO không hợp lệ trả về 400. Lỗi xác thực và vai trò tuân theo quy tắc 401/403 ở 5.3.

## 5. Kiểm tra trong Swagger local

Dùng `/api/docs` để tra cứu không cần token, sau đó kiểm tra thao tác ghi danh mục với tài khoản ADMIN test riêng. Kiểm tra chuyên khoa đã ngừng hoạt động không còn xuất hiện trong danh sách công khai. Lần kiểm tra HTTP 200 với `search=cardio` trước đây phù hợp hợp đồng query này; kết quả phụ thuộc dữ liệu local.

> 📷 Ảnh cần bổ sung: các endpoint Specialty trong Swagger và kết quả tìm kiếm đã che dữ liệu nhạy cảm.

