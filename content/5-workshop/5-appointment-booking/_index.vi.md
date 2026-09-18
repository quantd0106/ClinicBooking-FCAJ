---
title: "5.5. Đặt lịch khám"
weight: 5
chapter: false
---

# 5.5. Đặt lịch khám

Giai đoạn này giải thích nghiệp vụ trung tâm của hệ thống: tạo và xem lịch khám, hủy lịch, bác sĩ chuyển trạng thái, ngăn đặt trùng khi có request đồng thời và lưu lịch sử trạng thái.

Theo dõi Backend hiện có qua appointment controller, service và tầng persistence. Dùng Swagger UI local từ 5.3 cùng lịch làm việc bác sĩ từ 5.4. Các ví dụ dùng placeholder thay cho token hoặc định danh thật.

## Nội dung

1. [5.5.1. Tạo lịch khám](1-create-appointment/)
2. [5.5.2. Vòng đời lịch khám](2-appointment-lifecycle/)
3. [5.5.3. Ngăn đặt trùng lịch](3-double-booking-protection/)
4. [5.5.4. Lịch sử trạng thái lịch khám](4-status-history/)

## Quy tắc nghiệp vụ đã kiểm chứng

- Lịch khám PENDING và CONFIRMED chặn khung giờ.
- Lịch CANCELLED không chặn đặt lại; khung giờ vẫn phải đáp ứng kiểm tra lịch làm việc và thời điểm bắt đầu trong tương lai.
- Chuyển trạng thái sai vòng đời bị từ chối, kể cả với ADMIN.
- Tạo lịch khám và lịch sử trạng thái ban đầu thành công nguyên tử.
- Server kiểm soát vai trò và ownership.

