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

*(Bạn tự chạy các prompt trên AI rồi dán kết quả thật và ảnh chụp màn hình vào cột "Kết quả AI thật". Cột "Kết quả mong đợi" dùng để đối chiếu.)*

| Test | Tin nhắn khách | Kết quả mong đợi | Kết quả AI thật |
| --- | --- | --- | --- |
| 1. Đầy đủ | "Mình là Lan, 0912345678, giao 12 Nguyễn Huệ Q1. Lấy 2 sữa tươi, 1 vỉ trứng." | `hop_le`, tổng tiền 2×32.000 + 35.000 = **99.000 đ**, tin nhắn xác nhận | *(dán)* |
| 2. Thiếu thông tin | "Cho mình 3 bó rau muống nhé." | `can_bo_sung`, `van_de` gồm tên, SĐT, địa chỉ; tin nhắn xin khách bổ sung | *(dán)* |
| 3. Dữ liệu sai lệch | "Tôi tên Minh, sđt 09123, giao 5 Lê Lợi, mua -2 sữa tươi và 1 con cá hồi." | `can_bo_sung`, `van_de` gồm SĐT sai định dạng, số lượng không hợp lệ, cá hồi không có trong danh mục | *(dán)* |

## 6. Nhận xét

- Câu lệnh điều kiện ("nếu thiếu thì ghi vào van_de, không được đoán") giúp AI không bỏ sót và không bịa dữ liệu.
- CoT ở Prompt 2 giúp tính tiền đúng; các bước khác không cần CoT nên đỡ tốn token.
- Chia chuỗi giúp biết sai ở bước nào. Nhược điểm: chậm hơn và gọi AI nhiều lần hơn một prompt duy nhất.
