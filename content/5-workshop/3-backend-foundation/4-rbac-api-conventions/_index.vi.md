---
title: "5.3.4. RBAC và quy ước API"
weight: 4
chapter: false
---

# 5.3.4. RBAC và quy ước API

## 1. Tách biệt xác thực và phân quyền

Xác thực trả lời **“Người dùng là ai?”** Phân quyền trả lời **“Người dùng đã xác thực này được phép làm gì?”**

Dự án có đúng ba vai trò: `PATIENT`, `DOCTOR` và `ADMIN`. Xác thực JWT xác định danh tính người dùng hiện tại. Kiểm tra vai trò và ownership của tài nguyên tiếp tục kiểm soát quyền truy cập.

```text
JwtAuthGuard
→ danh tính đã xác thực
→ RolesGuard kiểm tra vai trò yêu cầu
→ controller
→ service áp dụng quy tắc nghiệp vụ và ownership
```

## 2. Áp dụng guard/decorator đã kiểm chứng

`src/common/auth/roles.decorator.ts` định nghĩa decorator `Roles` thực tế. Đoạn sau chỉ lược bỏ import:

```typescript
export const ROLES_KEY = 'roles';
export const Roles = (...roles: Role[]): MethodDecorator & ClassDecorator =>
  SetMetadata(ROLES_KEY, roles);
```

`RolesGuard` đọc metadata này từ handler hoặc controller bằng `Reflector.getAllAndOverride()`. Khi đã cấu hình các vai trò cần thiết, guard kiểm tra `request.user` đã xác thực rồi kiểm tra vai trò của user có được phép hay không.

Ví dụ thực tế, `src/appointments/appointments.controller.ts` áp dụng `@UseGuards(JwtAuthGuard, RolesGuard)` ở cấp class và `@Roles(Role.PATIENT)` cho handler tạo lịch. Đoạn decorator nguyên bản sau lược bỏ các decorator Swagger; hai comment đánh dấu phần code bao quanh đã bỏ:

```typescript
// Class-level decorators:
@UseGuards(JwtAuthGuard, RolesGuard)
// Method-level decorators on create():
@Post()
@Roles(Role.PATIENT)
```

Đây là trích đoạn decorator, không phải controller có thể biên dịch độc lập. Kết hợp các decorator này giới hạn `POST /api/v1/appointments` cho PATIENT đã xác thực. Hành vi đặt lịch chi tiết thuộc phần Appointment tiếp theo.

Vượt qua kiểm tra vai trò không đồng nghĩa được truy cập mọi tài nguyên. Các kiểm tra ownership của dự án vẫn cần được thực hiện ở tầng service.

## 3. Phân biệt 401 và 403

| Response | Ý nghĩa | Ví dụ trong dự án |
| --- | --- | --- |
| 401 Unauthorized | Xác thực chưa thành công. | Thiếu token, token không hợp lệ/hết hạn; thông tin đăng nhập sai. |
| 403 Forbidden | Danh tính đã xác thực không có quyền. | Vai trò không được `RolesGuard` cho phép hoặc không đáp ứng ownership tài nguyên. |

Trong ví dụ, `JwtAuthGuard` chạy trước `RolesGuard`. `RolesGuard` cũng kiểm tra phòng vệ: trả về 401 nếu có yêu cầu vai trò nhưng không có user, và 403 nếu vai trò user không được phép.

## 4. Dùng tiền tố API và chính sách thời gian

Các route API nghiệp vụ dùng tiền tố `/api/v1` cấu hình tại `src/bootstrap.ts`. Swagger UI được đăng ký riêng tại `/api/docs`.

Dùng ISO-8601 cho các thời điểm ngày/giờ khi phù hợp, ưu tiên UTC với hậu tố `Z`. Backend tính toán theo UTC; múi giờ lịch khám/hiển thị là `Asia/Ho_Chi_Minh`.

Cần phân biệt trong triển khai thực tế:

- Các trường ngày/giờ lịch làm việc và lịch khám biểu diễn lịch tại phòng khám, với định dạng như `YYYY-MM-DD` và `HH:mm`.
- `ClinicTimeService` diễn giải các giá trị lịch này theo `APP_TIMEZONE`, chuyển thành thời điểm UTC để so sánh và validation.
- Xử lý theo UTC không có nghĩa một trường date hoặc time riêng của MySQL đã là thời điểm UTC hoàn chỉnh. Giữ ngữ cảnh múi giờ phòng khám khi kết hợp các giá trị này.

Nội dung dựa trên `src/common/time/clinic-time.service.ts` và tài liệu API/cơ sở dữ liệu của Backend.

## 5. Tuân theo quy ước HTTP response

| Status | Trường hợp sử dụng |
| --- | --- |
| 200 OK | Truy xuất hoặc đăng nhập thành công. |
| 201 Created | Tạo mới thành công, bao gồm đăng ký. |
| 400 Bad Request | DTO đầu vào hoặc dữ liệu nghiệp vụ không hợp lệ. |
| 401 Unauthorized | Thiếu xác thực hoặc xác thực không hợp lệ. |
| 403 Forbidden | Không được phép theo vai trò hoặc quyền trên tài nguyên. |
| 404 Not Found | Tài nguyên yêu cầu không tồn tại. |
| 409 Conflict | Xung đột như email đã tồn tại hoặc khung giờ đã được đặt. |

Xung đột đặt lịch ở phần tiếp theo dùng 409 cùng mã nghiệp vụ như `SLOT_ALREADY_BOOKED`. Phần nền tảng không triển khai hoặc lặp lại transaction đặt lịch chi tiết.

## 6. Đọc cấu trúc error response chuẩn hóa

`HttpExceptionFilter` toàn cục trả về đúng ba trường `statusCode`, `code` và `message`. Dòng sau được trích từ `src/common/errors/http-exception.filter.ts`:

```typescript
response.status(statusCode).json({ statusCode, code, message });
```

Ví dụ an toàn tương ứng DTO lỗi thông tin đăng nhập đã kiểm chứng:

```json
{
  "statusCode": 401,
  "code": "INVALID_CREDENTIALS",
  "message": "Invalid email or password."
}
```

Filter giữ mã nghiệp vụ khi exception cung cấp, nếu không sẽ chọn mã mặc định theo status. Mảng thông báo validation được nối thành một chuỗi, phân tách bằng dấu chấm phẩy. Lỗi chưa xử lý dùng HTTP 500 và thông báo chung, không công khai chi tiết exception nội bộ.

## Kiểm tra hoàn thành

Kiểm tra Swagger local hiển thị các endpoint Auth, đăng ký công khai không được chọn vai trò đặc quyền, route bảo vệ yêu cầu Bearer token và lỗi vai trò khác với lỗi xác thực. Không đưa token thật, thông tin đăng nhập hoặc dữ liệu cá nhân vào báo cáo.

