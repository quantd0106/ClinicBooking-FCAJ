---
title: "5.3.3. Xác thực JWT"
weight: 3
chapter: false
---

# 5.3.3. Xác thực JWT

## Mục tiêu và các thành phần đã kiểm chứng

Hiểu cách Backend đăng ký bệnh nhân, cấp JWT access token và xác thực request đến endpoint được bảo vệ. Triển khai đã kiểm chứng sử dụng `AuthController`, `AuthService`, `UsersService`, `JwtStrategy` và `JwtAuthGuard`.

| Endpoint | Quyền truy cập | Response thành công |
| --- | --- | --- |
| `POST /api/v1/auth/register` | Công khai | 201; `id`, `email`, `role`. |
| `POST /api/v1/auth/login` | Công khai | 200; `accessToken`, `tokenType`, `expiresIn` và `user` gồm `id`, `role`. |
| `GET /api/v1/auth/me` | Bearer JWT | 200; `id`, `email`, `role` của người dùng đã xác thực. |

Các trường này theo `src/auth/dto/auth-response.dto.ts`. Response thông tin tài khoản không chứa password hash; access token là dữ liệu nhạy cảm và không được công khai.

## 1. Đăng ký tài khoản PATIENT

```text
Client
→ POST /api/v1/auth/register
→ kiểm tra RegisterDto
→ hash mật khẩu bằng bcrypt
→ tạo user với vai trò PATIENT cùng hồ sơ bệnh nhân liên kết
→ trả về id, email, role
```

`RegisterDto` chỉ chứa `email` và `password`. Email được trim, chuyển thành chữ thường, kiểm tra định dạng email và giới hạn 191 ký tự. Mật khẩu được kiểm tra là chuỗi dài 12–72 ký tự.

Đăng ký công khai **chỉ tạo PATIENT**. Client không được chọn DOCTOR hoặc ADMIN. Trường `role` gửi thêm nằm ngoài hợp đồng DTO và bị `ValidationPipe` toàn cục từ chối.

`AuthService.register()` dùng `bcrypt.hash()` với `BCRYPT_ROUNDS = 10` đã kiểm chứng. `UsersService.createPatient()` cố định vai trò và tạo hồ sơ bệnh nhân liên kết bằng nested write của Prisma. Đoạn sau trích nguyên bản từ phương thức này; đã lược bỏ hàm bao ngoài:

```typescript
const data: Prisma.UserCreateInput = {
  email,
  passwordHash,
  role: Role.PATIENT,
  isActive: true,
  patient: { create: {} },
};
return this.prisma.user.create({ data });
```

Giá trị lưu trữ là password hash, không phải mật khẩu dạng plaintext. Email đã tồn tại trả về 409 với `EMAIL_ALREADY_REGISTERED`.

## 2. Đăng nhập và nhận access token

```text
Client
→ POST /api/v1/auth/login
→ kiểm tra LoginDto
→ tìm user theo email
→ so sánh mật khẩu bằng bcrypt
→ kiểm tra tài khoản đang hoạt động
→ cấp JWT access token
```

`LoginDto` cũng chỉ chứa `email` và `password`; validation mật khẩu cho phép 1–72 ký tự. `AuthService.login()` dùng `bcrypt.compare()` để so sánh mật khẩu gửi lên với hash đã lưu.

Payload JWT được ký chứa `sub` (ID người dùng) và `role`. Response có loại token Bearer và thời hạn theo giây đã cấu hình. Cấu hình ký JWT lấy từ `ConfigService`; giá trị secret không được đưa vào client, đoạn source minh họa hoặc báo cáo.

Thông tin đăng nhập không hợp lệ hoặc tài khoản không hoạt động trả về 401 với `INVALID_CREDENTIALS`.

## 3. Gửi request đã xác thực

```http
GET /api/v1/auth/me
Authorization: Bearer <token>
```

```text
Client
→ Authorization: Bearer <token>
→ JwtAuthGuard
→ JwtStrategy kiểm tra chữ ký và thời hạn
→ tìm user đang hoạt động qua payload.sub
→ gắn người dùng đã xác thực vào request
→ controller/tầng nghiệp vụ
```

`JwtStrategy` trích Bearer token và đặt `ignoreExpiration: false`. Phương thức `validate()` tải người dùng hiện tại từ cơ sở dữ liệu, trả về `id`, `email` và vai trò hiện tại trong cơ sở dữ liệu. User đã xóa hoặc không hoạt động bị từ chối.

Phương thức nguyên bản sau lấy từ `src/auth/jwt-auth.guard.ts`; đã lược bỏ import và class bao ngoài:

```typescript
handleRequest<TUser = unknown>(err: unknown, user: TUser | false): TUser {
  if (err || !user) {
    throw new UnauthorizedException({
      code: ApiErrorCode.Unauthorized,
      message: 'Authentication is invalid or has expired.',
    });
  }
  return user;
}
```

Class bao ngoài kế thừa `AuthGuard('jwt')`. Handler `/auth/me` dùng `@UseGuards(JwtAuthGuard)` và lấy danh tính đã xác thực qua `@CurrentUser()`.

## 4. Kiểm tra hợp đồng xác thực

Dùng Swagger UI local tại `/api/docs` để kiểm tra đăng ký, đăng nhập và endpoint bảo vệ `/api/v1/auth/me` với tài khoản test riêng của bạn. Không đưa mật khẩu tài khoản hoặc token thật vào ảnh chụp.

| Tình huống | Response dự kiến |
| --- | --- |
| Thiếu JWT, JWT không hợp lệ hoặc hết hạn | 401 `UNAUTHORIZED`. |
| JWT tham chiếu user không tồn tại/không hoạt động | 401 `UNAUTHORIZED`. |
| Thông tin đăng nhập không hợp lệ hoặc tài khoản đăng nhập không hoạt động | 401 `INVALID_CREDENTIALS`. |
| DTO đăng ký/đăng nhập không hợp lệ, kể cả gửi trường role khi đăng ký | 400 lỗi validation. |

Danh tính hợp lệ nhưng không có vai trò cần thiết là lỗi phân quyền, được trình bày tại [5.3.4](../4-rbac-api-conventions/), và trả về 403.

{{< notice warning >}}
Không công khai mật khẩu plaintext, password hash, khóa ký JWT hoặc access token thật. Che dữ liệu nhạy cảm trong ảnh chụp kết quả đăng nhập.
{{< /notice >}}

<!-- TODO_SCREENSHOT: các endpoint Auth trong Swagger và response đăng nhập thành công đã che access token cùng định danh cá nhân. -->

