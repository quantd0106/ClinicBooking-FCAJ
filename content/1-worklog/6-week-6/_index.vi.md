---
title: "Tuần 6 - CloudWatch, CloudTrail và Project Clinic Booking"
weight: 6
chapter: false
pre: "<b>1.6. </b>"
---

# WORKLOG TUẦN 6

**Thời gian:** 07/09/2026 - 13/09/2026

## Mục tiêu tuần 6:

- Tìm hiểu Amazon CloudWatch và AWS CloudTrail.
- Tìm hiểu Log, Metric và Alarm phục vụ giám sát tài nguyên AWS.
- Tổng hợp các kiến thức AWS đã học trong những tuần trước.
- Bắt đầu áp dụng kiến thức vào project Clinic Appointment Booking System.
- Xác định kiến trúc Cloud phù hợp cho project.

## Các công việc cần triển khai trong tuần này:

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Nguồn tài liệu |
| --- | --- | --- | --- | --- |
| 2 - 3 | Tìm hiểu Amazon CloudWatch và AWS CloudTrail | 07/09/2026 | 08/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 4 | Thực hành theo dõi Logs, Metrics và trạng thái tài nguyên bằng Amazon CloudWatch | 09/09/2026 | 09/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 5 | Nghiên cứu CloudWatch Logs, Metrics, Alarms và cách theo dõi lỗi ứng dụng trên AWS | 10/09/2026 | 10/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |
| 6 - 7 | Tổng hợp kiến thức và bắt đầu xây dựng Clinic Appointment Booking System | 11/09/2026 | 12/09/2026 | — |
| 7 - Chủ nhật | Nghiên cứu kiến trúc triển khai Clinic Appointment Booking System trên AWS | 12/09/2026 | 13/09/2026 | [AWS Study Group](https://cloudjourney.awsstudygroup.com/) |

### Nội dung triển khai hệ thống

- Backend sử dụng Node.js, TypeScript và NestJS.
- Cơ sở dữ liệu MySQL và Prisma ORM.
- Authentication bằng JWT.
- Phân quyền PATIENT / DOCTOR / ADMIN.
- Quản lý chuyên khoa, bác sĩ và lịch làm việc.
- Xây dựng luồng đặt lịch khám.

### Kiến trúc AWS dự kiến

```text
Client
→ Amazon API Gateway
→ AWS Lambda
→ Amazon RDS for MySQL
```

Đồng thời xác định vai trò của Amazon S3, Amazon VPC, Security Groups, AWS IAM, AWS Secrets Manager và Amazon CloudWatch.

## Kết quả đạt được tuần 6:

- Hiểu được vai trò cơ bản của Amazon CloudWatch và AWS CloudTrail.
- Biết sử dụng Logs và Metrics để theo dõi hoạt động của tài nguyên AWS.
- Tổng hợp và vận dụng các kiến thức AWS đã học vào một bài toán thực tế.
- Bắt đầu xây dựng Backend của Clinic Appointment Booking System.
- Xác định được kiến trúc AWS theo hướng Serverless cho hệ thống.
- Tạo nền tảng để tiếp tục triển khai project chi tiết trong phần Workshop.
