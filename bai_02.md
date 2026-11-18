**Bước 1: Đề xuất 2 prompt mẫu**

Viết rõ 2 prompt tương ứng với 2 phương án giải quyết.

**Prompt 1:**

```
Bạn là nhân viên xử lý đơn hàng của cửa hàng.
Bảng giá: A 100.000đ, B 50.000đ, C 30.000đ. Mã GIAM10 giảm 10% nếu tiền hàng từ 200.000đ. Phí ship 20.000đ, miễn phí nếu tiền hàng sau giảm từ 300.000đ.

Hãy suy nghĩ từng bước: trích xuất sản phẩm, số lượng, địa chỉ, SĐT từ tin nhắn; tính tiền hàng, giảm giá, phí ship, tổng thanh toán; sau đó viết tin nhắn xác nhận cho khách.

Tin nhắn khách: """{{tin_nhan}}"""
```

**Prompt 2:**

```
# VAI TRÒ
Bạn là hệ thống xử lý đơn hàng tự động của cửa hàng. Thực hiện tuần tự 3 TRẠM trong một lần trả lời, Output trạm trước là Input trạm sau.

# DỮ LIỆU CỬA HÀNG
Bảng giá: A 100.000đ | B 50.000đ | C 30.000đ
Chính sách: GIAM10 giảm 10% khi tiền hàng ≥ 200.000đ; phí ship 20.000đ, miễn phí nếu tiền hàng sau giảm ≥ 300.000đ; tổng > 5.000.000đ → need_human_review=true.

# TRẠM 1: TRÍCH XUẤT VÀ KIỂM TRA (ưu tiên từ trên xuống)
1. Sản phẩm không có trong bảng giá → status="ERROR", không đoán.
2. Thiếu địa chỉ, SĐT hoặc số lượng → status="NEEDS_INFO", liệt kê missing_fields.
3. SĐT không đúng 10 chữ số → status="NEEDS_INFO".
4. Số lượng ≤ 0 hoặc > 20 → status="ERROR".
5. Hợp lệ → status="OK". Không bịa thông tin; trường trống để "".

# TRẠM 2: TÍNH TIỀN BẰNG CHAIN-OF-THOUGHT
- Trạm 1 khác "OK" → tram_2={"status":"STOPPED"}, không tính toán.
- Nếu OK, suy luận từng bước: (1) thành tiền từng dòng; (2) subtotal; (3) kiểm tra điều kiện mã, tính discount (không hợp lệ → 0 và ghi lý do); (4) phí ship theo tiền hàng sau giảm; (5) total = subtotal − discount + ship, kiểm tra lại từng phép tính, so ngưỡng 5.000.000đ.

# TRẠM 3: SOẠN TIN PHẢN HỒI (tối đa 100 từ, thân thiện)
- NEEDS_INFO: xin khách bổ sung đúng missing_fields. ERROR: báo không hợp lệ, đề nghị chọn lại. Hợp lệ: xác nhận đơn và các khoản tiền. Chỉ dùng số liệu từ Trạm 1, 2.

# ĐỊNH DẠNG OUTPUT
Chỉ trả về MỘT JSON hợp lệ, không thêm văn bản ngoài:
{"tram_1":{"status":"","items":[{"name":"","qty":0}],"address":"","phone":"","promo_code":"","missing_fields":[],"errors":[]},"tram_2":{"status":"","reasoning_steps":[],"subtotal":0,"discount":0,"shipping_fee":0,"total":0,"need_human_review":false},"tram_3":{"reply_message":""}}

# TIN NHẮN KHÁCH
"""{{tin_nhan}}"""
```
**Bước 2: Lập bảng so sánh Trade-off**

Lập bảng so sánh tiêu chí.

<table style="min-width: 441px;">
<colgroup><col style="min-width: 25px;"><col style="width: 208px;"><col style="width: 208px;"></colgroup><tbody><tr><td colspan="1" rowspan="1"><p><strong>Tiêu chí</strong></p></td><td colspan="1" rowspan="1" colwidth="208"><p><strong>Giải pháp 1 (Đơn giản)</strong></p></td><td colspan="1" rowspan="1" colwidth="208"><p><strong>Giải pháp 2 (Chi tiết)</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>Độ dễ hiểu</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Cao</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Trung bình</p></td></tr><tr><td colspan="1" rowspan="1"><p>Tốn token</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Ít</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Nhiều hơn</p></td></tr><tr><td colspan="1" rowspan="1"><p>Chặn lỗi biên</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Trung bình</p></td><td colspan="1" rowspan="1" colwidth="208"><p>Cực tốt</p></td></tr></tbody>
</table>

