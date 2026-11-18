# Bài 1: Thiết kế giải pháp Prompting ứng dụng thực tế (Logic & Chuỗi)

## 1. Bài toán chọn

**Cửa hàng MartX nhận đơn hàng qua tin nhắn tự do của khách** (Zalo, Facebook). Nhân viên phải đọc từng tin, ghi lại tên, số điện thoại, địa chỉ, món hàng, rồi tính tiền và nhắn xác nhận. Việc này chậm và hay sai.

**Mục tiêu:** dùng prompt để tự động hóa: đọc tin nhắn, kiểm tra thông tin, tính tiền, soạn tin xác nhận.

Dùng 3 kỹ thuật của Session 4:
- **Prompt Chaining:** chia 3 bước nối tiếp (trích xuất → tính tiền → soạn tin).
- **Chain-of-Thought:** bước tính tiền cho AI suy nghĩ từng bước để không tính sai.
- **Câu lệnh điều kiện xử lý lỗi:** nếu dữ liệu trống hoặc sai thì AI không được đoán mà phải báo lại.

## 2. Bước 1: Thiết kế giải pháp (Input / Output)

### Input

| Input | Mô tả |
| --- | --- |
| Tin nhắn của khách | Văn bản tự do, có thể thiếu hoặc sai thông tin |
| Bảng giá và danh mục hàng | Sữa tươi 1L: 32.000 đ; Trứng gà (vỉ 10): 35.000 đ; Rau muống (bó): 10.000 đ |

### Output mong muốn (JSON)

```json
{
  "trang_thai": "hop_le | can_bo_sung | loi",
  "khach_hang": {"ten": "...", "sdt": "...", "dia_chi": "..."},
  "mon_hang": [{"ten": "...", "so_luong": 0, "don_gia": 0, "thanh_tien": 0}],
  "tong_tien": 0,
  "van_de": ["liệt kê trường thiếu hoặc sai"],
  "tin_nhan_xac_nhan": "..."
}
```

### Quy tắc kiểm tra dữ liệu (để chặn lỗi)

- Thiếu tên, số điện thoại hoặc địa chỉ → ghi vào `van_de`, `trang_thai = can_bo_sung`.
- Số điện thoại không đủ 10 số → báo sai định dạng.
- Số lượng không phải số dương → báo lỗi.
- Món không có trong bảng giá → báo "không có trong danh mục", không tự bịa giá.
- Không được tự đoán thông tin còn thiếu.

## 3. Luồng chuỗi prompt

```
Tin nhắn khách
     ↓
[Prompt 1] Trích xuất + kiểm tra → JSON
     ↓
Có lỗi/thiếu? → có: dừng, chuyển sang Prompt 3 (nhờ khách bổ sung)
     ↓ không
[Prompt 2] Tính tiền bằng CoT → tổng tiền
     ↓
[Prompt 3] Soạn tin nhắn xác nhận (hoặc xin bổ sung)
```

## 4. Bước 2: Prompt hoàn chỉnh

### Prompt 1: Trích xuất và kiểm tra

```
Bạn là nhân viên nhập đơn của cửa hàng MartX.
Từ tin nhắn của khách, trích xuất thông tin theo JSON dưới đây.
Chỉ trả về JSON thuần, không viết thêm chữ nào.

Bảng giá (chỉ được dùng các món này): Sữa tươi 1L = 32000; Trứng gà (vỉ 10) = 35000; Rau muống (bó) = 10000.

Quy tắc:
- Nếu thiếu tên, SĐT hoặc địa chỉ: để trống trường đó và ghi tên trường vào "van_de".
- Nếu SĐT không đủ 10 số: ghi "SĐT sai định dạng" vào "van_de".
- Nếu số lượng không phải số dương: ghi "Số lượng không hợp lệ" vào "van_de".
- Nếu món không có trong bảng giá: ghi "Món X không có trong danh mục" vào "van_de".
- Tuyệt đối không tự đoán hay bịa thông tin còn thiếu.
- Nếu "van_de" khác rỗng: "trang_thai" = "can_bo_sung", ngược lại = "hop_le".

JSON mẫu: {"trang_thai":"","khach_hang":{"ten":"","sdt":"","dia_chi":""},"mon_hang":[{"ten":"","so_luong":0}],"van_de":[]}

Tin nhắn: [dán tin nhắn của khách]
```

### Prompt 2: Tính tiền bằng CoT (chỉ chạy khi trang_thai = hop_le)

```
Dữ liệu đơn hàng: [dán JSON từ Prompt 1]
Bảng giá: Sữa tươi 1L = 32000; Trứng gà (vỉ 10) = 35000; Rau muống (bó) = 10000.

Hãy suy nghĩ từng bước:
Bước 1: Với mỗi món, tính thành tiền = số lượng x đơn giá.
Bước 2: Cộng tất cả thành tiền để ra tổng tiền.
Bước 3: Kiểm tra lại phép tính một lần nữa.
Dòng cuối cùng ghi đúng dạng: [RESULT: tổng tiền]
```

### Prompt 3: Soạn tin nhắn

```
Bạn là nhân viên CSKH của MartX. Giọng thân thiện, dưới 80 từ.
- Nếu trang_thai = "hop_le": viết tin xác nhận đơn, nêu các món và tổng tiền [RESULT].
- Nếu trang_thai = "can_bo_sung": viết tin lịch sự xin khách bổ sung đúng các mục
  trong "van_de", không nhắc tới thông tin nội bộ.
- Chỉ trả về nội dung tin nhắn.
Dữ liệu: [dán JSON và kết quả tính tiền nếu có]
```

