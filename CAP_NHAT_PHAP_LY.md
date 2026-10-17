# Đối chiếu Pháp lý Kế toán - Thuế (2018 → 2026) cho Tony Accounting

> Tài liệu này rà soát các văn bản pháp luật kế toán/thuế ban hành TỪ KHI phần mềm ngừng được
> cập nhật (bản `.uql` cuối: **TA_2018_05_15** = tháng 5/2018), đối chiếu khả năng xử lý trong
> phần mềm Tony Accounting. Mục tiêu: biết cái gì CẦN cập nhật, cái nào SỬA ĐƯỢC, cái nào PHẢI
> làm bằng công cụ ngoài.
>
> Cập nhật: 2026-10-17. Nguồn: tra cứu web (KPMG, Deloitte, MUFG, chinhphu.vn, báo chí kế toán).
> Nội dung pháp lý được diễn giải lại ngắn gọn để tuân thủ giấy phép nguồn.
> **Lưu ý: đây là tổng hợp kỹ thuật, KHÔNG phải tư vấn pháp lý/thuế. Số liệu và hiệu lực cần
> kế toán của công ty xác nhận lại với cơ quan thuế.**

---

## A. Hiện trạng phần mềm (những gì ĐÃ có trong DB)

Qua khảo sát `schema_full.sql` và thư mục cài đặt:

- **Chế độ kế toán:** hỗ trợ CẢ HAI
  - **TT200/2014** (doanh nghiệp lớn): BCTC `B01-DN200`, `B02-DN200`, `B03-DN-TT200`, `B09-DN200`.
  - **TT133/2016** (DN nhỏ và vừa): `B01a/B01b-DNN133`, `B02-DNN133`, `B03-DNN-TT133`, `B09-DNN133`.
- **Tờ khai GTGT** (mẫu 01/GTGT) qua nhiều đời thông tư — có 4 bảng phiên bản trong DB:
  `TO_KHAI_GTGT_26`, `_119`, `_156`, `_28` (ứng với các TT 26/2015, 119/2014, 156/2013, 28/2011...).
  Phiên bản mới nhất phần mềm hỗ trợ: **mẫu theo TT26/2015** (bảng `_26`, có thêm cột `DATE_SELECTION_TYPE`).
- **Tích hợp HTKK:** config trỏ `HTKK130/300/320` -> phần mềm XUẤT dữ liệu sang HTKK để nộp thuế,
  KHÔNG tự nộp. => Mẫu tờ khai điện tử mới nhất do HTKK lo, không phải Tony.
- **Cập nhật cuối cùng:** 05/2018. Từ đó KHÔNG có văn bản mới nào được nạp.

---

## B. Văn bản mới 2018 → 2026 & mức ảnh hưởng

### B1. HÓA ĐƠN ĐIỆN TỬ — thay đổi LỚN NHẤT, BẮT BUỘC
- **NĐ 123/2020/NĐ-CP** + **TT 78/2021/TT-BTC**: hóa đơn điện tử bắt buộc toàn quốc (từ 1/7/2022).
- **NĐ 70/2025/NĐ-CP** (hiệu lực **1/6/2025**): sửa đổi NĐ123; mở rộng, yêu cầu HĐĐT khởi tạo từ
  máy tính tiền kết nối dữ liệu cơ quan thuế với một số nhóm; bổ sung quy định thời điểm lập, nội dung HĐ.
- **TT 32/2025/TT-BTC**: hướng dẫn thi hành về hóa đơn, chứng từ (thay các hướng dẫn cũ).
- **Ảnh hưởng Tony:** phần mềm đời 2018 KHÔNG phát hành HĐĐT theo chuẩn mới (XML, chữ ký số,
  mã cơ quan thuế, truyền nhận). => **KHÔNG sửa trong Tony được** theo kiểu vá SQL/report.
  Phải dùng **phần mềm HĐĐT riêng** (Viettel, VNPT, MISA meInvoice, FPT.eInvoice...). Tony chỉ còn
  vai trò sổ sách/hạch toán nội bộ.

### B2. THUẾ GTGT GIẢM 2% (10% -> 8%) — ảnh hưởng mẫu tờ khai & hạch toán
- Chuỗi nghị quyết/nghị định giảm 2% GTGT từ 2022, gia hạn nhiều lần.
- **NĐ 174/2025/NĐ-CP**: tiếp tục giảm 2% GTGT **đến hết 2026** (một số nhóm hàng/dịch vụ 10% -> 8%).
- **Ảnh hưởng Tony — ĐÃ KIỂM TRA DB (2026-10-17):**
  - Thuế suất GTGT trong phần mềm là trường SỐ TỰ DO: cột `VAT_PERCENT DM_DOUBLE`
    (DM_DOUBLE = DOUBLE PRECISION, KHÔNG có CHECK giới hạn).
  - KHÔNG có bảng danh mục thuế suất cố định, KHÔNG có domain khóa giá trị.
  - => **KHÔNG CẦN sửa DB.** Kế toán chỉ cần gõ `8` vào ô VAT% khi lập chứng từ/hóa đơn là hạch toán
    8% được ngay. Phần mềm vốn đã hỗ trợ mọi thuế suất (0/5/8/10...).
  - Cần kiểm tra thêm (khi dùng thực tế): report/tờ khai GTGT in từ Tony có gom đúng nhóm 8% không —
    nhưng vì nộp qua HTKK (xem B4) nên điểm này ít quan trọng.

