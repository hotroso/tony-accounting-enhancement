# ISSUE-02 — Viết lại 3 procedure PHÁT SINH (INT_PS_*)

**Loại:** Tối ưu hiệu năng (viết lại truy vấn, KHÔNG đổi công thức kế toán)
**Rủi ro số liệu:** TRUNG BÌNH — đụng vào tính phát sinh Nợ/Có. Đã chứng minh tương đương (phân tích tĩnh).
**Procedure:** `INT_PS_1_TK`, `INT_PS_1_TK_DT0`, `INT_PS_1_TK_1_DT0_DT1`

## Cấu trúc khóa làm nền chứng minh (trích từ bản gốc)

- `ENTRIES`: PK(ID); cột `DOCUMENT NOT NULL` đơn trị => **mỗi bút toán thuộc ĐÚNG 1 chứng từ**.
- `CREDIT_DEBIT`: PK(ID); cột `ENTRY NOT NULL` đơn trị => **mỗi dòng Nợ/Có thuộc ĐÚNG 1 bút toán**.
- Chuỗi quan hệ: `DOCUMENTS (1) -> (n) ENTRIES (1) -> (n) CREDIT_DEBIT`.
  => JOIN từ cha xuống con KHÔNG nhân bản dòng con => SUM không bị cộng trùng.

Đây là tiền đề mấu chốt: mọi phép "gộp JOIN" dưới đây an toàn CHÍNH VÌ quan hệ đơn trị này.

---

## 2.1. INT_PS_1_TK — phát sinh Nợ/Có 1 tài khoản trong kỳ

### BẢN GỐC làm gì (nguồn chân lý)
```sql
-- Quét CREDIT_DEBIT LẦN 1: SIDE=0 (Nợ) -> DEBIT_AMOUNT, DEBIT
SELECT SUM(AMOUNT), SUM(OFFICIAL_SUM_OF_MONEY) FROM CREDIT_DEBIT
WHERE COMPANY=:P_COMPANY AND SIDE=0 AND ACCOUNT=:P_ACCOUNT
  AND EXISTS (SELECT ID FROM ENTRIES WHERE COMPANY=:P_COMPANY AND CREDIT_DEBIT.ENTRY=ID
              AND EXISTS (SELECT ID FROM DOCUMENTS WHERE COMPANY=:P_COMPANY AND ENTRIES.DOCUMENT=ID
                          AND INPUT_DATE BETWEEN :P_TU_NGAY AND :P_DEN_NGAY))
INTO :DEBIT_AMOUNT, :DEBIT;
-- Quét CREDIT_DEBIT LẦN 2: SIDE=1 (Có) -> CREDIT_AMOUNT, CREDIT  (y hệt, đổi SIDE=1)
-- NULL -> 0 cho cả 4 biến. SUSPEND.
```
Bản chất: "lấy các dòng CREDIT_DEBIT của tài khoản P_ACCOUNT, thuộc bút toán của chứng từ có
INPUT_DATE trong khoảng ngày; tổng riêng bên Nợ (SIDE=0) và bên Có (SIDE=1)."

### CHANGELOG đổi thành
```sql
-- Chốt sớm: không có chứng từ nào trong kỳ -> tất cả = 0, thoát.
IF (NOT EXISTS(SELECT 1 FROM DOCUMENTS WHERE COMPANY=:P_COMPANY
               AND INPUT_DATE BETWEEN :P_TU_NGAY AND :P_DEN_NGAY)) THEN
    (trả 0 hết + SUSPEND)
ELSE
    -- Quét 1 LẦN, tách Nợ/Có bằng IIF:
    SELECT SUM(IIF(cd.SIDE=0, cd.AMOUNT,0)),            SUM(IIF(cd.SIDE=0, cd.OFFICIAL_SUM_OF_MONEY,0)),
           SUM(IIF(cd.SIDE=1, cd.AMOUNT,0)),            SUM(IIF(cd.SIDE=1, cd.OFFICIAL_SUM_OF_MONEY,0))
    FROM ( SELECT e.ID AS ENTRY_ID FROM DOCUMENTS d
           JOIN ENTRIES e ON e.DOCUMENT=d.ID AND e.COMPANY=d.COMPANY
           WHERE d.COMPANY=:P_COMPANY AND d.INPUT_DATE BETWEEN :P_TU_NGAY AND :P_DEN_NGAY ) de
    JOIN CREDIT_DEBIT cd ON cd.ENTRY=de.ENTRY_ID AND cd.COMPANY=:P_COMPANY AND cd.ACCOUNT=:P_ACCOUNT
    INTO :DEBIT_AMOUNT, :DEBIT, :CREDIT_AMOUNT, :CREDIT;
    -- NULL -> 0. SUSPEND.
```

