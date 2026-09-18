---
title: "5.3.2. Prisma ORM và MySQL"
weight: 2
chapter: false
---

# 5.3.2. Prisma ORM và MySQL

## Mục tiêu và lựa chọn công nghệ

Chuẩn bị quản lý schema và truy cập cơ sở dữ liệu cho Backend local. MySQL hỗ trợ dữ liệu quan hệ có transaction, phù hợp với các quan hệ giữa người dùng, bác sĩ, lịch làm việc và lịch khám.

Prisma quản lý schema, lịch sử migration và client được sinh với kiểu dữ liệu an toàn. Các service NestJS dùng Prisma Client qua dependency injection thay vì xử lý truy cập cơ sở dữ liệu trong controller.

## 1. Kiểm tra schema và cấu hình kết nối

Schema đã kiểm chứng là `prisma/schema.prisma`. `datasource db` dùng provider `mysql` và lấy cấu hình kết nối từ môi trường. Báo cáo không cần giá trị kết nối thật.

Schema chứa tám thực thể cốt lõi đã giới thiệu tại [5.2.2. Thiết kế cơ sở dữ liệu](../../2-architecture/2-database-design/). Model Prisma dùng tên như `User` và `Patient`, còn `@@map` ánh xạ sang tên bảng như `users` và `patients`.

Đoạn schema nguyên bản sau định nghĩa client được sinh và các vai trò ứng dụng; phần schema còn lại được lược bỏ:

```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["native", "rhel-openssl-3.0.x"]
}

enum Role {
  PATIENT
  DOCTOR
  ADMIN
}
```

Các binary target native và Linux trong schema phục vụ quy trình local và đóng gói Lambda hiện có. Phần này không sửa schema và không lặp lại ERD đầy đủ.

## 2. Thực hiện quy trình phát triển

```text
Prisma schema
→ migration
→ MySQL
→ Prisma Client
→ NestJS service
```

Chuẩn bị cấu hình kết nối riêng đến cơ sở dữ liệu phát triển local. Các script sau được định nghĩa trong `package.json` của Backend:

```shell
npm run prisma:validate
npm run prisma:migrate:status
npm run prisma:generate
```

Validate kiểm tra schema. Migration status kiểm tra trạng thái cơ sở dữ liệu so với lịch sử migration. Generate sinh Prisma Client cho các service TypeScript.

Khi chủ động thay đổi schema trong bản làm việc phát triển, tạo và áp dụng migration có tên, sau đó sinh lại client:

```shell
npm run prisma:migrate:dev -- --name <descriptive-change>
npm run prisma:generate
```

Cách này tương ứng với quy trình `npx prisma migrate dev --name <descriptive-change>` trong README. Lệnh migration thay đổi cơ sở dữ liệu phát triển; chỉ dùng với đúng cơ sở dữ liệu local dự kiến.

{{< notice warning >}}
Giữ connection string thật ở chế độ riêng tư. Kiểm tra cơ sở dữ liệu đích trước khi áp dụng migration; không dùng development migration hoặc thao tác reset với cơ sở dữ liệu production.
{{< /notice >}}

## 3. Sử dụng Prisma qua NestJS

`src/database/prisma.service.ts` định nghĩa `PrismaService` dùng chung. Các phương thức sau được trích từ class này; đã lược bỏ import và khai báo class:

```typescript
async onModuleInit(): Promise<void> {
  await this.$connect();
}

async onModuleDestroy(): Promise<void> {
  await this.$disconnect();
}
```

Service kế thừa `PrismaClient` và triển khai các interface lifecycle của NestJS. Service kết nối khi module khởi tạo và ngắt kết nối khi module bị hủy. Ví dụ, `UsersService` inject `PrismaService` để tìm người dùng và tạo PATIENT cùng hồ sơ bệnh nhân tương ứng.

## 4. Giữ lịch sử migration nhất quán

Backend có `prisma/migrations/` với các migration hiện có cho nền tảng xác thực, mô hình dữ liệu cốt lõi và danh tính bệnh nhân/lý do thay đổi trạng thái. Mỗi thay đổi schema cần được theo dõi qua Prisma Migrate để môi trường local, test và AWS sau này dùng cùng lịch sử.

Script `migrate:deploy` đã kiểm chứng chạy `prisma migrate deploy` để áp dụng các migration hiện có khi triển khai. Chi tiết kết nối RDS và migration lúc triển khai thuộc phần triển khai tiếp theo.

## 5. Nhận biết seed phát triển tùy chọn

Backend khai báo `prisma:seed` và cấu hình `prisma/seed.ts`. Luồng điều khiển seed đã kiểm tra có bước kiểm tra môi trường production và yêu cầu bật seed phát triển rõ ràng qua `ALLOW_DEV_SEED`.

Seed là bước chuẩn bị dữ liệu phát triển tùy chọn, tách biệt với migration schema. Giữ thông tin đăng nhập dùng cho seed ở chế độ riêng tư.

<!-- TODO_SCREENSHOT: kết quả Prisma migration local, đã che chi tiết kết nối và thông tin đăng nhập. -->
