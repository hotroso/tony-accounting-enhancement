# Giai đoạn 3 — Điều tra chuyên sâu phần TỐI ƯU (kiểm chứng logic tiền bạc)

> Mục tiêu: với MỖI procedure bị viết lại, khẳng định logic mới cho KẾT QUẢ SỐ giống hệt logic cũ,
> chỉ khác tốc độ. Không chấp nhận "nhìn có vẻ đúng" — phải lập luận dựa trên cấu trúc bảng + quan hệ khóa.
>
> Phương pháp: trích body ở 2 commit (12ec8af gốc, 8cae432 tối ưu), đối chiếu từng nhánh.
> Cấu trúc khóa đã xác minh (từ schema):
> - ENTRIES: PK=ID; DOCUMENT NOT NULL (đơn trị) => mỗi entry thuộc ĐÚNG 1 document.
> - CREDIT_DEBIT: PK=ID; ENTRY NOT NULL (đơn trị) => mỗi dòng Nợ/Có thuộc ĐÚNG 1 entry.
> - Chuỗi quan hệ: DOCUMENTS (1)->(n) ENTRIES (1)->(n) CREDIT_DEBIT. Join cha->con KHÔNG nhân bản.

Cập nhật: 2026-10-17

---

## ✅ 1. INT_PS_1_TK — phát sinh Nợ/Có 1 tài khoản trong kỳ — KẾT LUẬN: TƯƠNG ĐƯƠNG, AN TOÀN

**Gốc:** quét CREDIT_DEBIT 2 lần (SIDE=0 cho Nợ, SIDE=1 cho Có), lọc bằng EXISTS lồng 2 tầng
(CREDIT_DEBIT -> ENTRIES -> DOCUMENTS trong khoảng ngày). NULL->0.

**Tối ưu:** chốt sớm (không có DOCUMENT trong kỳ -> trả 0); ngược lại quét 1 lần, JOIN phẳng
DOCUMENTS->ENTRIES->CREDIT_DEBIT, dùng SUM(IIF(SIDE=0,...)) / SUM(IIF(SIDE=1,...)). NULL->0.

**Đối chiếu:**
- Nhờ quan hệ cha->con đơn trị (đã xác minh khóa), JOIN KHÔNG nhân bản dòng CREDIT_DEBIT.
  => Tổng theo SIDE giống hệt cách quét riêng của bản gốc. TƯƠNG ĐƯƠNG.
- Chốt sớm: tập rỗng -> bản gốc SUM ra NULL -> ép 0; bản tối ưu trả 0 trực tiếp. CÙNG kết quả.
- IIF(SIDE=0/1): mỗi dòng chỉ 1 SIDE nên tách tổng chính xác.

**Lưu ý nhỏ (không ảnh hưởng VND):**
- RETURNS đổi DM_DOUBLE -> DECIMAL(18,4). Cột nguồn AMOUNT/OFFICIAL_SUM_OF_MONEY vẫn DOUBLE.
  Số tiền VND (nguyên đồng) -> DECIMAL(18,4) chứa chính xác. Chỉ làm tròn nếu có >4 chữ số thập phân
  (hiếm; ngoại tệ/đơn giá). Với Bảng cân đối VND: KHÔNG ảnh hưởng.

**Trạng thái:** PASS phân tích tĩnh. Vẫn nên đối chiếu số thực tế ở bước kiểm thử cuối.

---

## ✅ 2. INT_PS_1_TK_DT0 — phát sinh theo đối tượng 0 — KẾT LUẬN: TƯƠNG ĐƯƠNG, AN TOÀN

**Gốc:** `UNION ALL` 2 nhánh (SIDE=0 -> cột DEBIT; SIDE=1 -> cột CREDIT), rồi GROUP BY OBJECT0.
Lọc ACCOUNT, OBJECT0 IS NOT NULL, document trong kỳ.
**Tối ưu:** 1 lần quét, SUM(IIF(SIDE=0/1...)), GROUP BY OBJECT0, cùng điều kiện lọc.
**Đối chiếu:** biến đổi "UNION ALL + SUM/GROUP BY" <-> "SUM(IIF()) gộp" là tương đương đại số kinh điển.
Mỗi dòng CD chỉ 1 SIDE nên chỉ cộng đúng 1 cột. Join cha->con không nhân bản. TƯƠNG ĐƯƠNG.

## ✅ 3. INT_PS_1_TK_1_DT0_DT1 — phát sinh theo đối tượng 1 — KẾT LUẬN: TƯƠNG ĐƯƠNG, AN TOÀN

Y hệt (2), chỉ khác: lọc OBJECT0 = :P_OBJECT0 (thay IS NOT NULL) và GROUP BY OBJECT1.
Cả 2 phiên bản giữ đúng điều kiện. TƯƠNG ĐƯƠNG.

---

## Cấu trúc khóa bổ sung (đã xác minh)
- PERIODS: PK(ID) => PERIOD id DUY NHẤT TOÀN CỤC (1 period chỉ thuộc 1 company).
  UNIQUE(BEGINNING_DATE, COMPANY) + UNIQUE(ENDING_DATE, COMPANY) => trong 1 company các kỳ KHÔNG
  chồng ngày => truy vấn `:NGAY BETWEEN BEGINNING_DATE AND ENDING_DATE` trả ĐÚNG 1 kỳ.