### Chứng minh tương đương
1. **Cùng tập dòng CREDIT_DEBIT:** cả hai lấy dòng CD của P_ACCOUNT mà ENTRY của nó thuộc DOCUMENT
   có INPUT_DATE trong kỳ. Gốc dùng EXISTS lồng; changelog dùng JOIN cha->con. Nhờ quan hệ đơn trị
   (ENTRY->1 DOCUMENT, CD->1 ENTRY), JOIN KHÔNG nhân bản => đúng cùng tập dòng, mỗi dòng 1 lần.
2. **Tách Nợ/Có:** mỗi dòng CD chỉ có 1 giá trị SIDE. Gốc lọc SIDE=0/SIDE=1 ở 2 lần quét;
   changelog dùng `IIF(SIDE=0,...)` / `IIF(SIDE=1,...)` trên cùng tập. Mỗi dòng chỉ cộng vào ĐÚNG
   một bên => tổng Nợ, tổng Có giống hệt.
3. **Chốt sớm:** nếu không có DOCUMENT trong kỳ -> tập rỗng. Gốc: SUM rỗng = NULL -> ép 0.
   Changelog: trả 0 trực tiếp. CÙNG kết quả (0,0,0,0).
4. **NULL->0:** cả hai xử lý giống nhau.

**KẾT LUẬN: TƯƠNG ĐƯƠNG.**

---

## 2.2. INT_PS_1_TK_DT0 — phát sinh theo đối tượng 0 (GROUP BY OBJECT0)

- **GỐC:** `UNION ALL` 2 nhánh (nhánh SIDE=0 đẩy số vào cột DEBIT & 0 vào CREDIT; nhánh SIDE=1 ngược lại),
  bọc ngoài `SUM(...) GROUP BY OBJECT0`. Lọc `ACCOUNT=?`, `OBJECT0 IS NOT NULL`, chứng từ trong kỳ.
- **CHANGELOG:** 1 lần quét, `SUM(IIF(SIDE=0,...))` / `SUM(IIF(SIDE=1,...))`, `GROUP BY OBJECT0`,
  cùng điều kiện lọc.
- **Chứng minh:** biến đổi "UNION ALL rồi SUM/GROUP BY" <-> "SUM(IIF()) gộp" là tương đương đại số.
  Mỗi dòng CD xuất hiện 1 lần (quan hệ đơn trị), chỉ cộng vào đúng 1 cột theo SIDE. Cùng GROUP BY,
  cùng điều kiện. **TƯƠNG ĐƯƠNG.**

## 2.3. INT_PS_1_TK_1_DT0_DT1 — phát sinh theo đối tượng 1 (GROUP BY OBJECT1)

- Y hệt 2.2, chỉ khác: lọc `OBJECT0 = :P_OBJECT0` (thay vì IS NOT NULL) và `GROUP BY OBJECT1`.
  Cả bản gốc và changelog đều giữ đúng hai điều kiện này. **TƯƠNG ĐƯƠNG.**

---

## Lưu ý kiểu dữ liệu (áp dụng cho cả 3 proc)
- RETURNS đổi `DM_DOUBLE` -> `DECIMAL(18,4)`. Cột nguồn `AMOUNT`, `OFFICIAL_SUM_OF_MONEY` vẫn là
  `DM_DOUBLE` (DOUBLE PRECISION). Số tiền VND nguyên đồng -> DECIMAL(18,4) chứa chính xác tuyệt đối.
  Chỉ phát sinh làm tròn nếu có >4 chữ số thập phân (hiếm; ngoại tệ/đơn giá lẻ).
  => BẮT BUỘC soi ở bước kiểm thử số thực nếu DN có giao dịch ngoại tệ.

## Trạng thái
- ✅ Phân tích tĩnh: tương đương.
- ⏳ Kiểm thử số thực: chưa chạy.
