---
title: "5.3.1. Khởi tạo dự án NestJS"
weight: 1
chapter: false
---

# 5.3.1. Khởi tạo dự án NestJS

## Mục tiêu và lựa chọn framework

Chuẩn bị Backend NestJS hiện có để phát triển tại môi trường local. NestJS kết hợp TypeScript, các module theo chức năng và dependency injection, giúp mở rộng API mà không đưa truy cập cơ sở dữ liệu và quy tắc nghiệp vụ vào controller.

Guards xử lý xác thực và phân quyền, Pipes kiểm tra dữ liệu đầu vào, còn Interceptors xử lý các tác vụ dùng chung như ghi log request. Dự án tuân theo luồng Controller → Service/tầng nghiệp vụ → Prisma.

## 1. Chuẩn bị dự án hiện có

Clone hoặc tải Backend từ nguồn dự án bạn có quyền sử dụng, sau đó mở thư mục bằng Visual Studio Code. Kiểm tra sự tồn tại của `package.json`, `src/` và `prisma/schema.prisma`. Workshop tiếp tục dự án này thay vì tạo một bộ khung NestJS khác.

Cài đặt các dependency đã khai báo trong Backend:

```shell
npm install
```

Các dependency đã kiểm chứng gồm NestJS 11, TypeScript, Prisma Client và Prisma CLI 6.12.0, `@nestjs/config`, `@nestjs/swagger`, `class-validator`, `class-transformer`, Passport JWT và bcrypt.

## 2. Chuẩn bị cấu hình môi trường

`src/app.module.ts` đăng ký `ConfigModule` toàn cục với cơ chế cache và `environmentValidationSchema`. `ConfigService` cung cấp cấu hình cho ứng dụng, kết nối cơ sở dữ liệu và thiết lập JWT.

Chuẩn bị cấu hình local riêng theo tài liệu Backend. Các dòng sau chỉ là placeholder minh họa, không phải giá trị sử dụng được hay một file cấu hình đầy đủ:

```text
DATABASE_URL=<mysql-connection-string>
JWT_SECRET=<secure-secret>
PORT=<local-port>
```

Dự án còn kiểm tra `NODE_ENV`, `JWT_EXPIRES_IN_SECONDS` và `APP_TIMEZONE`. Múi giờ lịch khám/hiển thị là `Asia/Ho_Chi_Minh`. Không đưa connection string hoặc khóa ký thật vào báo cáo.

{{< notice warning >}}
Giữ cấu hình local ở chế độ riêng tư. Không commit file .env, thông tin đăng nhập cơ sở dữ liệu, khóa ký JWT hoặc AWS access keys.
{{< /notice >}}

## 3. Hiểu bootstrap và tiền tố API

`src/main.ts` gọi `createNestApplication()` và lắng nghe trên `PORT` được cấu hình. `src/app.factory.ts` tạo ứng dụng Nest, sau đó gọi `configureApplication()` và `configureSwagger()`.

Đoạn sau được trích từ `src/bootstrap.ts`; đã lược bỏ import, hàm bao ngoài và phần đăng ký filter/interceptor:

```typescript
app.setGlobalPrefix('api/v1');
app.useGlobalPipes(
  new ValidationPipe({
    transform: true,
    whitelist: true,
    forbidNonWhitelisted: true,
    forbidUnknownValues: true,
  }),
);
```

Tiền tố API là `/api/v1`. Pipe toàn cục chuyển đổi dữ liệu DTO và từ chối các trường ngoài hợp đồng DTO đã khai báo validation. Bootstrap này cũng đăng ký `HttpExceptionFilter` và `HttpLoggingInterceptor`.

## 4. Xác định các module

| Module | Source đã kiểm chứng | Vai trò |
| --- | --- | --- |
| Auth | `src/auth/auth.module.ts` | Đăng ký, đăng nhập, JWT strategy và guard xác thực. |
| Users | `src/users/users.module.ts` | Tìm người dùng và tạo PATIENT; được AuthModule import. |
| Database | `src/database/database.module.ts` | Truy cập Prisma dùng chung và các service lưu trữ dữ liệu. |
| Common | `src/common/common.module.ts` | Hỗ trợ dùng chung; code liên quan gồm phân quyền, ownership, thời gian và xử lý lỗi. |
| Specialties | `src/specialties/specialties.module.ts` | API chuyên khoa y tế. |
| Doctors | `src/doctors/doctors.module.ts` | API tìm kiếm và hồ sơ bác sĩ. |
| Schedules | `src/schedules/schedules.module.ts` | Lịch làm việc và khung giờ còn trống của bác sĩ. |
| Appointments | `src/appointments/appointments.module.ts` | Các thao tác với lịch khám. |
| Files | `src/files/files.module.ts` | Các thao tác với file hồ sơ riêng tư. |

Các module chức năng đã tồn tại. Phần nền tảng giải thích cách chúng kết nối; các luồng xử lý tiếp theo được trình bày riêng.

## 5. Kiểm tra Backend local

Sau khi chuẩn bị cấu hình riêng và cơ sở dữ liệu theo [5.3.2](../2-prisma-mysql/), dùng các script đã kiểm chứng trong `package.json`:

```shell
npm run start:dev
```

Chạy riêng các lệnh kiểm tra sau; server phát triển tiếp tục chạy cho đến khi bạn dừng:

```shell
npm run build
npm run lint
```

`start:dev` chạy `nest start --watch`, `build` chạy `nest build`, còn `lint` chạy ESLint trên `src`, `test` và các file TypeScript của Prisma.

`src/swagger.ts` đăng ký Swagger UI tại `/api/docs`, có hỗ trợ Bearer authentication. Mở `http://localhost:<local-port>/api/docs` với port đã cấu hình cho Backend. Swagger là dịch vụ riêng với site Hugo trên port 1313.

<!-- TODO_SCREENSHOT: cấu trúc thư mục Backend và Swagger UI hiển thị các endpoint Auth. -->