### B3. LUẬT THUẾ GTGT 2024 (số 48/2024/QH15) — hiệu lực 1/7/2025
- Thay đổi ngưỡng, đối tượng; **siết điều kiện khấu trừ**: yêu cầu **chứng từ thanh toán KHÔNG
  dùng tiền mặt** cho cả giao dịch < 20 triệu (trước đây được tiền mặt).
- **Ảnh hưởng Tony:** chủ yếu nghiệp vụ & chứng từ, phần mềm không "chặn" được logic này;
  là vấn đề quy trình kế toán hơn là tính năng. Theo dõi, ít tác động kỹ thuật trực tiếp.

### B4. TỜ KHAI THUẾ theo TT 80/2021/TT-BTC (hiệu lực 1/1/2022)
- TT80 thay thế phần quản lý thuế trong các TT cũ (156...), đổi/chuẩn hóa nhiều **mẫu tờ khai**
  (01/GTGT và phụ lục). Phần mềm Tony dừng ở mẫu TT26/2015 -> **mẫu tờ khai trong Tony đã cũ**.
- **Ảnh hưởng Tony:** tờ khai in trực tiếp từ Tony KHÔNG còn đúng mẫu hiện hành.
  NHƯNG vì thực tế **nộp qua HTKK/eTax** (HTKK luôn cập nhật mẫu mới nhất), nên:
  => Nên **kê khai thẳng trên HTKK/eTax**, không dùng mẫu tờ khai cũ trong Tony. SỬA mẫu trong Tony
  là có thể (sửa report .repx) nhưng công sức lớn và dễ lạc hậu lại -> KHÔNG khuyến nghị.

### B5. CHẾ ĐỘ KẾ TOÁN (TT200/2014, TT133/2016)
- Tới 2026, **TT200 và TT133 VẪN là chế độ kế toán hiện hành** cho DN (chưa bị thay bằng thông tư
  kế toán DN mới). Có dự thảo sửa đổi nhưng cần xác nhận hiệu lực tại thời điểm dùng.
- **Ảnh hưởng Tony:** hệ thống tài khoản & mẫu BCTC trong phần mềm (DN200/DNN133) **về cơ bản
  còn dùng được**. Cần rà vài mẫu biểu/chỉ tiêu nhỏ nếu có sửa đổi -> SỬA report .repx được.

---

## C. Phân loại khả năng xử lý

| Nhóm | Văn bản | Trong Tony? | Hướng xử lý |
|---|---|---|---|
| Hóa đơn điện tử | NĐ123/2020, TT78/2021, NĐ70/2025, TT32/2025 | ❌ Không | Dùng phần mềm HĐĐT riêng; Tony làm sổ sách |
| GTGT 8% | NĐ174/2025 (đến hết 2026) | ✅ SẴN SÀNG | VAT% là trường số tự do -> gõ 8 là xong, KHÔNG cần sửa DB |
| Mẫu tờ khai GTGT | TT80/2021 | ⚠️ Nên bỏ | Kê khai thẳng HTKK/eTax (mẫu luôn mới) |
| Luật GTGT 2024 | 48/2024/QH15 | ➖ Quy trình | Nghiệp vụ kế toán; ít tác động phần mềm |
| Chế độ kế toán | TT200/TT133 | ✅ Còn dùng | Rà mẫu BCTC .repx nếu có sửa đổi nhỏ |

---

## D. Khuyến nghị thứ tự làm (chờ chủ máy xác nhận trọng tâm)

1. **Xác nhận bối cảnh công ty vợ:** đang dùng TT200 hay TT133? Đã có phần mềm HĐĐT riêng chưa?
   Hiện kê khai thuế bằng HTKK/eTax hay in từ Tony? Có dùng thuế suất 8% chưa?
2. ~~Bổ sung thuế suất 8%~~ -> ĐÃ KIỂM TRA: KHÔNG cần làm gì, VAT% là trường số tự do, gõ 8 là xong.
3. **Việc KHÔNG nên ôm vào Tony:** phát hành hóa đơn điện tử, cập nhật mẫu tờ khai thuế mới
   -> để công cụ chuyên dụng (HĐĐT riêng + HTKK/eTax).
4. **Rà report BCTC** TT200/TT133 nếu có chỉ tiêu nào lệch mẫu hiện hành -> sửa `.repx`.

> Mọi can thiệp DB vẫn theo nguyên tắc an toàn ở PLAN.md: tắt app, backup, script revert được, commit git.