- BALANCES: PK(ID); PERIOD NOT NULL; UNIQUE(COMPANY, PERIOD, ACCOUNT, CURRENCY, OBJECT0/1/2).

## CMD_GET_PERIOD_FROM_DATE (bản gốc) — lõi lấy kỳ
```
SELECT ID, BEGINNING_DATE, ENDING_DATE, LOCKED FROM PERIODS
WHERE COMPANY=:P_COMPANY AND :P_WORKING_DATE BETWEEN BEGINNING_DATE AND ENDING_DATE
```
=> GIỐNG HỆT truy vấn PERIODS mà bản tối ưu inline. Phần IS_BEGINNING/ACCOUNTING_SETTINGS của proc này
KHÔNG được INT_SD_* dùng (chúng chỉ lấy ID + BEGINNING_DATE). => Thay thế TƯƠNG ĐƯƠNG.

## ✅ 4. INT_SD_1_TK_DN — số dư 1 TK đầu ngày — TƯƠNG ĐƯƠNG, AN TOÀN
- Công thức số dư (BEGINNING + phát sinh, chuẩn hóa Nợ/Có) giống hệt 2 bản.
- Lấy kỳ: thay CMD_GET_PERIOD_FROM_DATE -> PERIODS trực tiếp (tương đương, xem trên).
- Bỏ `BALANCES.COMPANY=:P_COMPANY`, chỉ `PERIOD=:PERIOD AND ACCOUNT=:P_ACCOUNT`: AN TOÀN vì PERIOD id
  toàn cục -> mọi BALANCES của period đó đã đúng company.
- Ngày ngoài mọi kỳ: cả 2 bản -> PERIOD NULL (hành vi giống nhau).

## ✅ 5. INT_SD_1_TK_DT0_DN — số dư theo ĐT0 — TƯƠNG ĐƯƠNG, AN TOÀN
- 2 nhánh (ngày đầu kỳ / không) giống hệt bản gốc; UNION ALL (BALANCES + INT_PS_1_TK_DT0), GROUP BY OBJECT0.
- Phần DEBIT=DEBIT-CREDIT, chuẩn hóa dấu, điều kiện SUSPEND (lọc dòng 0 trừ TK hàng hóa) GIỐNG TỪNG DÒNG.
- Khác biệt chỉ là lấy kỳ + bỏ COMPANY + PLAN INDEX (đã chứng minh an toàn ở #4).

## ✅ 6. INT_SD_1_TK_1_DT0_DT1_DN — số dư theo ĐT1 — TƯƠNG ĐƯƠNG, AN TOÀN
- Cùng khuôn #5, thêm 1 cấp: lọc OBJECT0=:P_OBJECT0 AND OBJECT1 IS NOT NULL, GROUP BY OBJECT1,
  gọi INT_PS_1_TK_1_DT0_DT1, ép PLAN IDX_BAL_PERIOD_ACC_OBJ0. Logic Nợ/Có + SUSPEND giống hệt.

---

## KẾT LUẬN TỔNG (phân tích tĩnh)
- **6/6 procedure viết lại: TƯƠNG ĐƯƠNG về số liệu với bản gốc.** Khác biệt chỉ là cách diễn đạt SQL
  nhanh hơn (gộp quét, JOIN phẳng, chốt sớm, ép PLAN), KHÔNG đổi công thức kế toán.
- Biến đổi dựa trên quan hệ khóa đã xác minh (DOCUMENTS->ENTRIES->CREDIT_DEBIT đơn trị; PERIOD toàn cục)
  nên KHÔNG có cộng trùng / sai kỳ.

## ⚠️ RỦI RO VẬN HÀNH (không phải sai số, nhưng bắt buộc xử lý trong script update)
1. **PLAN (B INDEX (...)) ép cứng tên index.** Nếu index chưa tồn tại / đổi tên -> procedure LỖI.
   => Script update PHẢI tạo index TRƯỚC, rồi mới ALTER procedure. Cân nhắc: có thể bỏ ép PLAN để
      optimizer tự chọn (an toàn hơn, chỉ chậm hơn chút nếu thống kê tốt) — bàn với chủ máy.
2. **DOUBLE -> NUMERIC(18,4)/DECIMAL(18,4)** ở RETURNS. Với VND nguyên đồng: chính xác tuyệt đối.
   Rủi ro làm tròn chỉ khi có >4 chữ số thập phân (ngoại tệ/đơn giá lẻ). Cần soi ở KIỂM THỬ số thực.
3. Index trùng: DOCUMENTS_COMPANY_IDX và IDX_DOC_COMPANY_DATE cùng (COMPANY, INPUT_DATE) -> bỏ 1.

## BƯỚC KIỂM THỬ SỐ THỰC (BẮT BUỘC trước khi xuất script dùng thật)
- Chạy Bảng cân đối + các sổ gọi 6 proc trên, cùng tham số, trên bản GỐC và bản TỐI ƯU.
- So khớp TỪNG DÒNG. Chỉ PASS khi trùng khớp 100% (đặc biệt chú ý tài khoản có OBJECT, ngoại tệ).
