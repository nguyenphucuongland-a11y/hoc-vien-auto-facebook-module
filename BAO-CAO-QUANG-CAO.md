# Báo cáo quảng cáo Facebook trên Lark — bảng 14.4, 14.5 và bảng tổng hợp 14.6

Phần quảng cáo của module: mỗi sáng máy lấy **danh sách tài khoản quảng cáo → bảng 14.4** và **số liệu từng quảng cáo theo ngày → bảng 14.5**; **bảng tổng hợp 14.6** (dashboard) đọc thẳng từ 14.5 nên tự cập nhật theo.

```
Facebook Graph API (FB_USER_TOKEN có quyền ads_read)
        │  fetch-adaccounts · fetch-ads-insights (GitHub Actions trong kho của bạn)
        ▼
Lark Base ── 14.4 Ads Account list      (1 dòng / tài khoản quảng cáo)
          ├─ 14.5 Thống kê theo ngày    (1 dòng / quảng cáo × ngày)
          └─ 14.6 Báo cáo (dashboard)   ← đọc từ 14.5
Kích hoạt hằng ngày: 2 Lark Automation định kỳ → gọi GitHub (xem LARK-AUTOMATION.md)
```

## 1. Cài phần dữ liệu (14.4, 14.5)

Làm theo `TRIEN-KHAI.md` — phần quảng cáo cần thêm:

- **Token Facebook** có quyền `ads_read` (và `business_management` nếu tài khoản quảng cáo nằm trong Business Manager). Thiếu `ads_read` → workflow báo *"FB_USER_TOKEN thiếu quyền: ads_read"*.
- Chạy `init-tables` một lần → có sẵn bảng **14.4** và **14.5** đủ cột, đủ công thức.
- Biến `TABLE_ADS_ACCOUNT`, `TABLE_ADS_DAILY` để trống thì máy tự tìm bảng tên bắt đầu bằng `14.4`, `14.5`. `AD_ACCOUNT_ID` để trống = lấy **mọi** tài khoản token nhìn thấy.
- Chạy thử: `fetch-adaccounts` rồi `fetch-ads-insights` với `date_preset = maximum` (lấy toàn bộ lịch sử lần đầu).

## 2. Chạy tự động hằng ngày

Dựng 2 Lark Automation **định kỳ** gọi GitHub — thông số chi tiết trong `LARK-AUTOMATION.md`:

| Automation | Giờ gợi ý | Body gửi GitHub |
|---|---|---|
| Tài khoản quảng cáo (14.4) | 05:30 | `{"event_type":"fetch-adaccounts","client_payload":{}}` |
| Số liệu theo ngày (14.5) | 05:35 | `{"event_type":"fetch-ads-insights","client_payload":{"date_preset":"last_7d"}}` |

`last_7d` = mỗi ngày làm mới 7 ngày gần nhất (Facebook chốt số trễ vài ngày). Dòng cũ giữ nguyên — máy cập nhật theo cặp *quảng cáo × ngày*, không nhân đôi.

## 3. Dựng bảng tổng hợp 14.6 (làm tay một lần trong Lark)

Trong Base → **Dashboard** → tạo mới, đặt tên **"14.6 Báo cáo"**, nguồn dữ liệu cho mọi thẻ/biểu đồ: bảng **14.5 Facebook ads - Thống kê theo ngày**.

**5 thẻ chỉ số** (kiểu *Số liệu*, phép tính **Tổng / SUM**):

| Thẻ | Cột |
|---|---|
| Tổng chi phí | `Tổng chi phí` |
| Hiển thị | `Số lần hiển thị` |
| Nhấp | `Số lần nhấp` |
| Tin nhắn (Mess) | `Số Mess` |
| Lượt mua | `Lượt mua` |

**4 biểu đồ**:

| Biểu đồ | Kiểu | Trục / nhóm theo | Giá trị |
|---|---|---|---|
| Chi phí theo chiến dịch | Cột | `Tên Chiến Dịch` | Tổng `Tổng chi phí` |
| Chi phí theo tài khoản | Tròn hoặc cột | `Tên tài khoản` | Tổng `Tổng chi phí` |
| Chi phí theo ngày | Đường | `Ngày bắt đầu` | Tổng `Tổng chi phí` |
| Tin nhắn theo chiến dịch | Cột | `Tên Chiến Dịch` | Tổng `Số Mess` |

Muốn xem theo tháng/năm: lọc hoặc nhóm theo cột công thức `Tháng`, `Năm` có sẵn trong 14.5.

## 4. Đọc số cho đúng (bẫy đã gặp)

- **Không cộng dồn CPM, CPC, CTR, Tần Suất** — đây là chỉ số theo từng dòng. Trên dashboard chỉ cộng các đại lượng cộng được (chi phí, hiển thị, nhấp, Mess, mua). Muốn CTR/CPM trung bình thì dùng phép **Trung bình**, hoặc tính lại = tổng chi phí / tổng nhấp (hoặc / tổng hiển thị × 1000).
- **Máy không tự xoá dòng** — chỉ thêm/cập nhật. Muốn bỏ một tài khoản khỏi báo cáo: đặt `AD_ACCOUNT_ID` hoặc lọc trên dashboard, rồi xoá tay các dòng cũ của tài khoản đó.
- **Tổng chi tiêu ở 14.4 là luỹ kế toàn thời gian** của tài khoản (Facebook trả theo đơn vị nhỏ nhất của tiền tệ; VND không có số lẻ nên đúng bằng đồng).
- **Lệnh gọi GitHub từ Lark Automation** bắt buộc có header `User-Agent`, thiếu là lỗi 403; **Kiểu phản hồi chọn None** vì GitHub trả 204 rỗng.
- **Token Facebook hết hạn** (thường khoảng 60 ngày): workflow chạy đỏ, log báo token không hợp lệ → cấp token mới, cập nhật Secret `FB_USER_TOKEN`, không cần sửa code.
