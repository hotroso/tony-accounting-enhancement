# Tony Accounting — Kế hoạch bảo trì & nâng cấp

> File này là bộ nhớ dài hạn của dự án. Mỗi lần quay lại, ĐỌC FILE NÀY TRƯỚC để không quên nhiệm vụ và bối cảnh.

Cập nhật lần cuối: 2026-10-17 (phiên 2)

---

## 1. Bối cảnh

- Phần mềm kế toán **Tony Accounting** của Công ty TNHH Phần mềm T.O.N.Y. Nhà cung cấp **đã phá sản / không liên hệ được**. Không có mã nguồn (source code). Công ty vợ của chủ máy đang dùng thật.
- Thư mục cài đặt: `C:\Program Files (x86)\Tony Software\Tony Accounting`
- Kiến trúc:
  - Giao diện: .NET Framework 2.0 (runtime v2.0.50727), Windows Forms + DevExpress v8.2 / v11.2 (đã biên dịch, KHÔNG sửa được nếu thiếu source).
  - Framework nghiệp vụ: CSLA (`Csla.DLL`).
  - CSDL: **Firebird embedded 2.5.8** (`fbembed.dll` = WI-V2.5.8.27089).
  - **QUAN TRỌNG:** Phần lớn logic nghiệp vụ nằm trong CSDL (1044 stored procedures, 146 tables, 57 triggers — xem `Creator.cql`), KHÔNG nằm trong DLL. => Có thể cập nhật/tối ưu nghiệp vụ qua SQL (procedure/trigger/index) mà KHÔNG cần source code DLL. Đây là con đường khả thi.
  - Cơ chế update gốc của phần mềm: các file `.uql` (SQL migration) trong thư mục `Update\`, mới nhất ~2018. `Creator.cql` = script tạo schema (procedure trong đó là STUB rỗng `BEGIN SUSPEND; END`; logic thật nằm trong file DB đang chạy).

## 2. Môi trường & công cụ (đã xác minh)

- Firebird 2.5 đã cài: `C:\Program Files\Firebird\Firebird_2_5\bin\` (isql.exe, gbak.exe, gstat.exe, gfix.exe...). Tương thích ODS 2.5 với DB của phần mềm.
- Thư mục làm việc (code + git): `D:\001.Sac\firebird`
- File dữ liệu:
  - `D:\001.Sac\DULIEU_PMTONY.TDB` — bản ĐÃ tối ưu (optimized) một lần trước đó. `.TDB` = file Firebird đổi đuôi.
  - Chủ máy sẽ cung cấp lại **DATA GỐC** (chưa tối ưu) để tái hiện lỗi "Bảng cân đối chạy chậm ~30 phút".
- Git: dùng để lưu lịch sử cấu trúc DB và mọi thay đổi.

## 3. Nhiệm vụ của chủ máy (yêu cầu gốc)

1. Xem khả năng cập nhật tính năng / cập nhật theo thông tư, nghị định mới (từ 2017 → nay).
2. Tra cứu thông tư/nghị định mới trên web (KHÔNG cần dựng MCP riêng — web search built-in đã đủ).
3. **Tối ưu chức năng "Bảng cân đối"** — hiện chạy chậm ~30 phút trên data gốc. Cần xem giải pháp TRƯỚC khi sửa.

## 4. Các procedure liên quan "Bảng cân đối" (trong Creator.cql, bản stub)

- `RPT_BANG_CAN_DOI_TAI_KHOAN` (+ biến thể `_ALL`, `_ALLK`, `_C1`, `_C1K`, `_C2`, `_C2K`, `_CON`, `_CONK`)
- `INT_BANG_CAN_DOI_TAI_KHOAN_C`
- `INT_CAL_FINANCIAL_BALANCE`  <- nghi là phần tính toán nặng nhất
- Nhóm BALANCE khác: `CMD_GET_BALANCE_ACCOUNT`, `COPY_BALANCES`, `FINANCIAL_BALANCE_S_EC`, `ENTRY_BALANCE_S_RS`, `DOCUMENT_BALANCE_S_RS`...
> Lưu ý: logic THẬT phải trích từ file DB, KHÔNG dùng bản stub trong Creator.cql.

## 5. Lộ trình (giai đoạn)

### Giai đoạn 0 — An toàn (BẮT BUỘC trước mọi can thiệp)
- [ ] Phần mềm phải TẮT khi thao tác DB (Firebird embedded khóa file độc quyền).
- [ ] Backup file DB (copy) + backup logic (gbak) trước khi sửa bất cứ gì.

### Giai đoạn 1 — Trích xuất cấu trúc (✅ XONG — bản ĐÃ TỐI ƯU)
- [x] git init tại D:\001.Sac\firebird
- [x] Trích toàn bộ metadata (DDL) từ `DULIEU_PMTONY.TDB` (bản đã tối ưu) -> `schema_full.sql` (3.3 MB, 89.154 dòng).
      Nội dung đã xác minh ĐẦY ĐỦ LOGIC THẬT: 1043 procedure (phần CREATE là stub rỗng theo cơ chế isql,
      logic thật nằm ở 1043 khối ALTER PROCEDURE phía sau — 3625 DECLARE VARIABLE), 147 table, 57 trigger,
      44 index, 91 domain, 2 generator. Đã đọc tận mắt body của INT_CAL_FINANCIAL_BALANCE.
- [x] Backup nhị phân chuẩn bằng gbak: `optimized_metadata.fbk` (6.09 MB, metadata-only `-m`).
      ĐÃ TEST RESTORE thành công -> file hợp lệ, phục hồi 100%. (File .fbk KHÔNG commit git, giữ trên đĩa.)
- [x] git commit lưu cấu trúc bản đã tối ưu làm MỐC SO SÁNH.

> LƯU Ý cơ chế trích: `isql -extract` / gbak sinh ra procedure 2 lần — CREATE (stub `BEGIN SUSPEND; END`
> để phá tham chiếu vòng) rồi ALTER (body thật). Khi đọc file phải nhìn phần ALTER mới thấy logic.

### Giai đoạn 1b — Đối chiếu GỐC vs TỐI ƯU (✅ XONG)
- [x] Nhận lại data GỐC (từ backup DULIEU_PMTONY_20230618_190000.rar). Trích `schema_original.sql`.
- [x] Sắp lại lịch sử git ĐÚNG CHIỀU THỜI GIAN để diff đọc tự nhiên (gốc -> tối ưu):
      - `12ec8af` = schema bản GỐC chưa tối ưu (baseline)
      - `8cae432` = schema bản ĐÃ TỐI ƯU
      - Xem bất cứ lúc nào: `git diff 12ec8af 8cae432 -- schema_full.sql`
- [x] Phân tích XONG. Kết quả: xem mục 9.

### KẾT LUẬN diff (tóm tắt — chi tiết ở mục 9)
Để Bảng cân đối từ ~30 phút -> vài chục giây, bản tối ưu đã làm 4 nhóm việc:
1. **+13 INDEX** trên CREDIT_DEBIT, BALANCES, DOCUMENTS, PERIODS, ATOMS  <- ĐÒN QUYẾT ĐỊNH, an toàn.
2. **Viết lại ~6 procedure** tính phát sinh/số dư (INT_PS_1_TK*, INT_SD_1_TK*): bỏ EXISTS lồng,
   gộp quét SIDE 0/1 bằng SUM(IIF()), thêm chốt EXIT sớm, ép PLAN INDEX, thay CMD_GET_PERIOD_FROM_DATE
   bằng truy vấn PERIODS trực tiếp.  <- ĐỤNG LOGIC TIỀN, PHẢI KIỂM THỬ ĐỐI CHIẾU.
3. **+ bảng cache `X_BANG_CAN_DOI`** (có CREATED_AT + index) — nghi dùng cache kết quả.
4. **Đổi domain -> kiểu gốc**: DM_ID->BIGINT, DM_DATE->DATE, DM_DOUBLE->DECIMAL/NUMERIC(18,4).
   <- Lưu ý: NUMERIC(18,4) làm tròn KHÁC DOUBLE -> phải soi kỹ chênh lệch số dư.

### Giai đoạn 2 — Cập nhật thông tư/nghị định (✅ Rà soát XONG — xem CAP_NHAT_PHAP_LY.md)
- [x] Lập danh mục văn bản 2018→2026 + đối chiếu khả năng xử lý. Kết quả:
  - Phần mềm hỗ trợ CẢ TT200 (DN200) và TT133 (DNN133). Cập nhật cuối: TA_2018_05_15.uql (05/2018).
  - **Hóa đơn điện tử** (NĐ123/2020, TT78/2021, NĐ70/2025, TT32/2025): ❌ Tony KHÔNG làm được ->
    dùng phần mềm HĐĐT riêng; Tony chỉ còn vai trò sổ sách.
  - **GTGT 8%** (NĐ174/2025 đến hết 2026): ✅ ĐÃ KIỂM TRA DB -> VAT_PERCENT là trường số tự do
    (DM_DOUBLE, không CHECK, không bảng danh mục) -> gõ 8 là xong, KHÔNG cần sửa gì.
  - **Mẫu tờ khai GTGT** (TT80/2021): ⚠️ mẫu trong Tony cũ (dừng ở TT26/2015) -> kê khai thẳng HTKK/eTax.
  - **Chế độ kế toán TT200/TT133**: vẫn hiện hành -> BCTC còn dùng được, rà report .repx nếu cần.
- [ ] (TÙY chủ máy) Rà report BCTC .repx nếu phát hiện chỉ tiêu lệch mẫu hiện hành.
- [ ] CHỜ chủ máy xác nhận bối cảnh: dùng TT200 hay TT133? Đã có phần mềm HĐĐT riêng chưa?
      Kê khai bằng HTKK/eTax hay in từ Tony?

### Giai đoạn 3 — Điều tra chuyên sâu phần TỐI ƯU (✅ Phân tích tĩnh XONG — xem DIEU_TRA_TOI_UU.md)
> Chủ máy: index thì OK, nhưng thay đổi KẾT CẤU SOURCE của phần mềm TIỀN BẠC thì KHÔNG được làm hời hợt.
- [x] Soi 6/6 procedure viết lại, đối chiếu từng dòng + xác minh quan hệ khóa (DOCUMENTS->ENTRIES->
      CREDIT_DEBIT đơn trị; PERIOD id toàn cục; PERIODS không chồng kỳ). Trả lời hết các bẫy đã nêu:
      - SUM(IIF(SIDE=0/1)): đúng, mỗi dòng 1 SIDE nên không sót/không trùng.
      - Chốt sớm NOT EXISTS DOCUMENTS: tương đương tập rỗng->0 của bản gốc.
      - CMD_GET_PERIOD_FROM_DATE -> PERIODS BETWEEN: truy vấn GIỐNG HỆT; phần dư (IS_BEGINNING) không dùng.
      - Bỏ COMPANY khỏi WHERE BALANCES: an toàn vì PERIOD id toàn cục.
      - => 6/6 TƯƠNG ĐƯƠNG VỀ SỐ LIỆU (phân tích tĩnh).
- [ ] CÒN LẠI (chủ máy yêu cầu, sẽ làm khi có môi trường test): KIỂM THỬ ĐỐI CHIẾU SỐ THỰC —
      chạy cùng tham số trên bản GỐC vs TỐI ƯU, so từng dòng, PASS khi trùng 100%.
- [ ] Đo thời gian trước/sau, ghi lại.
- 3 RỦI RO VẬN HÀNH cần xử lý trong script (xem DIEU_TRA_TOI_UU.md): (1) PLAN ép cứng index ->
      tạo index trước; (2) DOUBLE->NUMERIC(18,4) soi làm tròn ngoại tệ; (3) 1 index DOCUMENTS trùng.

### Giai đoạn 4 — Script update tái lập (CHỈ sau khi Giai đoạn 3 PASS)
- [ ] Viết `update_optimize.sql` gói: CREATE INDEX + ALTER PROCEDURE + CREATE TABLE X_BANG_CAN_DOI (+ cache nếu có).
      Có phần kiểm tra tồn tại (tránh lỗi nếu chạy lại), có thứ tự đúng, có backup nhắc nhở ở đầu file.
- [ ] Viết kèm `rollback_optimize.sql` để gỡ nếu cần.
- [ ] Test áp script trên 1 bản copy data trước khi dùng thật.

## 6. Nguyên tắc an toàn
- LUÔN backup trước khi can thiệp DB.
- LUÔN thao tác khi phần mềm đã tắt.
- Mọi thay đổi SQL lưu thành script có thể revert, commit vào git.
- Không sửa DLL đã biên dịch.
- Không tự ý sửa file quyền của Kiro (~/.kiro/settings/permissions.yaml) — chủ máy tự sửa.

## 7. Nhật ký tiến độ
- 2026-10-17: Khảo sát xong kiến trúc. Xác định logic nằm trong DB (1044 SP). Tìm thấy Firebird 2.5 tools. Lập plan, git init, chuẩn bị trích cấu trúc từ bản data đã tối ưu.
- 2026-10-17 (phiên 2): Trích xuất XONG bản ĐÃ TỐI ƯU. Xác minh schema_full.sql có đủ logic thật
  (1043 proc có body ở phần ALTER). Tạo + test-restore backup gbak `optimized_metadata.fbk` -> hợp lệ.
  Cập nhật .gitignore (bỏ qua file tạm `_*` và thư mục session). Commit mốc "đã tối ưu".
  Phát hiện: Firebird server (fbserver.exe) đang chạy dạng Windows Service -> trích metadata qua
  `localhost:` ổn, nhưng nếu cần thao tác mức file (copy/gbak offline) thì phải dừng service trước.
  ĐANG CHỜ: data GỐC chưa tối ưu từ chủ máy để đối chiếu diff.
- 2026-10-17 (phiên 3): Nhận data GỐC, trích schema, sắp lịch sử git đúng chiều (12ec8af gốc -> 8cae432 tối ưu).
  Phân tích XONG diff: 13 index + 6 proc viết lại + bảng cache X_BANG_CAN_DOI + đổi domain. Chi tiết mục 9.
  CHỐT với chủ máy: làm Cập nhật thông tư/nghị định (GĐ2) TRƯỚC; rồi điều tra chuyên sâu phần tối ưu (GĐ3)
  như chuyên gia vì đụng logic tiền bạc; cuối cùng mới xuất script update (GĐ4) sau khi kiểm thử PASS.

## 8. Lệnh trích xuất CHUẨN (dùng LẠI y hệt cho bản gốc để diff đồng nhất)

Trích DDL đầy đủ (chạy khi app Tony đã tắt):
```
"C:\Program Files\Firebird\Firebird_2_5\bin\isql.exe" -user SYSDBA -password masterkey ^
  -extract "localhost:D:\001.Sac\<FILE_DATA>" -output "D:\001.Sac\firebird\schema_full.sql"
