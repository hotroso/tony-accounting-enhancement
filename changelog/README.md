# Changelog — Tối ưu Database Tony Accounting

Thư mục này chứa **script cập nhật database** và **tài liệu chứng minh** để tối ưu hiệu năng
chức năng "Bảng cân đối" (từ ~30 phút xuống còn vài chục giây), áp dụng **TRÊN BẢN GỐC**.

## Triết lý xây dựng (quan trọng)

- **Bản GỐC là nguồn chân lý DUY NHẤT.** Đây là bản đang chạy thật, cái ta tin tưởng.
  - Trích từ `DULIEU_PMTONY.TDB`, lưu trong git commit `12ec8af`.
- **Bản optimize chỉ là THAM KHẢO.** Một người (chủ máy) đã thử tối ưu trước đó; ta dùng nó để
  LẤY Ý TƯỞNG (nên thêm index nào, viết lại procedure ra sao), KHÔNG sao chép mù.
  - Lưu trong git commit `8cae432`.
- **Changelog này là bản LÀM MỚI, SẠCH.** Mỗi thay đổi được thẩm định lại TỪ BẢN GỐC:
  - Chỉ thêm index mà bản gốc CHƯA có (bỏ cái gốc đã có, bỏ cái trùng nhau trong bản tham khảo).
  - Chỉ đưa vào thứ thực sự phục vụ tối ưu (đã loại bảng mồ côi không procedure nào dùng).
  - Mỗi procedure viết lại được chứng minh tương đương XUẤT PHÁT TỪ logic bản gốc.

> ⚠️ **CẢNH BÁO** — Phần mềm liên quan TIỀN BẠC và PHÁP LÝ (thuế, BCTC).
> KHÔNG áp lên DB thật khi chưa: (1) backup đầy đủ; (2) tắt hẳn phần mềm Tony;
> (3) kiểm thử đối chiếu số liệu gốc vs sau tối ưu, PASS 100%.

## Cấu trúc thư mục

| File | Nội dung |
|---|---|
| `00-TONG-QUAN.md` | Tổng quan + danh sách thay đổi + trạng thái kiểm thử |
| `issues/ISSUE-01-them-index.md` | Chứng minh: thêm index nào, vì sao (xuất phát từ index gốc) |
| `issues/ISSUE-02-viet-lai-proc-phat-sinh.md` | Chứng minh 3 proc INT_PS_* tương đương |
| `issues/ISSUE-03-viet-lai-proc-so-du.md` | Chứng minh 3 proc INT_SD_* tương đương |
| `01-create-indexes.sql` | Script tạo index (chạy TRƯỚC) |
| `02-alter-procedures.sql` | Script cập nhật 6 procedure (chạy SAU index) |
| `99-rollback.sql` | Gỡ bỏ toàn bộ thay đổi |
| `apply.md` | Hướng dẫn áp dụng + kiểm thử theo thứ tự an toàn |

## Những gì ĐÃ LOẠI khỏi changelog (và lý do)

- `ATOMS_CODE_IDX (CODE)` — bản gốc ĐÃ CÓ `ATOMS_IDX_CODE (CODE)` (cùng cột). Không thêm.
- `IDX_DOC_COMPANY_DATE (COMPANY, INPUT_DATE)` — trùng `DOCUMENTS_COMPANY_IDX` (cùng cột). Chỉ giữ 1.
- Bảng `X_BANG_CAN_DOI` + index của nó — bảng MỒ CÔI, không procedure/trigger nào đọc/ghi
  (đã kiểm: chỉ xuất hiện ở comment + CREATE). Không đóng góp cho tối ưu "Bảng cân đối" -> loại.

## Trạng thái

- ✅ Phân tích tĩnh: 6/6 procedure tương đương số liệu với bản gốc (xem các ISSUE + `../DIEU_TRA_TOI_UU.md`).
- ⏳ Kiểm thử SỐ THỰC: **CHƯA chạy**. Bắt buộc trước khi dùng thật.
