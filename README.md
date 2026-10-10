# VibeCare Hospital Management System

## Mở demo trực tuyến

**[Bấm vào đây để mở phần mềm VibeCare HMS](https://vibecare-hms-lam-2026.tatm0967005.chatgpt.site/)**

Tại màn hình đăng nhập, chọn vai trò **Bệnh nhân** hoặc **Bác sĩ**, sau đó dùng tài khoản mẫu ở bảng bên dưới. Demo chạy trực tiếp trên trình duyệt, không cần tải mã nguồn hay mở CMD.

## 1. Mục tiêu demo

VibeCare HMS là phần mềm web minh họa quy trình quản lý bệnh viện theo tài liệu SRS và các sơ đồ UML của nhóm VibeCoding. Hệ thống phân luồng rõ ràng ngay từ đăng nhập:

- Bệnh nhân: đặt lịch, xác thực BHYT, thanh toán, check-in, theo dõi hàng đợi, xem hồ sơ khám, đơn thuốc và hóa đơn.
- Bác sĩ: xem lịch khám, gọi bệnh nhân, đọc tiền sử, nhập chẩn đoán, kê đơn, hoàn tất khám và tạo lịch tái khám tự động.

Toàn bộ dữ liệu nghiệp vụ được lưu trong Cloudflare D1, một cơ sở dữ liệu quan hệ tương thích SQLite. Giao diện dùng React 19, TypeScript, Vinext và Tailwind CSS.

## 2. Tài khoản demo

| Vai trò | Email | Mật khẩu |
| --- | --- | --- |
| Bệnh nhân | patient@hms.vn | Patient@123 |
| Bác sĩ | doctor@hms.vn | Doctor@123 |

Màn hình đăng nhập tự điền đúng tài khoản khi chọn vai trò để thuyết trình nhanh.

## 3. Kiến trúc mã nguồn

```text
hospital-management-demo/
├── app/
│   ├── api/hms/route.ts       # Controller + service nghiệp vụ + RBAC
│   ├── globals.css            # Theme và responsive styling
│   ├── layout.tsx             # Metadata, ngôn ngữ giao diện
│   └── page.tsx               # Giao diện bệnh nhân và bác sĩ
├── db/
│   ├── index.ts               # Kết nối D1 bằng Drizzle
│   └── schema.ts              # Mô hình dữ liệu quan hệ
├── drizzle/                   # Migration SQL do Drizzle sinh ra
├── lib/hms-types.ts           # Kiểu dữ liệu dùng chung frontend
└── .openai/hosting.json       # Khai báo D1 binding tên DB
```

Luồng xử lý chuẩn:

```text
React UI -> /api/hms -> kiểm tra session và role -> service nghiệp vụ
         -> câu lệnh SQL D1 -> audit log -> JSON response -> cập nhật UI
```

## 4. Giải thích phần code quan trọng

### Phân quyền đăng nhập

`POST /api/hms` với action `login` nhận email, mật khẩu và vai trò. Backend chỉ tạo session khi cả ba giá trị khớp. Các action tiếp theo đọc Bearer token, lấy user hiện tại rồi kiểm tra role trước khi thao tác dữ liệu.

Ví dụ:

```ts
if (session.role !== "DOCTOR") {
  return fail("Chỉ bác sĩ phụ trách được hoàn tất khám.", 403);
}
```

Đây là RBAC ở backend. Việc ẩn menu trên giao diện chỉ giúp UX, không thay thế kiểm tra quyền ở máy chủ.

### Vòng đời lịch hẹn

Các trạng thái bám theo state diagram:

```text
INITIATED
  -> INSURANCE_VERIFIED
  -> CONFIRMED
  -> IN_CONSULTATION
  -> COMPLETED

Bất kỳ lịch chưa khám nào có thể chuyển sang CANCELLED.
```

Trong demo, bước khởi tạo và xác thực BHYT được xử lý trong cùng một transaction đặt lịch. Lịch được lưu ở `CONFIRMED` sau khi khung giờ hợp lệ; hai cột `insurance_status` và `insurance_coverage` vẫn lưu bằng chứng xác thực.

### Xác thực BHYT và tính phí

Mã BHYT hợp lệ trong demo có dạng hai chữ cái và mười ba chữ số. Phí cần trả được tính:

```text
feePayable = consultationFee × (100 - insuranceCoverage) / 100
```

Ví dụ phí khám 350.000 VND, BHYT 65%, bệnh nhân thanh toán 122.500 VND. Thuốc phát sinh sau khám được cộng riêng vào hóa đơn.

### Hoàn tất khám

Action `completeConsultation` thực hiện một nhóm thao tác đồng bộ:

1. Tạo `medical_records` chứa chẩn đoán và ghi chú.
2. Tạo `prescriptions` và từng `prescription_items`.
3. Cập nhật lịch sang `COMPLETED`.
4. Cộng tiền thuốc vào hóa đơn.
5. Nếu bác sĩ chọn ngày tái khám, tự tạo một lịch mới lúc 09:00.
6. Gửi thông báo cho bệnh nhân và ghi `audit_logs`.

### Dữ liệu mẫu

Hàm `ensureDemoSeed()` chỉ thêm dữ liệu khi bảng `users` đang rỗng. Dữ liệu có sẵn gồm ba bệnh nhân, bốn bác sĩ, lịch hôm nay, lịch tương lai, một hồ sơ cũ, đơn thuốc, hóa đơn và thông báo. Vì vậy có thể demo ngay mà không nhập dữ liệu ban đầu.

## 5. Thiết kế cơ sở dữ liệu

| Bảng | Vai trò |
| --- | --- |
| users | Tài khoản, vai trò và thông tin đăng nhập |
| patients | Hồ sơ bệnh nhân và BHYT |
| doctors | Hồ sơ bác sĩ, chuyên khoa, phòng và phí khám |
| appointments | Lịch hẹn, trạng thái, hàng đợi và thanh toán |
| medical_records | Bệnh án được tạo sau buổi khám |
| prescriptions | Đơn thuốc thuộc một bệnh án |
| prescription_items | Các thuốc, liều dùng, tần suất và giá |
| invoices | Phí khám, thuốc, BHYT và trạng thái thanh toán |
| notifications | Thông báo lịch khám, hàng đợi và đơn thuốc |
| sessions | Phiên đăng nhập có thời hạn tám giờ |
| audit_logs | Nhật ký bất biến ở tầng ứng dụng cho thao tác quan trọng |

Quan hệ chính:

```text
User 1--1 Patient 1--* Appointment *--1 Doctor
Patient 1--* MedicalRecord 1--1 Prescription 1--* PrescriptionItem
Appointment 1--1 Invoice
```

## 6. Chạy cục bộ

```powershell
npm install
npm run db:generate
npm run build
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_*.sql
npm run dev
```

Sau đó mở địa chỉ local do terminal hiển thị. Nếu cơ sở dữ liệu đã được tạo, không chạy lại câu lệnh `d1 execute` cho migration cũ.

## 7. Lưu ý kỹ thuật

- Dữ liệu và thanh toán trong dự án là mô phỏng phục vụ học tập, không dùng cho chăm sóc y tế thật.
- SHA-256 có salt giúp demo không lưu mật khẩu dạng rõ. Hệ thống thật cần Argon2id hoặc bcrypt, MFA cho bác sĩ, TLS 1.3 và quản lý khóa bí mật chuyên dụng.
- Endpoint luôn kiểm tra role và quyền sở hữu bản ghi. Giao diện không được xem là ranh giới bảo mật.
- Các comment trong `app/api/hms/route.ts`, `db/schema.ts` và `app/page.tsx` giải thích nơi ánh xạ SRS, UML và quy tắc nghiệp vụ.