```
Backup metadata nhị phân (test restore để chắc hợp lệ):
```
"C:\Program Files\Firebird\Firebird_2_5\bin\gbak.exe" -b -m -user SYSDBA -password masterkey ^
  "localhost:D:\001.Sac\<FILE_DATA>" "D:\001.Sac\firebird\<ten>.fbk"
```
> Đăng nhập Firebird: SYSDBA / masterkey (mặc định).

## 9. Chi tiết phân tích diff GỐC → TỐI ƯU (12ec8af → 8cae432)

> Lệnh xem lại: `git diff 12ec8af 8cae432 -- schema_full.sql`

### 9.1. INDEX được thêm (13) — tác động lớn nhất, an toàn
```
ATOMS_CODE_IDX                ON ATOMS (CODE)
IDX_BAL_COMPANY_PERIOD        ON BALANCES (COMPANY, PERIOD)
IDX_BAL_PERIOD_ACC            ON BALANCES (PERIOD, ACCOUNT)
IDX_BAL_PERIOD_ACC_OBJ0       ON BALANCES (PERIOD, ACCOUNT, OBJECT0)
IDX_CD_COMPANY_ACC_OBJ0       ON CREDIT_DEBIT (COMPANY, ACCOUNT, OBJECT0)
IDX_CD_COMPANY_ACC_SIDE       ON CREDIT_DEBIT (COMPANY, ACCOUNT, SIDE)
IDX_CD_ENTRY_ACC_OBJ0         ON CREDIT_DEBIT (ENTRY, ACCOUNT, OBJECT0)
IDX_CD_ENTRY_ACC_SIDE         ON CREDIT_DEBIT (ENTRY, ACCOUNT, SIDE)
DOCUMENTS_COMPANY_IDX         ON DOCUMENTS (COMPANY, INPUT_DATE)
IDX_DOC_COMPANY_DATE          ON DOCUMENTS (COMPANY, INPUT_DATE)   -- trùng chức năng DOCUMENTS_COMPANY_IDX, xem lại
IDX_DOC_COMPANY_PERIOD        ON DOCUMENTS (COMPANY, PERIOD)
IDX_PERIODS_COMPANY_DATES     ON PERIODS (COMPANY, BEGINNING_DATE, ENDING_DATE)
X_BANG_CAN_DOI_CREATED_AT_IDX ON X_BANG_CAN_DOI (CREATED_AT)
```
> LƯU Ý: `DOCUMENTS_COMPANY_IDX` và `IDX_DOC_COMPANY_DATE` cùng (COMPANY, INPUT_DATE) -> 1 cái thừa, xem lại ở GĐ3.

### 9.2. Bảng mới `X_BANG_CAN_DOI` (nghi dùng cache kết quả Bảng cân đối)
Cột: UUID(PK), CAP, VIRTUAL_ID, PARENT_ID, ID, CODE, NAME, IS_ACCOUNT, IS_GOODS_ACCOUNT,
BEGINNING_AMOUNT/DEBIT/CREDIT, RISING_DEBIT_AMOUNT/DEBIT/CREDIT_AMOUNT/CREDIT,
ENDING_AMOUNT/DEBIT/CREDIT, CREATED_AT. -> Cần tìm proc nào GHI/ĐỌC bảng này để hiểu cơ chế cache.

### 9.3. Procedure bị VIẾT LẠI (đụng logic tiền — kiểm thử kỹ ở GĐ3)
- `INT_PS_1_TK`           — phát sinh Nợ/Có 1 tài khoản trong kỳ.
- `INT_PS_1_TK_1_DT0_DT1` — phát sinh theo đối tượng 1 (chi tiết).
- `INT_PS_1_TK_DT0`       — phát sinh theo đối tượng 0.
- `INT_SD_1_TK_DN`        — số dư 1 tài khoản đầu ngày.
- `INT_SD_1_TK_DT0_DN`    — số dư theo đối tượng 0 đầu ngày.
- `INT_SD_1_TK_1_DT0_DT1_DN` — số dư theo đối tượng 1 đầu ngày.

Kiểu thay đổi chung ở các proc trên:
- EXISTS lồng nhau (subquery tương quan)  ->  JOIN phẳng: lọc DOCUMENTS theo ngày trước (derived table) rồi JOIN CREDIT_DEBIT.
- 2 lần quét riêng SIDE=0 và SIDE=1  ->  1 lần quét, dùng SUM(IIF(SIDE=0,...)) / SUM(IIF(SIDE=1,...)).
- Thêm chốt sớm: IF NOT EXISTS(DOCUMENTS trong khoảng ngày) THEN EXIT (hoặc trả 0 rồi SUSPEND).
- Ép optimizer: thêm `PLAN (B INDEX (IDX_BAL_PERIOD_ACC[_OBJ0]))`.
- Thay gọi `CMD_GET_PERIOD_FROM_DATE(...)` bằng SELECT trực tiếp trên PERIODS ... BETWEEN BEGINNING_DATE AND ENDING_DATE.
- Đổi tham số/biến: DM_ID->BIGINT, DM_DATE->DATE, DM_DOUBLE->DECIMAL(18,4) hoặc NUMERIC(18,4).
  (Một số biến trung gian giữ DOUBLE PRECISION -> cần rà nhất quán kiểu để không lệch làm tròn.)

### 9.4. Một procedure khác (ngoài nhóm tốc độ — sửa đúng đắn join)
Đổi điều kiện JOIN từ `ATOMS.COMPANY = :P_COMPANY` / `ATOM_STRINGS.COMPANY = :P_COMPANY`
sang `... = ACCOUNTS.COMPANY` / `... = ATOMS.COMPANY` (join theo cột thay vì hằng tham số).
-> Thường vô hại/đúng hơn, nhưng vẫn ghi nhận để kiểm khi đối chiếu.

- 2026-10-17 (phiên 4): GĐ2 - rà soát pháp lý 2018->2026. Tạo CAP_NHAT_PHAP_LY.md.
  Phát hiện chính: (1) HĐĐT phải làm bằng công cụ ngoài, không nhét vào Tony được;
  (2) thuế 8% KHÔNG cần sửa DB vì VAT_PERCENT là trường số tự do; (3) tờ khai thuế nên dùng HTKK/eTax;
  (4) TT200/TT133 còn hiệu lực, BCTC còn dùng. CHỜ chủ máy xác nhận bối cảnh để chốt việc cần làm trong Tony.

- 2026-10-17 (phiên 5): GĐ3 - điều tra chuyên sâu 6 procedure tối ưu. Tạo DIEU_TRA_TOI_UU.md.
  Phương pháp: trích body 2 commit, đối chiếu từng dòng, xác minh quan hệ khóa từ schema.
  Kết luận phân tích tĩnh: 6/6 proc TƯƠNG ĐƯƠNG số liệu (không cộng trùng/sai kỳ nhờ khóa đơn trị +
  PERIOD toàn cục). Ghi 3 rủi ro vận hành cho script update. CÒN: kiểm thử số thực khi có môi trường test.
