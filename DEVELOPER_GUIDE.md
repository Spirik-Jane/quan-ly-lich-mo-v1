# DEVELOPER GUIDE — Phòng Mổ Thông Minh v2

> Tài liệu nội bộ, không hiển thị trong ứng dụng.

---

## Kiến trúc

```
src/
├── types.ts                   # Toàn bộ TypeScript types, enums, constants
├── App.tsx                    # Root component + MainApp + AuthGate
├── main.tsx                   # ReactDOM entry
├── contexts/
│   └── AuthContext.tsx        # Auth state (localStorage-based)
├── components/
│   ├── AuthPage.tsx           # Login / Register / ForgotPassword / OTP
│   ├── Sidebar.tsx            # Sidebar phân quyền theo role
│   ├── SurgeryModal.tsx       # Modal thêm/sửa ca mổ (tabbed)
│   └── Timeline.tsx           # Timeline Gantt hiển thị phòng mổ
└── utils/
    └── scheduler.ts           # Auto-schedule + CSV export + helpers
```

---

## Phân quyền (Roles)

| Role | Thêm ca | Xem lịch | Cập nhật tiến trình | Xếp phòng | Điều phối |
|------|---------|----------|---------------------|-----------|-----------|
| `ward_staff` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `or_staff` | ❌ | ✅ | ✅ | ❌ | ❌ |
| `manager` | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Cơ chế tự động xếp ca mổ (Auto-Schedule)

Hàm `autoSchedule()` trong `scheduler.ts` hoạt động như sau:

1. **Sắp xếp theo độ ưu tiên**: Cấp cứu → Khẩn → Phiên
2. **Chỉ xét phòng sạch** (`status === 'sach'` và `isAvailable === true`)
3. **Với mỗi ca chờ**, thử tìm slot sớm nhất trong từng phòng:
   - Lấy tất cả slot đã bị chiếm trong phòng (ca đã xếp + chặn giờ + buffer)
   - Gộp thêm slot bận của PTV chính (tránh xếp cùng lúc 2 ca cho 1 bác sĩ)
   - Tìm khoảng trống đủ dài cho ca mổ (duration phút)
4. **Chọn phòng** có thời điểm bắt đầu sớm nhất trong tất cả phòng
5. **Cập nhật** danh sách slot bận của phòng và PTV sau khi xếp
6. Ca không xếp được (không còn slot) → giữ trạng thái `pending`

**Lưu ý quan trọng cho auto-schedule:**
- Phòng **DƠ** (`status: 'do'`) và phòng **đang xông** (`dang_xong`) KHÔNG được chọn tự động
- Phòng **đóng cửa** (`isAvailable: false`) cũng không được chọn
- Phòng **sanh** chỉ hiện trong danh sách nếu phương pháp phẫu thuật có từ khóa "sanh/sinh/đẻ"

---

## Quản lý phòng dơ/sạch

```
Ca mổ hoàn thành (completed) 
   ↓
Có từ khóa phẫu thuật dơ? (trĩ, ruột thừa, hậu môn, áp xe...)
   ↓ YES                    ↓ NO
Phòng → status: 'do'       Phòng giữ nguyên
   ↓
Manager bấm "Bắt đầu xông" → status: 'dang_xong'
   ↓
Manager bấm "Xông xong — Sạch" → status: 'sach'
   ↓
Phòng sẵn sàng xếp ca mổ lại
```

Từ khóa gây dơ phòng (file `types.ts` → `DIRTY_SURGERY_KEYWORDS`):
```ts
['trĩ', 'ruột thừa', 'hậu môn', 'áp xe', 'rò hậu môn', 'nứt hậu môn', 
 'polyp hậu môn', 'cắt trĩ', 'viêm ruột thừa']
```

---

## Kết nối Google Sheets thực tế

Hiện tại dữ liệu lưu `localStorage`. Để kết nối Google Sheets:

1. Tạo Google Apps Script Webhook nhận POST từ app
2. Thêm service trong `src/services/googleSheets.ts` (đã có cấu trúc từ v1)
3. Map các field Surgery ↔ cột Sheet theo thứ tự:
   ```
   Dấu thời gian | Mã bệnh nhân | Họ Tên | Năm Sinh | Giới tính |
   Chẩn Đoán | Phương Pháp PT | PP Vô Cảm | Mức Độ | Bác Sĩ PT |
   Thời Gian DK | NV ĐK | Khoa Phòng | Ghi chú | Yêu cầu đặc biệt |
   Tiến Trình | Phòng Mổ | ĐIỀU PHỐI | Ghi Chú | PP giảm đau |
   Giờ thực tế | Vào PACU | Xuất/Hoàn tất
   ```

---

## Tích hợp OTP thực tế

Hiện OTP hiển thị trực tiếp trên UI (demo). Để production:

1. Zalo OTP: Dùng Zalo Official Account API
   - `POST https://openapi.zalo.me/v2.0/oa/message/cs`
   - Cần ZCA token + số điện thoại đã liên kết Zalo

2. SMS OTP: Dùng ESMS.vn hoặc Viettel SMS
   - `POST https://rest.esms.vn/MainService.svc/json/SendMultipleMessage_V4_post_json/`

3. Email OTP: Dùng EmailJS hoặc backend SMTP

Luồng: frontend → backend endpoint → Zalo/SMS API → user → frontend verify

---

## Tài khoản demo mặc định

| Role | SĐT | Mật khẩu |
|------|-----|---------|
| Quản lý | 0901000001 | admin123 |
| NV Phòng mổ | 0901000002 | 123456 |
| NV Trại | 0901000003 | 123456 |

---

## Deploy

```bash
npm install
npm run build
# → dist/ folder → upload lên hosting (Vercel, Netlify, cPanel...)
```

Cấu hình base URL nếu deploy ở subdirectory:
```ts
// vite.config.ts
export default { base: '/phong-mo/' }
```