**Xử lý đứt gãy:** nếu tin nhắn của AI bị cắt giữa chừng thì copy 3–5 từ cuối và ra lệnh "Hãy viết tiếp ngay sau cụm từ: [...]". Mỗi đơn hàng chạy ở một phiên chat mới và chỉ truyền JSON sang bước sau để không bị quên hay tốn token.

## 5. Kết quả chạy thử

Mô hình dùng: *(ghi tên AI bạn dùng, ví dụ Claude, ChatGPT)*. Cả 3 test dùng đúng các prompt ở mục 4.

| Test | Tin nhắn khách | Kết quả mong đợi | Kết quả AI thật |
| --- | --- | --- | --- |
| 1. Đầy đủ | "Mình là Lan, 0912345678, giao 12 Nguyễn Huệ Q1. Lấy 2 sữa tươi, 1 vỉ trứng." | `hop_le`, tổng tiền 2×32.000 + 35.000 = **99.000 đ**, tin nhắn xác nhận | `hop_le`, `van_de` rỗng; CoT tính 64.000 + 35.000 = **99.000 đ**, `[RESULT: 99000]`; có tin nhắn xác nhận. **Đạt** |
| 2. Thiếu thông tin | "Cho mình 3 bó rau muống nhé." | `can_bo_sung`, `van_de` gồm tên, SĐT, địa chỉ; tin nhắn xin khách bổ sung | `can_bo_sung`, `van_de`: thiếu tên, thiếu SĐT, thiếu địa chỉ; tin nhắn xin bổ sung đủ 3 mục, không tự bịa thông tin. **Đạt** |
| 3. Dữ liệu sai lệch | "Tôi tên Minh, sđt 09123, giao 5 Lê Lợi, mua -2 sữa tươi và 1 con cá hồi." | `can_bo_sung`, `van_de` gồm SĐT sai định dạng, số lượng không hợp lệ, cá hồi không có trong danh mục | `can_bo_sung`, `van_de`: SĐT sai định dạng, số lượng không hợp lệ, cá hồi không có trong danh mục; không tính tiền; tin nhắn nhờ khách kiểm tra lại. **Đạt** |

### Chi tiết kết quả từng test

**Test 1: Prompt 1 (JSON)**
```json
{"trang_thai":"hop_le","khach_hang":{"ten":"Lan","sdt":"0912345678","dia_chi":"12 Nguyễn Huệ Q1"},"mon_hang":[{"ten":"Sữa tươi 1L","so_luong":2},{"ten":"Trứng gà (vỉ 10)","so_luong":1}],"van_de":[]}
```
**Prompt 2 (CoT):** Bước 1: sữa tươi 2 × 32.000 = 64.000; trứng 1 × 35.000 = 35.000. Bước 2: tổng = 64.000 + 35.000 = 99.000. Bước 3: kiểm tra lại, đúng. `[RESULT: 99000]`

**Prompt 3 (tin nhắn):** "Chào chị Lan, MartX xác nhận đơn của chị gồm 2 sữa tươi 1L và 1 vỉ trứng gà, tổng tiền 99.000đ, giao đến 12 Nguyễn Huệ, Q1. Cảm ơn chị đã ủng hộ MartX!"

**Test 2: Prompt 1 (JSON)**
```json
{"trang_thai":"can_bo_sung","khach_hang":{"ten":"","sdt":"","dia_chi":""},"mon_hang":[{"ten":"Rau muống (bó)","so_luong":3}],"van_de":["Thiếu tên","Thiếu SĐT","Thiếu địa chỉ"]}
```
**Prompt 3:** "Chào bạn, MartX đã nhận yêu cầu mua 3 bó rau muống. Để chốt đơn, bạn vui lòng cho mình xin tên, số điện thoại và địa chỉ giao hàng nhé. Cảm ơn bạn!"

**Test 3: Prompt 1 (JSON)**
```json
{"trang_thai":"can_bo_sung","khach_hang":{"ten":"Minh","sdt":"09123","dia_chi":"5 Lê Lợi"},"mon_hang":[{"ten":"Sữa tươi 1L","so_luong":-2},{"ten":"Cá hồi","so_luong":1}],"van_de":["SĐT sai định dạng","Số lượng không hợp lệ","Món Cá hồi không có trong danh mục"]}
```
**Prompt 3:** "Chào anh Minh, MartX chưa chốt được đơn vì cần anh kiểm tra lại: số điện thoại chưa đủ 10 số, số lượng sữa tươi chưa hợp lệ và cá hồi hiện chưa có trong danh mục. Anh vui lòng gửi lại thông tin giúp MartX nhé. Cảm ơn anh!"

## 6. Nhận xét

- Câu lệnh điều kiện ("nếu thiếu thì ghi vào van_de, không được đoán") giúp AI không bỏ sót và không bịa dữ liệu.
- CoT ở Prompt 2 giúp tính tiền đúng; các bước khác không cần CoT nên đỡ tốn token.
- Chia chuỗi giúp biết sai ở bước nào. Nhược điểm: chậm hơn và gọi AI nhiều lần hơn một prompt duy nhất.
