# ISSUE-03 — Viết lại 3 procedure SỐ DƯ (INT_SD_*)

**Loại:** Tối ưu hiệu năng (viết lại truy vấn, KHÔNG đổi công thức kế toán)
**Rủi ro số liệu:** TRUNG BÌNH — đụng vào tính số dư tài khoản. Đã chứng minh tương đương (phân tích tĩnh).
**Procedure:** `INT_SD_1_TK_DN`, `INT_SD_1_TK_DT0_DN`, `INT_SD_1_TK_1_DT0_DT1_DN`

## Cấu trúc khóa làm nền chứng minh (trích từ bản gốc)

- `PERIODS`: PK(ID) => **PERIOD id là duy nhất TOÀN CỤC** (một kỳ chỉ thuộc đúng 1 company).
  Ràng buộc `UNIQUE(BEGINNING_DATE, COMPANY)` + `UNIQUE(ENDING_DATE, COMPANY)` => trong 1 company
  các kỳ KHÔNG chồng ngày => `:NGAY BETWEEN BEGINNING_DATE AND ENDING_DATE` trả ĐÚNG 1 kỳ.
- `BALANCES`: PK(ID); `PERIOD NOT NULL`; `UNIQUE(COMPANY, PERIOD, ACCOUNT, CURRENCY, OBJECT0,1,2)`.

## Thay đổi CHUNG của cả 3 proc (so với bản gốc)

### (a) Lấy kỳ kế toán
- **GỐC:** `SELECT ID, BEGINNING_DATE FROM CMD_GET_PERIOD_FROM_DATE(:P_COMPANY, :P_NGAY)`
- **CHANGELOG:** truy vấn PERIODS trực tiếp:
  ```sql
  SELECT p.ID, p.BEGINNING_DATE FROM PERIODS p
  WHERE p.COMPANY=:P_COMPANY AND :P_NGAY BETWEEN p.BEGINNING_DATE AND p.ENDING_DATE
  ```
- **Chứng minh tương đương:** lõi của `CMD_GET_PERIOD_FROM_DATE` (bản gốc) CHÍNH LÀ:
  ```sql
  SELECT ID, BEGINNING_DATE, ENDING_DATE, LOCKED FROM PERIODS
  WHERE COMPANY=:P_COMPANY AND :P_WORKING_DATE BETWEEN BEGINNING_DATE AND ENDING_DATE
  ```
  -> GIỐNG HỆT. Phần dư của proc đó (tính IS_BEGINNING từ ACCOUNTING_SETTINGS) KHÔNG được các
  INT_SD_* sử dụng (chúng chỉ lấy ID + BEGINNING_DATE). => inline hoá là tương đương.
- **Ngày ngoài mọi kỳ:** cả hai -> PERIOD = NULL (singleton SELECT không có dòng). Hành vi giống nhau.

### (b) Bỏ điều kiện COMPANY khi quét BALANCES
- **GỐC:** `WHERE BALANCES.COMPANY=:P_COMPANY AND PERIOD=:PERIOD AND ACCOUNT=:P_ACCOUNT ...`
- **CHANGELOG:** `WHERE b.PERIOD=:PERIOD AND b.ACCOUNT=:P_ACCOUNT ...` (bỏ COMPANY) + `PLAN (B INDEX ...)`.
- **Chứng minh an toàn:** PERIOD id DUY NHẤT TOÀN CỤC (PK). Mọi dòng BALANCES có `PERIOD=:PERIOD`
  đương nhiên thuộc đúng company đã chọn (vì kỳ đó của company đó). => Bỏ lọc COMPANY KHÔNG đổi tập dòng.

### (c) Công thức tính số dư — GIỮ NGUYÊN
Phần tính `AMOUNT = BEGINNING + phát sinh`, `DEBIT = ... - ...`, chuẩn hoá dấu Nợ/Có
(`IF DEBIT>0 THEN CREDIT=0 ELSE CREDIT=ABS(DEBIT),DEBIT=0`), điều kiện SUSPEND — **giống hệt từng dòng**
giữa bản gốc và changelog. KHÔNG đụng công thức.

---

## 3.1. INT_SD_1_TK_DN — số dư 1 tài khoản đầu ngày
- Lấy số dư đầu kỳ từ BALANCES (SUM AMOUNT/OFFICIAL_DEBIT/OFFICIAL_CREDIT), cộng phát sinh từ
  `INT_PS_1_TK(... BEGINNING_DATE .. NGAY-1)` nếu NGAY không phải đầu kỳ. Hai bản giống hệt ngoài (a)(b).
- **TƯƠNG ĐƯƠNG.**

## 3.2. INT_SD_1_TK_DT0_DN — số dư theo đối tượng 0
- 2 nhánh: ngày = đầu kỳ (lấy thẳng BALANCES, GROUP BY OBJECT0) / ngược lại (`UNION ALL` BALANCES +
  `INT_PS_1_TK_DT0`, GROUP BY OBJECT0). Phần xử lý Nợ/Có và điều kiện SUSPEND giống hệt bản gốc.
- Khác biệt chỉ là (a)(b) + ép PLAN `IDX_BAL_PERIOD_ACC`. **TƯƠNG ĐƯƠNG.**

## 3.3. INT_SD_1_TK_1_DT0_DT1_DN — số dư theo đối tượng 1
- Cùng khuôn 3.2, thêm 1 cấp: lọc `OBJECT0=:P_OBJECT0 AND OBJECT1 IS NOT NULL`, GROUP BY OBJECT1,
  gọi `INT_PS_1_TK_1_DT0_DT1`, ép PLAN `IDX_BAL_PERIOD_ACC_OBJ0`. **TƯƠNG ĐƯƠNG.**

---

## ⚠️ Rủi ro vận hành (bắt buộc xử lý)
1. `PLAN (B INDEX (IDX_BAL_PERIOD_ACC))` và `PLAN (B INDEX (IDX_BAL_PERIOD_ACC_OBJ0))` ép CỨNG tên index
   => index phải tồn tại TRƯỚC khi ALTER procedure, nếu không procedure LỖI.
   => Thứ tự: chạy `01-create-indexes.sql` TRƯỚC `02-alter-procedures.sql`.
   => Phương án an toàn hơn (chủ máy cân nhắc): BỎ ép PLAN, để optimizer tự chọn index.
2. RETURNS đổi `DM_DOUBLE` -> `NUMERIC(18,4)`. VND nguyên đồng: chính xác. Soi làm tròn nếu có ngoại tệ.

## Trạng thái
- ✅ Phân tích tĩnh: 3/3 tương đương.
- ⏳ Kiểm thử số thực: chưa chạy.