| Tiêu chí | Giải pháp 1 (Đơn giản) | Giải pháp 2 (Chi tiết) |
|---|---|---|
| Độ dễ hiểu / dễ viết | Cao, viết trong vài phút | Trung bình, cần thiết kế trạm và quy tắc |
| Tốn token | Ít (khoảng 120 token input) | Nhiều hơn (khoảng 600 token input) |
| Độ trễ | Thấp | Cao hơn do output dài (có reasoning_steps) |
| Chain-of-Thought | Có nhưng không kiểm soát, AI tự quyết bước | Có 5 bước cố định, kèm kiểm tra chéo |
| Chặn lỗi biên (thiếu SĐT, sản phẩm lạ, mã sai) | Trung bình, AI dễ tự đoán hoặc bỏ sót | Cực tốt, mỗi lỗi có điều kiện xử lý riêng |
| Xử lý đứt gãy | Không có, trạm sau vẫn tính dù dữ liệu sai | Có `STOPPED`, `NEEDS_INFO`, `ERROR` |
| Định dạng output | Văn bản tự do, khó parse | JSON cố định, hệ thống tự động đọc được |
| Khả năng gỡ lỗi | Thấp, không biết sai ở đâu | Cao, mỗi khối tương ứng một trạm |
| Độ ổn định giữa các lần chạy | Thấp | Cao |
| Phù hợp | Thử nghiệm nhanh, đơn lẻ | Hệ thống tự động hóa thực tế |

**Bước 3: Lựa chọn và chạy thử**

Chọn phương án tối ưu nhất, giải thích lý do chọn và chạy thử lấy kết quả.

- Phương án được chọn: Giải pháp 2 (Chi tiết)

    - Lý do chọn:

        - Đơn hàng liên quan đến tiền, sai một phép tính hoặc tự đoán dữ liệu thiếu sẽ gây thiệt hại trực tiếp, nên độ chính xác quan trọng hơn chi phí token.
Chi phí token tăng thêm rất nhỏ so với thiệt hại do một đơn tính sai hoặc giao nhầm.
Output JSON cố định giúp hệ thống tự động đọc và đưa vào Validation Gate, điều Giải pháp 1 không làm được.
Cơ chế STOPPED/NEEDS_INFO ngăn lỗi lan truyền, bảo đảm nguyên tắc không tự suy đoán.

**Test 1:**
```
Test 1 (dữ liệu đúng): “Cho mình đặt 2 sản phẩm A và 1 sản phẩm B. Giao đến 25 đường Lê Lợi, Quận 1. SĐT 0901234567. Mình có mã GIAM10.”

Giải pháp 1, kết quả AI:

Mình đã nhận đơn: 2 sản phẩm A, 1 sản phẩm B. Tiền hàng 250.000đ, giảm 10% còn 225.000đ, cộng phí ship 20.000đ, tổng 245.000đ. Giao đến 25 đường Lê Lợi, Quận 1. Cảm ơn bạn!

Đúng, nhưng không có JSON, không có bước kiểm tra, khó đưa vào hệ thống tự động.
Giải pháp 2, kết quả AI:

json
{"tram_1":{"status":"OK","items":[{"name":"Sản phẩm A","qty":2},{"name":"Sản phẩm B","qty":1}],"address":"25 
```

**Test 2:**
```
Test 2 (bẫy dữ liệu): “Cho mình 3 sản phẩm D, giao về nhà nha. Mình có mã GIAM50.”

Giải pháp 1, kết quả AI:

Mình đã nhận đơn 3 sản phẩm D. Tiền hàng 90.000đ, phí ship 20.000đ, tổng 110.000đ. Cảm ơn bạn!

Sai: AI tự bịa giá cho sản phẩm D (không có trong bảng giá), bỏ qua việc thiếu địa chỉ và SĐT, không xử lý mã GIAM50 sai.

Giải pháp 2, kết quả AI:

json
{"tram_1":{"status":"ERROR","items":[],"address":"","phone":"","promo_code":"GIAM50","missing_fields":["address","phone"],"errors":["Sản phẩm 'D' không có trong bảng giá"]},"tram_2":{"status":"STOPPED","reasoning_steps":[],"subtotal":0,"discount":0,"shipping_fee":0,"total":0,"need_human_review":false},"tram_3":{"reply_message":"Chào bạn, rất tiếc sản phẩm D hiện không có trong bảng giá của cửa hàng. Bạn vui lòng chọn lại sản phẩm A, B hoặc C, đồng thời cho mình xin địa chỉ giao hàng và số điện thoại nhé. Cảm ơn bạn!"}}

Đúng: phát hiện sản phẩm lạ, dừng tính tiền, xin bổ sung thông tin.
```

## Kết luận so sánh

| Test case | Giải pháp 1 | Giải pháp 2 |
|---|---|---|
| Test 1 (dữ liệu đúng) | Đúng | Đúng, kèm lộ trình suy luận và JSON |
| Test 2 (dữ liệu bẫy) | **Sai** (bịa giá, bỏ sót lỗi) | **Đúng** (chặn lỗi, dừng chuỗi) |