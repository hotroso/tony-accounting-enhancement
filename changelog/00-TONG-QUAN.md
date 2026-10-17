# 00 — Tổng quan changelog tối ưu (áp lên BẢN GỐC)

Changelog này được xây MỚI, thẩm định từ bản gốc (`12ec8af`). Bản optimize (`8cae432`) chỉ tham khảo.

## Danh sách thay đổi (chỉ gồm thứ thực sự cần, áp lên gốc)

| Issue | Thay đổi | Số lượng | Rủi ro số liệu |
|---|---|---|---|
| ISSUE-01 | Thêm index tổ hợp mà bản gốc CHƯA có | 10 index | THẤP (không đổi kết quả) |
| ISSUE-02 | Viết lại 3 procedure PHÁT SINH (INT_PS_*) | 3 proc | TRUNG BÌNH (đụng tính tiền) |
| ISSUE-03 | Viết lại 3 procedure SỐ DƯ (INT_SD_*) | 3 proc | TRUNG BÌNH (đụng tính tiền) |

## 10 index sẽ thêm (đã loại trùng với gốc + trùng nội bộ)

Trên BALANCES: `IDX_BAL_COMPANY_PERIOD`, `IDX_BAL_PERIOD_ACC`, `IDX_BAL_PERIOD_ACC_OBJ0`
Trên CREDIT_DEBIT: `IDX_CD_COMPANY_ACC_OBJ0`, `IDX_CD_COMPANY_ACC_SIDE`, `IDX_CD_ENTRY_ACC_OBJ0`, `IDX_CD_ENTRY_ACC_SIDE`
Trên DOCUMENTS: `DOCUMENTS_COMPANY_IDX`, `IDX_DOC_COMPANY_PERIOD`
Trên PERIODS: `IDX_PERIODS_COMPANY_DATES`

(Chi tiết đối chiếu với index gốc: xem ISSUE-01.)

## 6 procedure viết lại

PHÁT SINH (ISSUE-02): `INT_PS_1_TK`, `INT_PS_1_TK_DT0`, `INT_PS_1_TK_1_DT0_DT1`
SỐ DƯ (ISSUE-03): `INT_SD_1_TK_DN`, `INT_SD_1_TK_DT0_DN`, `INT_SD_1_TK_1_DT0_DT1_DN`

## Thứ tự áp dụng BẮT BUỘC

1. `01-create-indexes.sql` — tạo index TRƯỚC.
   (Vì các procedure viết lại có `PLAN (B INDEX (...))` ép cứng tên index -> index phải tồn tại trước,
    nếu không procedure sẽ lỗi biên dịch.)
2. `02-alter-procedures.sql` — cập nhật procedure SAU.

## Trạng thái kiểm thử số thực
**CHƯA CHẠY.** Phân tích tĩnh chứng minh tương đương trên logic. Bắt buộc đối chiếu số thực
(gốc vs sau tối ưu, cùng tham số, so từng dòng Bảng cân đối) trước khi dùng thật — xem `apply.md`.

## Quyết định còn mở (chờ chủ máy)
- Các procedure dùng `PLAN (B INDEX ...)` ép cứng. Phương án AN TOÀN HƠN: bỏ ép PLAN, để optimizer
  tự chọn index (chỉ cần thống kê tốt). Hiện changelog GIỮ ép PLAN đúng như ý tưởng bản tham khảo.
  Chủ máy quyết có bỏ ép PLAN hay không ở bước kiểm thử.
