# ISSUE-01 — Thêm 10 index tổ hợp (áp lên bản gốc)

**Loại:** Tối ưu hiệu năng
**Rủi ro số liệu:** THẤP — index KHÔNG thay đổi kết quả truy vấn, chỉ thay đổi tốc độ truy xuất.
**Xuất phát:** index hiện có trong BẢN GỐC (`12ec8af`).

## 1. Index hiện có trong BẢN GỐC (trên các bảng liên quan)

Trích trực tiếp từ `12ec8af:schema_full.sql`:
```
ATOMS_IDX_CODE        ON ATOMS (CODE)
ATOMS_IDX_COMPANY     ON ATOMS (COMPANY)
BALANCES_COMPANY      ON BALANCES (COMPANY)
CREDIT_DEBIT_IDX_COMPANY ON CREDIT_DEBIT (COMPANY)
CREDIT_DEBIT_SIDE     ON CREDIT_DEBIT (SIDE)
DOCUMENTS_IDX_COMPANY ON DOCUMENTS (COMPANY)
DOCUMENTS_INPUT_DATE  ON DOCUMENTS (INPUT_DATE)
ENTRIES_ENTRY_TYPE    ON ENTRIES (ENTRY_TYPE)
ENTRIES_IDX_COMPANY   ON ENTRIES (COMPANY)
PERIODS: (chỉ có PK + UNIQUE(BEGINNING_DATE,COMPANY), UNIQUE(ENDING_DATE,COMPANY) — không index thường)
```
=> Bản gốc chỉ có index ĐƠN CỘT. Các truy vấn của Bảng cân đối lọc theo NHIỀU cột
(PERIOD+ACCOUNT, ENTRY+ACCOUNT+SIDE, COMPANY+INPUT_DATE...) không có index tổ hợp phù hợp
=> Firebird phải quét rộng hoặc dùng index kém chọn lọc => chậm.

## 2. 10 index CẦN THÊM (đã loại trùng)

```sql
-- BALANCES (số dư theo kỳ)
CREATE INDEX IDX_BAL_COMPANY_PERIOD   ON BALANCES (COMPANY, PERIOD);
CREATE INDEX IDX_BAL_PERIOD_ACC       ON BALANCES (PERIOD, ACCOUNT);
CREATE INDEX IDX_BAL_PERIOD_ACC_OBJ0  ON BALANCES (PERIOD, ACCOUNT, OBJECT0);
-- CREDIT_DEBIT (phát sinh Nợ/Có)
CREATE INDEX IDX_CD_COMPANY_ACC_OBJ0  ON CREDIT_DEBIT (COMPANY, ACCOUNT, OBJECT0);
CREATE INDEX IDX_CD_COMPANY_ACC_SIDE  ON CREDIT_DEBIT (COMPANY, ACCOUNT, SIDE);
CREATE INDEX IDX_CD_ENTRY_ACC_OBJ0    ON CREDIT_DEBIT (ENTRY, ACCOUNT, OBJECT0);
CREATE INDEX IDX_CD_ENTRY_ACC_SIDE    ON CREDIT_DEBIT (ENTRY, ACCOUNT, SIDE);
-- DOCUMENTS (chứng từ)
CREATE INDEX DOCUMENTS_COMPANY_IDX    ON DOCUMENTS (COMPANY, INPUT_DATE);
CREATE INDEX IDX_DOC_COMPANY_PERIOD   ON DOCUMENTS (COMPANY, PERIOD);
-- PERIODS (kỳ kế toán)
CREATE INDEX IDX_PERIODS_COMPANY_DATES ON PERIODS (COMPANY, BEGINNING_DATE, ENDING_DATE);
```

## 3. Giải trình từng index phục vụ truy vấn nào (đối chiếu procedure sẽ viết lại)

| Index | Phục vụ |
|---|---|
| `IDX_BAL_PERIOD_ACC (PERIOD, ACCOUNT)` | INT_SD_1_TK_DN: `WHERE PERIOD=? AND ACCOUNT=?` (procedure ép PLAN index này) |
| `IDX_BAL_PERIOD_ACC_OBJ0 (PERIOD, ACCOUNT, OBJECT0)` | INT_SD_1_TK_1_DT0_DT1_DN: lọc PERIOD+ACCOUNT+OBJECT0 (ép PLAN) |
| `IDX_BAL_COMPANY_PERIOD (COMPANY, PERIOD)` | Truy vấn BALANCES theo công ty + kỳ |
| `IDX_CD_ENTRY_ACC_SIDE (ENTRY, ACCOUNT, SIDE)` | JOIN CREDIT_DEBIT theo ENTRY, lọc ACCOUNT + SIDE |
| `IDX_CD_ENTRY_ACC_OBJ0 (ENTRY, ACCOUNT, OBJECT0)` | JOIN CREDIT_DEBIT theo ENTRY, lọc ACCOUNT + OBJECT0 |
| `IDX_CD_COMPANY_ACC_SIDE / _OBJ0` | Truy vấn CREDIT_DEBIT lọc company + account (+ side/obj0) |
| `DOCUMENTS_COMPANY_IDX (COMPANY, INPUT_DATE)` | Lọc chứng từ `COMPANY=? AND INPUT_DATE BETWEEN ? AND ?` |
| `IDX_DOC_COMPANY_PERIOD (COMPANY, PERIOD)` | Lọc chứng từ theo kỳ |
| `IDX_PERIODS_COMPANY_DATES (COMPANY, BEGINNING_DATE, ENDING_DATE)` | Tìm kỳ: `COMPANY=? AND :NGAY BETWEEN BEGINNING_DATE AND ENDING_DATE` |

## 4. Những index trong bản THAM KHẢO đã bị LOẠI khỏi changelog (và lý do)

| Index tham khảo | Lý do loại |
|---|---|
| `ATOMS_CODE_IDX (CODE)` | Bản gốc ĐÃ CÓ `ATOMS_IDX_CODE (CODE)` — cùng cột. Không cần thêm. |
| `IDX_DOC_COMPANY_DATE (COMPANY, INPUT_DATE)` | Trùng `DOCUMENTS_COMPANY_IDX (COMPANY, INPUT_DATE)`. Chỉ giữ 1. |
| `X_BANG_CAN_DOI_CREATED_AT_IDX` | Index của bảng mồ côi X_BANG_CAN_DOI (loại — xem README). |

## 5. An toàn / revert
- CREATE INDEX không mất dữ liệu, không đổi kết quả tính toán.
- Revert: `DROP INDEX <ten>` (xem `99-rollback.sql`).
- Tạo index trên CREDIT_DEBIT / BALANCES (nhiều triệu dòng) có thể mất vài phút; phần mềm phải TẮT.
