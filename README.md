# 🏦 BTVN03 – Thiết Kế & Cài Đặt CSDL Quản Lý Cầm Đồ

Họ và tên: Phan Văn Hải

MSSV: K235480106020

> **Môn học:** Hệ Quản Trị Cơ Sở Dữ Liệu

> **Nền tảng:** SQL Server  


## 1. Mô Tả Bài Toán

Hệ thống quản lý **cửa hàng cầm đồ** cần theo dõi:

| Yếu tố | Chi tiết |
|--------|----------|
| Khách hàng | Thông tin cá nhân, lịch sử vay |
| Hợp đồng | Số tiền vay, 2 mốc deadline, trạng thái |
| Tài sản thế chấp | Loại tài sản, giá trị định giá, trạng thái |
| Lãi suất | Lãi đơn trước Deadline1, lãi kép từ sau Deadline1 |
| Thanh toán | Trả từng phần, log dòng tiền, điều kiện lấy lại tài sản |

### Cơ chế lãi suất

```
Lãi đơn  : 0.5% / ngày trên dư nợ gốc, tính đến Deadline1
Lãi kép  : Tính trên (Gốc + Lãi đơn tích lũy) kể từ sau Deadline1
          Công thức: GốcMới × ((1 + 0.005)^số_ngày - 1)
```

---

## 2. Thiết Kế ERD & Các Bảng Dữ Liệu

### 2.1 Sơ đồ ERD 



**Quan hệ chi tiết:**
- `KhachHang` 1–N `HopDong`: Một khách hàng có thể có nhiều hợp đồng.
- `HopDong` 1–N `TaiSan`: Một hợp đồng gồm nhiều tài sản thế chấp.
- `HopDong` 1–N `ThanhToan_Log`: Một hợp đồng có nhiều lần trả tiền.
- `TaiSan` 1–N `TraTaiSan_Log`: Ghi nhận tài sản nào được trả lại.

### 2.2 Mô tả các bảng (chuẩn 3NF)

| Bảng | Mô tả |
|------|-------|
| `KhachHang` | Lưu thông tin khách hàng |
| `HopDong` | Hợp đồng cầm cố, gồm số tiền vay, 2 deadline, trạng thái |
| `TaiSan` | Tài sản thế chấp, liên kết với HopDong |
| `ThanhToan_Log` | Log từng lần khách trả tiền |
| `TraTaiSan_Log` | Log từng lần tài sản được trả lại khách |
| `NhanVien` | Nhân viên thu tiền (audit) |

> 📸 **[Chèn ảnh ERD diagram tại đây]**  
> *(Ảnh chụp màn hình sơ đồ ERD từ dbdiagram.io / SQL Server Management Studio)*

---

## 3. Script Tạo CSDL & Thêm Dữ Liệu Mẫu

### 3.1 Tạo Database & Các Bảng

**Phân tích:** Khởi tạo toàn bộ schema theo chuẩn 3NF. Mỗi bảng đều có khóa chính `INT IDENTITY`, ràng buộc `NOT NULL` cho các trường bắt buộc, và `FOREIGN KEY` để đảm bảo toàn vẹn dữ liệu. Sử dụng `CHECK` constraint để giới hạn miền giá trị trạng thái.

```sql
-- ============================================================
--  TẠO DATABASE
-- ============================================================
CREATE DATABASE QuanLyCamDo;
GO

USE QuanLyCamDo;
GO

-- ============================================================
--  BẢNG: NhanVien
--  Lưu thông tin nhân viên thu tiền (dùng cho audit log)
-- ============================================================
CREATE TABLE NhanVien (
    NhanVienID   INT           IDENTITY(1,1) PRIMARY KEY,
    HoTen        NVARCHAR(100) NOT NULL,
    SoDienThoai  VARCHAR(15),
    VaiTro       NVARCHAR(50)  DEFAULT N'Thu ngân'   -- Ví dụ: Thu ngân, Quản lý
);
GO

-- ============================================================
--  BẢNG: KhachHang
--  Lưu thông tin khách hàng đến cầm đồ
-- ============================================================
CREATE TABLE KhachHang (
    KhachHangID  INT           IDENTITY(1,1) PRIMARY KEY,
    HoTen        NVARCHAR(100) NOT NULL,
    SoDienThoai  VARCHAR(15)   NOT NULL UNIQUE,       -- Số ĐT là định danh duy nhất
    CCCD         VARCHAR(20)   UNIQUE,                -- Căn cước công dân
    DiaChi       NVARCHAR(255),
    NgayTao      DATETIME      DEFAULT GETDATE()
);
GO

-- ============================================================
--  BẢNG: HopDong
--  Mỗi hợp đồng = 1 lần vay tiền của 1 khách hàng
--  Trạng thái: 'Đang vay' | 'Đang trả góp' | 'Quá hạn'
--              | 'Đã thanh toán đủ' | 'Đã thanh lý'
-- ============================================================
CREATE TABLE HopDong (
    HopDongID     INT            IDENTITY(1,1) PRIMARY KEY,
    KhachHangID   INT            NOT NULL,
    SoTienVay     DECIMAL(18,2)  NOT NULL,             -- Tiền gốc ban đầu
    SoTienConLai  DECIMAL(18,2)  NOT NULL,             -- Dư nợ gốc còn lại (sau các lần trả)
    NgayVay       DATE           NOT NULL DEFAULT CAST(GETDATE() AS DATE),
    Deadline1     DATE           NOT NULL,             -- Mốc chuyển sang lãi kép
    Deadline2     DATE           NOT NULL,             -- Mốc thanh lý tài sản
    TrangThai     NVARCHAR(50)   NOT NULL DEFAULT N'Đang vay',
    GhiChu        NVARCHAR(500),
    CONSTRAINT FK_HopDong_KhachHang FOREIGN KEY (KhachHangID) REFERENCES KhachHang(KhachHangID),
    -- Ràng buộc: Deadline1 phải trước Deadline2
    CONSTRAINT CK_Deadline        CHECK (Deadline2 > Deadline1),
    -- Ràng buộc: Tiền vay phải dương
    CONSTRAINT CK_SoTienVay       CHECK (SoTienVay > 0),
    CONSTRAINT CK_SoTienConLai    CHECK (SoTienConLai >= 0),
    CONSTRAINT CK_TrangThai_HD    CHECK (TrangThai IN (
        N'Đang vay', N'Đang trả góp', N'Quá hạn', N'Đã thanh toán đủ', N'Đã thanh lý'
    ))
);
GO

-- ============================================================
--  BẢNG: TaiSan
--  Mỗi tài sản gắn với 1 hợp đồng
--  Trạng thái: 'Đang cầm cố' | 'Đã trả khách' | 'Sẵn sàng thanh lý' | 'Đã bán thanh lý'
-- ============================================================
CREATE TABLE TaiSan (
    TaiSanID      INT            IDENTITY(1,1) PRIMARY KEY,
    HopDongID     INT            NOT NULL,
    TenTaiSan     NVARCHAR(200)  NOT NULL,             -- Tên/mô tả tài sản
    LoaiTaiSan    NVARCHAR(100),                        -- Ví dụ: Điện thoại, Xe máy, Trang sức
    GiaTriDinhGia DECIMAL(18,2)  NOT NULL,             -- Giá trị khi định giá lúc cầm
    TrangThai     NVARCHAR(50)   NOT NULL DEFAULT N'Đang cầm cố',
    GhiChu        NVARCHAR(500),
    CONSTRAINT FK_TaiSan_HopDong  FOREIGN KEY (HopDongID) REFERENCES HopDong(HopDongID),
    CONSTRAINT CK_GiaTriDinhGia   CHECK (GiaTriDinhGia > 0),
    CONSTRAINT CK_TrangThai_TS    CHECK (TrangThai IN (
        N'Đang cầm cố', N'Đã trả khách', N'Sẵn sàng thanh lý', N'Đã bán thanh lý'
    ))
);
GO

-- ============================================================
--  BẢNG: ThanhToan_Log
--  Audit log: ghi lại MỖI lần khách trả tiền
--  Không bao giờ ghi đè — chỉ INSERT thêm dòng mới
-- ============================================================
CREATE TABLE ThanhToan_Log (
    LogID         INT            IDENTITY(1,1) PRIMARY KEY,
    HopDongID     INT            NOT NULL,
    NhanVienID    INT,                                  -- Nhân viên thu tiền
    NgayTra       DATETIME       NOT NULL DEFAULT GETDATE(),
    SoTienTra     DECIMAL(18,2)  NOT NULL,             -- Số tiền khách trả lần này
    SoTienConNo   DECIMAL(18,2)  NOT NULL,             -- Dư nợ còn lại SAU khi trả
    LoaiThanhToan NVARCHAR(50)   DEFAULT N'Trả thường', -- 'Trả thường' | 'Gia hạn'
    GhiChu        NVARCHAR(500),
    CONSTRAINT FK_Log_HopDong    FOREIGN KEY (HopDongID)  REFERENCES HopDong(HopDongID),
    CONSTRAINT FK_Log_NhanVien   FOREIGN KEY (NhanVienID) REFERENCES NhanVien(NhanVienID),
    CONSTRAINT CK_SoTienTra      CHECK (SoTienTra > 0)
);
GO

-- ============================================================
--  BẢNG: TraTaiSan_Log
--  Ghi nhận từng lần tài sản được trả lại cho khách
-- ============================================================
CREATE TABLE TraTaiSan_Log (
    LogID         INT            IDENTITY(1,1) PRIMARY KEY,
    TaiSanID      INT            NOT NULL,
    HopDongID     INT            NOT NULL,
    NhanVienID    INT,
    NgayTra       DATETIME       NOT NULL DEFAULT GETDATE(),
    GhiChu        NVARCHAR(500),
    CONSTRAINT FK_TraTS_TaiSan   FOREIGN KEY (TaiSanID)   REFERENCES TaiSan(TaiSanID),
    CONSTRAINT FK_TraTS_HopDong  FOREIGN KEY (HopDongID)  REFERENCES HopDong(HopDongID),
    CONSTRAINT FK_TraTS_NhanVien FOREIGN KEY (NhanVienID) REFERENCES NhanVien(NhanVienID)
);
GO
```

**Kết quả tạo bảng:**

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/b5413e5b-6f8f-4720-b349-8c0c4af058e2" />

> 📸 SSMS – Object Explorer hiển thị đủ 6 bảng

---

### 3.2 Thêm Dữ Liệu Mẫu

**Phân tích:** Dữ liệu mẫu được chọn để bao phủ tất cả các trường hợp nghiệp vụ cần kiểm thử: hợp đồng còn trong hạn, hợp đồng đã qua Deadline1 (tính lãi kép), hợp đồng đã qua Deadline2 (sẵn sàng thanh lý), và hợp đồng đã thanh toán xong.

```sql
-- ============================================================
--  THÊM DỮ LIỆU MẪU: NhanVien
-- ============================================================
INSERT INTO NhanVien (HoTen, SoDienThoai, VaiTro) VALUES
(N'Nguyễn Văn An',   '0901111111', N'Thu ngân'),
(N'Trần Thị Bình',   '0902222222', N'Quản lý'),
(N'Lê Văn Cường',    '0903333333', N'Thu ngân');
GO

-- ============================================================
--  THÊM DỮ LIỆU MẪU: KhachHang
--  5 khách hàng với các tình huống khác nhau
-- ============================================================
INSERT INTO KhachHang (HoTen, SoDienThoai, CCCD, DiaChi) VALUES
(N'Phạm Minh Đức',   '0911000001', '001234567890', N'12 Lý Thường Kiệt, Hà Nội'),
(N'Hoàng Thị Thu',   '0911000002', '001234567891', N'45 Trần Phú, Hà Nội'),
(N'Vũ Quốc Hùng',    '0911000003', '001234567892', N'78 Nguyễn Huệ, TP.HCM'),
(N'Đặng Thị Lan',    '0911000004', '001234567893', N'33 Bạch Đằng, Đà Nẵng'),
(N'Bùi Thanh Tùng',  '0911000005', '001234567894', N'90 Lê Lợi, Cần Thơ');
GO

-- ============================================================
--  THÊM DỮ LIỆU MẪU: HopDong
--
--  HD001 – Phạm Minh Đức  : Còn trong hạn (Deadline1 = 60 ngày tới)
--  HD002 – Hoàng Thị Thu  : Đã qua Deadline1, chưa qua Deadline2 → lãi kép
--  HD003 – Vũ Quốc Hùng   : Đã qua Deadline2 → sẵn sàng thanh lý
--  HD004 – Đặng Thị Lan   : Đã thanh toán xong
--  HD005 – Bùi Thanh Tùng : Đang trả góp
-- ============================================================
INSERT INTO HopDong (KhachHangID, SoTienVay, SoTienConLai, NgayVay, Deadline1, Deadline2, TrangThai) VALUES
-- HD001: Vay 5 triệu, còn hạn
(1, 5000000,  5000000,  DATEADD(DAY,-10, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 50, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 80, CAST(GETDATE() AS DATE)),
                        N'Đang vay'),
-- HD002: Vay 10 triệu, đã qua Deadline1 được 15 ngày → tính lãi kép
(2, 10000000, 10000000, DATEADD(DAY,-75, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY,-15, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 15, CAST(GETDATE() AS DATE)),
                        N'Quá hạn'),
-- HD003: Vay 3 triệu, đã qua cả Deadline2
(3, 3000000,  3000000,  DATEADD(DAY,-120,CAST(GETDATE() AS DATE)),
                        DATEADD(DAY,-60, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY,-10, CAST(GETDATE() AS DATE)),
                        N'Quá hạn'),
-- HD004: Vay 2 triệu, đã thanh toán hết
(4, 2000000,  0,        DATEADD(DAY,-30, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 30, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 60, CAST(GETDATE() AS DATE)),
                        N'Đã thanh toán đủ'),
-- HD005: Vay 8 triệu, đã trả một phần (còn 5 triệu)
(5, 8000000,  5000000,  DATEADD(DAY,-20, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 40, CAST(GETDATE() AS DATE)),
                        DATEADD(DAY, 70, CAST(GETDATE() AS DATE)),
                        N'Đang trả góp');
GO

-- ============================================================
--  THÊM DỮ LIỆU MẪU: TaiSan
--  Mỗi hợp đồng có 2–3 tài sản thế chấp
--  Tổng giá trị tài sản luôn lớn hơn số tiền vay (quy tắc nghiệp vụ)
-- ============================================================
INSERT INTO TaiSan (HopDongID, TenTaiSan, LoaiTaiSan, GiaTriDinhGia, TrangThai) VALUES
-- HD001 – Phạm Minh Đức (vay 5tr)
(1, N'iPhone 14 Pro Max 256GB màu tím',  N'Điện thoại',  4000000, N'Đang cầm cố'),
(1, N'MacBook Air M1 2020',              N'Laptop',       3000000, N'Đang cầm cố'),

-- HD002 – Hoàng Thị Thu (vay 10tr, đã quá hạn)
(2, N'Xe máy Honda Wave RSX 2020',       N'Xe máy',       9000000, N'Đang cầm cố'),
(2, N'Đồng hồ Casio G-Shock GW-M5610',  N'Đồng hồ',      2000000, N'Đang cầm cố'),

-- HD003 – Vũ Quốc Hùng (vay 3tr, quá hạn Deadline2)
(3, N'Nhẫn vàng 18K nặng 3 chỉ',        N'Trang sức',    5000000, N'Sẵn sàng thanh lý'),
(3, N'Laptop Dell Inspiron 15 3000',     N'Laptop',       2500000, N'Sẵn sàng thanh lý'),

-- HD004 – Đặng Thị Lan (đã thanh toán)
(4, N'iPad Pro 12.9 inch Wi-Fi 256GB',   N'Máy tính bảng',3000000, N'Đã trả khách'),

-- HD005 – Bùi Thanh Tùng (đang trả góp, vay 8tr còn 5tr)
(5, N'Xe máy Yamaha Exciter 155 2022',   N'Xe máy',       7000000, N'Đang cầm cố'),
(5, N'Apple Watch Series 8 GPS 45mm',    N'Đồng hồ',      2500000, N'Đang cầm cố'),
(5, N'AirPods Pro 2nd Generation',       N'Phụ kiện',     1500000, N'Đã trả khách');
GO

-- ============================================================
--  THÊM DỮ LIỆU MẪU: ThanhToan_Log
--  Ghi lại lịch sử trả tiền của HD004 và HD005
-- ============================================================
INSERT INTO ThanhToan_Log (HopDongID, NhanVienID, NgayTra, SoTienTra, SoTienConNo, LoaiThanhToan, GhiChu) VALUES
-- HD004: Đặng Thị Lan trả hết
(4, 1, DATEADD(DAY,-5, GETDATE()), 2300000, 0, N'Trả thường', N'Trả hết gốc + lãi'),

-- HD005: Bùi Thanh Tùng trả từng phần
(5, 1, DATEADD(DAY,-15, GETDATE()), 2000000, 6800000, N'Trả thường', N'Trả lần 1'),
(5, 2, DATEADD(DAY,-7,  GETDATE()), 1800000, 5000000, N'Trả thường', N'Trả lần 2');
GO
```

**Kết quả thêm dữ liệu:**

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/2cb35900-f0aa-48b5-b0b6-1a4d2e52c1e9" />


**Phân tích kết quả:**
- 5 khách hàng, 5 hợp đồng bao phủ đủ 5 trạng thái hợp đồng.
- Tổng giá trị tài sản mỗi hợp đồng đều ≥ số tiền vay (đảm bảo quy tắc nghiệp vụ).
- HD002 và HD003 ở trạng thái "Quá hạn", trong đó HD003 đã qua cả Deadline2.

---

## 4. Event 1 – Đăng Ký Hợp Đồng Mới

**Phân tích:** Stored Procedure `sp_DangKyHopDong` nhận vào thông tin khách hàng, danh sách tài sản (dạng XML hoặc JSON), số tiền vay và số ngày đến hai deadline. Procedure thực hiện trong một transaction để đảm bảo tính atomicity — nếu bất kỳ bước nào thất bại, toàn bộ thao tác rollback.

```sql
-- ============================================================
--  EVENT 1: STORED PROCEDURE – Đăng ký hợp đồng mới
--
--  Tham số đầu vào:
--    @KhachHangID   : ID khách hàng đã tồn tại
--    @SoTienVay     : Số tiền muốn vay (đồng)
--    @SoNgayDeadline1: Số ngày từ hôm nay đến Deadline1
--    @SoNgayDeadline2: Số ngày từ hôm nay đến Deadline2
--    @DanhSachTaiSan : Danh sách tài sản dạng JSON
--                      Ví dụ: '[{"Ten":"iPhone","Loai":"Dien thoai","GiaTri":5000000}]'
--  Trả về:
--    @NewHopDongID  : ID hợp đồng vừa tạo
-- ============================================================
CREATE OR ALTER PROCEDURE sp_DangKyHopDong
    @KhachHangID      INT,
    @SoTienVay        DECIMAL(18,2),
    @SoNgayDeadline1  INT,          -- Ví dụ: 30 (ngày)
    @SoNgayDeadline2  INT,          -- Ví dụ: 60 (ngày)
    @DanhSachTaiSan   NVARCHAR(MAX) -- JSON array của tài sản
AS
BEGIN
    SET NOCOUNT ON;

    -- ── Khai báo biến ──────────────────────────────────────
    DECLARE @NewHopDongID INT;
    DECLARE @TongGiaTri   DECIMAL(18,2);
    DECLARE @NgayHom_Nay  DATE = CAST(GETDATE() AS DATE);

    -- ── Bắt đầu transaction ────────────────────────────────
    BEGIN TRANSACTION;
    BEGIN TRY

        -- Bước 1: Kiểm tra khách hàng tồn tại
        IF NOT EXISTS (SELECT 1 FROM KhachHang WHERE KhachHangID = @KhachHangID)
            RAISERROR(N'Khách hàng không tồn tại trong hệ thống.', 16, 1);

        -- Bước 2: Kiểm tra Deadline2 phải lớn hơn Deadline1
        IF @SoNgayDeadline1 <= 0 OR @SoNgayDeadline2 <= 0
            RAISERROR(N'Số ngày đến các deadline phải lớn hơn 0.', 16, 1);

        IF @SoNgayDeadline2 <= @SoNgayDeadline1
            RAISERROR(N'Deadline2 phải sau Deadline1.', 16, 1);

        -- Bước 3: Kiểm tra tổng giá trị tài sản >= số tiền vay
        IF @SoTienVay <= 0
            RAISERROR(N'Số tiền vay phải lớn hơn 0.', 16, 1);

        SELECT @TongGiaTri = SUM(CAST(j.GiaTri AS DECIMAL(18,2)))
        FROM OPENJSON(@DanhSachTaiSan)
        WITH (
            Ten    NVARCHAR(200) '$.Ten',
            Loai   NVARCHAR(100) '$.Loai',
            GiaTri DECIMAL(18,2) '$.GiaTri'
        ) j;

        IF @TongGiaTri IS NULL
            RAISERROR(N'Danh sách tài sản không hợp lệ hoặc đang rỗng.', 16, 1);

        IF EXISTS (
            SELECT 1
            FROM OPENJSON(@DanhSachTaiSan)
            WITH (
                Ten    NVARCHAR(200) '$.Ten',
                Loai   NVARCHAR(100) '$.Loai',
                GiaTri DECIMAL(18,2) '$.GiaTri'
            ) j
            WHERE j.GiaTri IS NULL OR j.GiaTri <= 0 OR j.Ten IS NULL
        )
            RAISERROR(N'Dữ liệu tài sản không hợp lệ.', 16, 1);

        IF @TongGiaTri < @SoTienVay
            RAISERROR(N'Tổng giá trị tài sản phải >= số tiền vay.', 16, 1);

        -- Bước 4: Tạo hợp đồng mới
        INSERT INTO HopDong (KhachHangID, SoTienVay, SoTienConLai,
                             NgayVay, Deadline1, Deadline2, TrangThai)
        VALUES (
            @KhachHangID,
            @SoTienVay,
            @SoTienVay,                                         -- Dư nợ ban đầu = tiền vay
            @NgayHom_Nay,
            DATEADD(DAY, @SoNgayDeadline1, @NgayHom_Nay),      -- Deadline1
            DATEADD(DAY, @SoNgayDeadline2, @NgayHom_Nay),      -- Deadline2
            N'Đang vay'
        );

        SET @NewHopDongID = SCOPE_IDENTITY();                   -- Lấy ID vừa tạo

        -- Bước 5: Thêm từng tài sản thế chấp vào bảng TaiSan
        INSERT INTO TaiSan (HopDongID, TenTaiSan, LoaiTaiSan, GiaTriDinhGia, TrangThai)
        SELECT
            @NewHopDongID,
            j.Ten,
            j.Loai,
            j.GiaTri,
            N'Đang cầm cố'
        FROM OPENJSON(@DanhSachTaiSan)
        WITH (
            Ten    NVARCHAR(200) '$.Ten',
            Loai   NVARCHAR(100) '$.Loai',
            GiaTri DECIMAL(18,2) '$.GiaTri'
        ) j;

        COMMIT TRANSACTION;

        -- Trả kết quả về cho caller
        SELECT
            @NewHopDongID          AS HopDongID,
            @SoTienVay             AS SoTienVay,
            DATEADD(DAY, @SoNgayDeadline1, @NgayHom_Nay) AS Deadline1,
            DATEADD(DAY, @SoNgayDeadline2, @NgayHom_Nay) AS Deadline2,
            N'Hợp đồng đã tạo thành công' AS ThongBao;

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        -- Ném lại lỗi để caller biết
        THROW;
    END CATCH
END;
GO

-- ── Kiểm thử Event 1 ─────────────────────────────────────────
-- Tạo hợp đồng mới cho khách hàng ID=1, vay 6 triệu,
-- Deadline1 sau 30 ngày, Deadline2 sau 60 ngày,
-- thế chấp 2 tài sản tổng giá trị 8 triệu
EXEC sp_DangKyHopDong
    @KhachHangID     = 1,
    @SoTienVay       = 6000000,
    @SoNgayDeadline1 = 30,
    @SoNgayDeadline2 = 60,
    @DanhSachTaiSan  = N'[
        {"Ten":"Samsung Galaxy S23 Ultra","Loai":"Dien thoai","GiaTri":5000000},
        {"Ten":"Tai nghe Sony WH-1000XM5","Loai":"Phu kien","GiaTri":3000000}
    ]';
GO
```

**Kết quả chạy `sp_DangKyHopDong`:**

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/9111bdc7-b4f0-4bb0-acc9-63806e8607ce" />
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/104e2826-0e66-406d-a68b-1f0d4433a4d6" />

 Tạo và kích hoạt EXEC sp_DangKyHopDong

**Phân tích kết quả:**
- Procedure kiểm tra 3 điều kiện trước khi tạo hợp đồng: khách hàng tồn tại, Deadline2 > Deadline1, tổng tài sản ≥ tiền vay.
- Sử dụng `OPENJSON` để parse danh sách tài sản JSON, tiện lợi khi gọi từ ứng dụng.
- Transaction đảm bảo: nếu insert tài sản thất bại, hợp đồng cũng bị rollback.

---

## 5. Event 2 – Tính Toán Công Nợ Thời Gian Thực

**Phân tích:** Hai function được xây dựng theo mô hình phân tầng: `fn_CalcMoneyTransaction` tính lãi cho một lần vay đơn lẻ, rồi `fn_CalcMoneyContract` tổng hợp. Logic lãi kép được xử lý bằng công thức toán học thay vì vòng lặp để tối ưu hiệu năng.

```sql
-- ============================================================
--  EVENT 2a: FUNCTION – Tính tiền phải trả cho 1 hợp đồng
--            tính đến ngày TargetDate
--
--  Công thức:
--    Lãi đơn mỗi ngày = 0.5% / triệu / ngày
--    Trong hạn (≤ Deadline1) : TổngTrả = Gốc × (1 + 0.005 × số_ngày)
--    Quá hạn  (> Deadline1) :
--      LãiĐơnTíchLũy = Gốc × 0.005 × số_ngày_đến_DL1
--      GốcMới        = Gốc + LãiĐơnTíchLũy
--      LãiKép        = GốcMới × ((1 + 0.005)^số_ngày_sau_DL1 - 1)
--      TổngTrả       = Gốc + LãiĐơnTíchLũy + LãiKép
-- ============================================================
CREATE OR ALTER FUNCTION fn_CalcMoneyContract (
    @HopDongID  INT,
    @TargetDate DATE       -- Ngày muốn tính đến
)
RETURNS DECIMAL(18,2)
AS
BEGIN
    DECLARE
        @SoTienVayGoc   DECIMAL(18,2),   -- Tiền gốc ban đầu
        @SoTienConLai   DECIMAL(18,2),   -- Dư nợ gốc còn lại (đã trừ các lần trả gốc)
        @NgayVay        DATE,
        @Deadline1      DATE,
        @SoNgayTrongHan INT,             -- Số ngày từ ngày vay đến Deadline1
        @SoNgayQuaHan   INT,             -- Số ngày từ Deadline1 đến TargetDate
        @LaiDonTichLuy  DECIMAL(18,2),   -- Lãi đơn tích lũy đến Deadline1
        @GocMoi         DECIMAL(18,2),   -- Gốc mới = Gốc + LãiĐơn (cơ sở tính lãi kép)
        @LaiKep         DECIMAL(18,2),   -- Lãi kép tính từ sau Deadline1
        @TongTienPhai   DECIMAL(18,2);   -- Kết quả trả về

    -- Lấy thông tin hợp đồng
    SELECT
        @SoTienVayGoc = SoTienVay,
        @SoTienConLai = SoTienConLai,
        @NgayVay      = NgayVay,
        @Deadline1    = Deadline1
    FROM HopDong
    WHERE HopDongID = @HopDongID;

    -- Nếu hợp đồng không tồn tại → trả về NULL
    IF @SoTienConLai IS NULL RETURN NULL;

    -- ── Trường hợp 1: Chưa quá Deadline1 → chỉ tính lãi đơn ──
    IF @TargetDate <= @Deadline1
    BEGIN
        SET @SoNgayTrongHan = DATEDIFF(DAY, @NgayVay, @TargetDate);

        -- Lãi đơn = Gốc × 0.5% × số_ngày
        SET @TongTienPhai = @SoTienConLai
                          + (@SoTienConLai * 0.005 * @SoNgayTrongHan);
    END

    -- ── Trường hợp 2: Đã quá Deadline1 → lãi kép ──────────
    ELSE
    BEGIN
        -- Số ngày trong hạn (từ ngày vay đến Deadline1)
        SET @SoNgayTrongHan = DATEDIFF(DAY, @NgayVay, @Deadline1);

        -- Số ngày quá hạn (từ Deadline1 đến TargetDate)
        SET @SoNgayQuaHan   = DATEDIFF(DAY, @Deadline1, @TargetDate);

        -- Lãi đơn tích lũy tính đến Deadline1
        SET @LaiDonTichLuy  = @SoTienConLai * 0.005 * @SoNgayTrongHan;

        -- Gốc mới để tính lãi kép = Gốc + Lãi đơn tích lũy
        SET @GocMoi         = @SoTienConLai + @LaiDonTichLuy;

        -- Lãi kép: GốcMới × ((1 + 0.005)^ngày_quá_hạn - 1)
        -- Dùng POWER() của SQL Server — tương đương Math.pow()
        SET @LaiKep         = @GocMoi * (POWER(CAST(1.005 AS FLOAT), @SoNgayQuaHan) - 1);

        -- Tổng phải trả = Gốc gốc + Lãi đơn + Lãi kép
        SET @TongTienPhai   = @SoTienConLai + @LaiDonTichLuy + CAST(@LaiKep AS DECIMAL(18,2));
    END

    RETURN @TongTienPhai;
END;
GO

-- ============================================================
--  Kiểm thử Event 2
-- ============================================================

-- Test 1: HD001 – Còn trong hạn (chỉ lãi đơn)
SELECT
    1                                             AS HopDongID,
    N'Còn trong hạn'                              AS TinhHuong,
    5000000                                        AS TienGoc,
    dbo.fn_CalcMoneyContract(1, CAST(GETDATE() AS DATE)) AS TongTienPhai_HomNay;

-- Test 2: HD002 – Đã qua Deadline1 (tính lãi kép)
SELECT
    2                                              AS HopDongID,
    N'Đã qua Deadline1 – lãi kép'                 AS TinhHuong,
    10000000                                        AS TienGoc,
    dbo.fn_CalcMoneyContract(2, CAST(GETDATE() AS DATE)) AS TongTienPhai_HomNay,
    dbo.fn_CalcMoneyContract(2, DATEADD(MONTH,1,CAST(GETDATE() AS DATE))) AS TongTienPhai_Sau1Thang;
GO
```

**Kết quả tính lãi:**
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/932b1900-8312-4b0d-9ac9-4177dad88200" />
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/66a48bf6-26b2-4a45-96bd-7ad05fcb5afe" />

Sử dụng hàm  fn_CalcMoneyContract tại đây

**Phân tích kết quả:**
- HD001 (còn hạn 50 ngày): Lãi đơn nhẹ, tổng tiền tăng tuyến tính.
- HD002 (quá hạn 15 ngày): Lãi kép khiến tổng tiền tăng nhanh hơn lãi đơn. Sau 1 tháng nữa, con số này sẽ lớn hơn đáng kể do tính chất lũy thừa.
- Hàm `POWER(1.005, n)` là cốt lõi tính lãi kép hàng ngày.

---

## 6. Event 3 – Xử Lý Trả Nợ & Hoàn Trả Tài Sản

**Phân tích:** Đây là procedure phức tạp nhất vì phải xử lý nhiều nhánh: tài sản đã thanh lý, trả hết nợ, trả một phần. Ngoài ra còn tính toán danh sách tài sản được đề xuất trả lại dựa trên điều kiện tài sản còn lại ≥ dư nợ.

```sql
-- ============================================================
--  EVENT 3: STORED PROCEDURE – Xử lý trả nợ
--
--  Logic:
--    1. Nếu hợp đồng đã thanh lý → thông báo, không thu tiền
--    2. Tính tổng nợ hiện tại (Gốc + Lãi)
--    3. Trừ tiền khách trả vào dư nợ
--    4. Nếu trả đủ → cập nhật trạng thái "Đã thanh toán đủ", trả hết tài sản
--    5. Nếu chưa đủ → cập nhật "Đang trả góp", ghi log
--    6. Gợi ý danh sách tài sản có thể trả lại
-- ============================================================
CREATE OR ALTER PROCEDURE sp_XuLyTraNo
    @HopDongID    INT,
    @SoTienTra    DECIMAL(18,2),
    @NhanVienID   INT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE
        @TrangThaiHD    NVARCHAR(50),
        @TongNoCurrent  DECIMAL(18,2),
        @SoTienConLai   DECIMAL(18,2),
        @DuNoSauKhiTra  DECIMAL(18,2),
        @NgayHomNay     DATE = CAST(GETDATE() AS DATE);

    -- ── Bước 1: Lấy trạng thái hợp đồng ──────────────────
    SELECT
        @TrangThaiHD  = TrangThai,
        @SoTienConLai = SoTienConLai
    FROM HopDong
    WHERE HopDongID = @HopDongID;

    IF @TrangThaiHD IS NULL
    BEGIN
        SELECT N'Hợp đồng không tồn tại.' AS ThongBao; RETURN;
    END

    -- ── Bước 2: Kiểm tra tài sản đã bị thanh lý ──────────
    IF @TrangThaiHD = N'Đã thanh lý'
    BEGIN
        SELECT N'Tài sản đã bị thanh lý. Không thu tiền, không trả đồ.' AS ThongBao;
        RETURN;
    END

    -- ── Bước 3: Tính tổng nợ hiện tại (Gốc + Lãi) ────────
    IF @SoTienTra <= 0
    BEGIN
        SELECT N'Số tiền trả phải lớn hơn 0.' AS ThongBao; RETURN;
    END

    SET @TongNoCurrent = dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay);

    BEGIN TRANSACTION;
    BEGIN TRY

        -- ── Bước 4: Tính dư nợ sau khi trừ tiền trả ──────
        SET @DuNoSauKhiTra = @TongNoCurrent - @SoTienTra;

        IF @DuNoSauKhiTra <= 0
        BEGIN
            -- ── TRƯỜNG HỢP A: Trả đủ hoặc thừa ───────────

            -- Cập nhật hợp đồng: dư nợ = 0, trạng thái = Đã thanh toán
            UPDATE HopDong
            SET SoTienConLai = 0,
                TrangThai    = N'Đã thanh toán đủ'
            WHERE HopDongID = @HopDongID;

            -- Trả tất cả tài sản còn đang cầm cố
            DECLARE @TaiSanDuocTra TABLE (TaiSanID INT PRIMARY KEY);

            INSERT INTO @TaiSanDuocTra (TaiSanID)
            SELECT TaiSanID
            FROM TaiSan
            WHERE HopDongID = @HopDongID
              AND TrangThai  = N'Đang cầm cố';

            UPDATE ts
            SET TrangThai = N'Đã trả khách'
            FROM TaiSan ts
            JOIN @TaiSanDuocTra x ON ts.TaiSanID = x.TaiSanID;

            -- Ghi log trả tài sản
            INSERT INTO TraTaiSan_Log (TaiSanID, HopDongID, NhanVienID, GhiChu)
            SELECT x.TaiSanID, @HopDongID, @NhanVienID, N'Trả hết khi thanh toán đủ'
            FROM @TaiSanDuocTra x;

            -- Ghi log thanh toán
            INSERT INTO ThanhToan_Log (HopDongID, NhanVienID, NgayTra, SoTienTra, SoTienConNo, GhiChu)
            VALUES (@HopDongID, @NhanVienID, GETDATE(), @SoTienTra, 0, N'Thanh toán đủ, trả hết tài sản');

            SELECT N'Đã thanh toán đủ. Toàn bộ tài sản đã được trả lại.' AS ThongBao,
                   @TongNoCurrent     AS TongNoTruocKhiTra,
                   @SoTienTra         AS SoTienKhachTra,
                   0                  AS DuNoConLai;
        END
        ELSE
        BEGIN
            -- ── TRƯỜNG HỢP B: Chưa trả đủ ─────────────────

            -- Trừ tiền trả vào dư nợ gốc (ghi nhận phần gốc đã trả)
            -- Giả định: tiền trả ưu tiên trả lãi trước, sau đó mới trả gốc
            DECLARE @LaiHienTai DECIMAL(18,2) = @TongNoCurrent - @SoTienConLai;
            DECLARE @ConLaiGocSauKhiTra DECIMAL(18,2);

            IF @SoTienTra >= @LaiHienTai
                -- Trả đủ lãi, còn dư thì trừ vào gốc
                SET @ConLaiGocSauKhiTra = @SoTienConLai - (@SoTienTra - @LaiHienTai)
            ELSE
                -- Chưa đủ trả lãi, gốc giữ nguyên
                SET @ConLaiGocSauKhiTra = @SoTienConLai;

            -- Cập nhật dư nợ gốc
            UPDATE HopDong
            SET SoTienConLai = @ConLaiGocSauKhiTra,
                TrangThai    = N'Đang trả góp'
            WHERE HopDongID = @HopDongID;

            -- Ghi log thanh toán
            INSERT INTO ThanhToan_Log (HopDongID, NhanVienID, NgayTra, SoTienTra, SoTienConNo, GhiChu)
            VALUES (@HopDongID, @NhanVienID, GETDATE(), @SoTienTra,
                    dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay),
                    N'Trả góp một phần');

            SELECT N'Đã ghi nhận thanh toán một phần.' AS ThongBao,
                   @TongNoCurrent         AS TongNoTruocKhiTra,
                   @SoTienTra             AS SoTienKhachTra,
                   @DuNoSauKhiTra         AS DuNoConLai;
        END

        COMMIT TRANSACTION;

        -- ── Bước 5: Gợi ý tài sản có thể trả lại ─────────
        -- Điều kiện: Tổng giá trị tài sản còn lại (sau khi trả tài sản X)
        --            vẫn >= dư nợ hiện tại
        DECLARE @DuNoMoi DECIMAL(18,2) = dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay);
        DECLARE @TongGiaTriConLai DECIMAL(18,2);

        SELECT @TongGiaTriConLai = SUM(GiaTriDinhGia)
        FROM TaiSan
        WHERE HopDongID = @HopDongID AND TrangThai = N'Đang cầm cố';

        SELECT
            ts.TaiSanID,
            ts.TenTaiSan,
            ts.LoaiTaiSan,
            ts.GiaTriDinhGia,
            (@TongGiaTriConLai - ts.GiaTriDinhGia) AS GiaTriConLaiNeuTraTS,
            @DuNoMoi                                AS DuNoHienTai,
            CASE
                WHEN (@TongGiaTriConLai - ts.GiaTriDinhGia) >= @DuNoMoi
                THEN N'✅ Có thể trả'
                ELSE N'❌ Không thể trả (dư nợ > giá trị tài sản còn)'
            END AS GoiY
        FROM TaiSan ts
        WHERE ts.HopDongID = @HopDongID
          AND ts.TrangThai  = N'Đang cầm cố'
        ORDER BY ts.GiaTriDinhGia ASC;

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO

-- ── Kiểm thử: Khách HD005 trả 2.000.000đ ────────────────────
EXEC sp_XuLyTraNo
    @HopDongID  = 5,
    @SoTienTra  = 2000000,
    @NhanVienID = 1;
GO
```

**Kết quả chạy `sp_XuLyTraNo`:**
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d93f710c-d7ca-4865-a435-3539a91bba3e" />
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/76afdae1-27be-460e-9109-305554971b4d" />

 Tạo va chạy sp_XuLyTraNo kiểm tra ThanhToan_Log 

**Phân tích kết quả:**
- Bảng gợi ý hiển thị từng tài sản và cho biết nếu trả tài sản đó thì giá trị còn lại có đủ bảo đảm dư nợ không.
- Logic ưu tiên trả lãi trước, sau đó mới trừ vào gốc — đúng thực tế nghiệp vụ cầm đồ.

---

## 7. Event 4 – Truy Vấn Danh Sách Nợ Xấu

**Phân tích:** Query này tổng hợp dữ liệu từ nhiều bảng, sử dụng lại function `fn_CalcMoneyContract` để tính toán lãi kép đến ngày hiện tại và sau 1 tháng. Kết quả giúp nhân viên nhận biết ngay các hợp đồng cần theo dõi đặc biệt.

```sql
-- ============================================================
--  EVENT 4: TRUY VẤN DANH SÁCH NỢ XẤU
--
--  Điều kiện nợ xấu: Đã qua Deadline1 mà chưa thanh toán đủ
--  Các cột yêu cầu:
--    - Tên KH, SĐT
--    - Số tiền vay gốc
--    - Số ngày quá hạn (kể từ Deadline1)
--    - Tổng tiền phải trả hiện tại
--    - Tổng tiền phải trả sau 1 tháng nữa
-- ============================================================
SELECT
    kh.HoTen                                         AS [Tên Khách Hàng],
    kh.SoDienThoai                                    AS [Số Điện Thoại],
    hd.SoTienVay                                      AS [Tiền Gốc (đ)],

    -- Số ngày quá hạn = số ngày từ Deadline1 đến hôm nay
    DATEDIFF(DAY, hd.Deadline1, CAST(GETDATE() AS DATE))
                                                       AS [Số Ngày Quá Hạn],

    -- Tổng tiền phải trả đến hôm nay (Gốc + Lãi đơn + Lãi kép)
    dbo.fn_CalcMoneyContract(hd.HopDongID, CAST(GETDATE() AS DATE))
                                                       AS [Tổng Nợ Hiện Tại (đ)],

    -- Tổng tiền phải trả nếu để thêm 30 ngày nữa
    dbo.fn_CalcMoneyContract(hd.HopDongID, DATEADD(MONTH, 1, CAST(GETDATE() AS DATE)))
                                                       AS [Tổng Nợ Sau 1 Tháng (đ)],

    hd.Deadline1                                       AS [Deadline 1],
    hd.Deadline2                                       AS [Deadline 2],
    hd.TrangThai                                       AS [Trạng Thái]

FROM HopDong hd
JOIN KhachHang kh ON kh.KhachHangID = hd.KhachHangID

WHERE
    -- Đã quá Deadline1
    hd.Deadline1 < CAST(GETDATE() AS DATE)
    -- Chưa thanh toán xong và chưa thanh lý
  AND hd.TrangThai NOT IN (N'Đã thanh toán đủ', N'Đã thanh lý')

ORDER BY
    -- Ưu tiên hiển thị những khoản quá hạn lâu nhất lên đầu
    DATEDIFF(DAY, hd.Deadline1, CAST(GETDATE() AS DATE)) DESC;
GO
```

**Kết quả danh sách nợ xấu:**

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/bfe2c749-0ffb-4548-99a1-3d7ed355d562" />


**Phân tích kết quả:**
- HD002 (Hoàng Thị Thu): Quá hạn 15 ngày, lãi kép đang tích lũy.
- HD003 (Vũ Quốc Hùng): Quá hạn lâu hơn, đã qua cả Deadline2, dư nợ cao nhất.
- Cột "Tổng Nợ Sau 1 Tháng" cho thấy tác động của lãi kép — con số tăng nhanh đáng kể, giúp nhân viên thuyết phục khách thanh toán sớm.

---

## 8. Event 5 – Quản Lý Thanh Lý Tài Sản (Trigger)

**Phân tích:** Ba trigger được thiết kế theo chuỗi trạng thái: `Đang vay` → `Quá hạn` → `Sẵn sàng thanh lý` → `Đã bán thanh lý`. Mỗi trigger chỉ kích hoạt khi đúng điều kiện chuyển trạng thái, tránh trigger vô hạn.

```sql
-- ============================================================
--  EVENT 5 – TRIGGER 1:
--  Chuyển HopDong sang "Quá hạn" khi cập nhật bất kỳ trường nào
--  mà Deadline1 đã qua & trạng thái vẫn là "Đang vay"
--
--  Ghi chú: SQL Server không có event scheduler tự động.
--  Trong thực tế, cần SQL Agent Job chạy script này định kỳ
--  hoặc trigger trên UPDATE. Ta dùng AFTER UPDATE ở đây.
-- ============================================================
CREATE OR ALTER TRIGGER trg_ChuyenTrangThai_QuaHan
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Cập nhật các hợp đồng vừa được UPDATE mà đã qua Deadline1
    -- Điều kiện: trạng thái cũ = 'Đang vay' (trong DELETED)
    --            và Deadline1 < ngày hôm nay
    UPDATE hd
    SET hd.TrangThai = N'Quá hạn'
    FROM HopDong hd
    JOIN inserted i ON hd.HopDongID = i.HopDongID
    JOIN deleted  d ON hd.HopDongID = d.HopDongID
    WHERE
        d.TrangThai = N'Đang vay'          -- Trạng thái TRƯỚC khi update
      AND hd.Deadline1 < CAST(GETDATE() AS DATE)  -- Đã quá Deadline1
      AND hd.TrangThai  = N'Đang vay';     -- Vẫn còn "Đang vay" (chưa bị trigger khác đổi)
END;
GO

-- ============================================================
--  EVENT 5 – TRIGGER 2:
--  Chuyển TaiSan sang "Sẵn sàng thanh lý"
--  khi HopDong đang "Quá hạn" mà đã qua Deadline2
-- ============================================================
CREATE OR ALTER TRIGGER trg_ChuyenTaiSan_SanSangThanhLy
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Lấy danh sách HopDong vừa update
    -- mà trạng thái hiện tại = 'Quá hạn' và đã qua Deadline2
    UPDATE ts
    SET ts.TrangThai = N'Sẵn sàng thanh lý'
    FROM TaiSan ts
    JOIN inserted i ON ts.HopDongID = i.HopDongID
    WHERE
        i.TrangThai = N'Quá hạn'                     -- Hợp đồng vừa ở trạng thái Quá hạn
      AND i.Deadline2 < CAST(GETDATE() AS DATE)       -- Đã qua Deadline2
      AND ts.TrangThai = N'Đang cầm cố';              -- Tài sản vẫn đang được cầm
END;
GO

-- ============================================================
--  EVENT 5 – TRIGGER 3:
--  Chuyển TaiSan sang "Đã bán thanh lý"
--  khi HopDong chuyển sang trạng thái "Đã thanh lý"
-- ============================================================
CREATE OR ALTER TRIGGER trg_ChuyenTaiSan_DaBanThanhLy
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE ts
    SET ts.TrangThai = N'Đã bán thanh lý'
    FROM TaiSan ts
    JOIN inserted i ON ts.HopDongID = i.HopDongID
    JOIN deleted  d ON ts.HopDongID = d.HopDongID
    WHERE
        i.TrangThai = N'Đã thanh lý'               -- Hợp đồng vừa chuyển sang "Đã thanh lý"
      AND d.TrangThai = N'Quá hạn'                  -- Từ trạng thái "Quá hạn"
      AND ts.TrangThai IN (N'Sẵn sàng thanh lý', N'Đang cầm cố');
END;
GO

-- ── Kiểm thử Trigger ──────────────────────────────────────────
-- Mô phỏng nhân viên xác nhận thanh lý HD003
UPDATE HopDong
SET TrangThai = N'Đã thanh lý'
WHERE HopDongID = 3;

-- Kiểm tra tài sản của HD003 đã chuyển trạng thái chưa
SELECT TaiSanID, TenTaiSan, TrangThai
FROM TaiSan
WHERE HopDongID = 3;
GO
```

**Kết quả kiểm thử Trigger:**
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/9eb8a6a4-8f0e-49bf-87f2-abab005b958d" />

**Phân tích kết quả:**
- Ngay sau khi UPDATE HopDong sang "Đã thanh lý", trigger `trg_ChuyenTaiSan_DaBanThanhLy` tự động kích hoạt.
- Toàn bộ tài sản của HD003 chuyển từ "Sẵn sàng thanh lý" → "Đã bán thanh lý" mà không cần thao tác thủ công.
- Sử dụng bảng ảo `inserted` và `deleted` trong trigger giúp chỉ xử lý đúng các hàng vừa thay đổi, không ảnh hưởng toàn bộ bảng.

---

## 9. Sự Kiện Bổ Sung – Gia Hạn Hợp Đồng

**Phân tích:** Khách trả hết lãi tích lũy để "reset" về trạng thái bình thường — Deadline1 và Deadline2 được dời về tương lai, tránh bị tính lãi kép trong kỳ mới.

```sql
-- ============================================================
--  SỰ KIỆN BỔ SUNG: Gia hạn hợp đồng
--
--  Khách trả toàn bộ lãi tính đến hiện tại
--  → Deadline1 và Deadline2 được dời lên một kỳ hạn mới
--  → Hợp đồng trở về trạng thái "Đang vay" (không còn lãi kép)
--
--  Tham số:
--    @HopDongID       : ID hợp đồng cần gia hạn
--    @SoNgayGiaHan    : Số ngày gia hạn thêm (Ví dụ: 30)
--    @NhanVienID      : Nhân viên thực hiện
-- ============================================================
CREATE OR ALTER PROCEDURE sp_GiaHanHopDong
    @HopDongID     INT,
    @SoNgayGiaHan  INT,
    @NhanVienID    INT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE
        @NgayHomNay     DATE           = CAST(GETDATE() AS DATE),
        @TongNoHienTai  DECIMAL(18,2),
        @SoTienConLai   DECIMAL(18,2),
        @TienLaiPhaiTra DECIMAL(18,2),  -- = Tổng nợ - Gốc = tiền lãi khách phải trả
        @Deadline1Cu    DATE,
        @Deadline2Cu    DATE;

    -- Lấy thông tin hợp đồng
    SELECT
        @SoTienConLai = SoTienConLai,
        @Deadline1Cu  = Deadline1,
        @Deadline2Cu  = Deadline2
    FROM HopDong WHERE HopDongID = @HopDongID;

    -- Tính tổng lãi cần trả để gia hạn
    SET @TongNoHienTai  = dbo.fn_CalcMoneyContract(@HopDongID, @NgayHomNay);
    SET @TienLaiPhaiTra = @TongNoHienTai - @SoTienConLai;

    BEGIN TRANSACTION;
    BEGIN TRY

        -- Dời Deadline1 và Deadline2 thêm @SoNgayGiaHan ngày từ hôm nay
        UPDATE HopDong
        SET
            Deadline1  = DATEADD(DAY, @SoNgayGiaHan, @NgayHomNay),
            Deadline2  = DATEADD(DAY, @SoNgayGiaHan * 2, @NgayHomNay),
            -- Giữ nguyên dư nợ gốc (khách chỉ trả lãi, không trả gốc)
            TrangThai  = N'Đang vay'
        WHERE HopDongID = @HopDongID;

        -- Ghi log khoản lãi khách đã trả để gia hạn
        INSERT INTO ThanhToan_Log
            (HopDongID, NhanVienID, NgayTra, SoTienTra, SoTienConNo, LoaiThanhToan, GhiChu)
        VALUES (
            @HopDongID,
            @NhanVienID,
            GETDATE(),
            @TienLaiPhaiTra,                -- Chỉ trả lãi, gốc giữ nguyên
            @SoTienConLai,                  -- Dư nợ gốc không đổi
            N'Gia hạn',
            N'Gia hạn hợp đồng ' + CAST(@SoNgayGiaHan AS VARCHAR) + N' ngày'
        );

        COMMIT TRANSACTION;

        SELECT
            N'Gia hạn thành công' AS ThongBao,
            @TienLaiPhaiTra       AS TienLaiKhachDaTra,
            @SoTienConLai         AS DuNoGocConLai,
            DATEADD(DAY, @SoNgayGiaHan, @NgayHomNay)     AS Deadline1_Moi,
            DATEADD(DAY, @SoNgayGiaHan * 2, @NgayHomNay) AS Deadline2_Moi;

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO

-- ── Kiểm thử: Gia hạn HD002 thêm 30 ngày ─────────────────────
EXEC sp_GiaHanHopDong
    @HopDongID    = 2,
    @SoNgayGiaHan = 30,
    @NhanVienID   = 2;
GO
```

**Kết quả gia hạn:**

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/ecabc32b-8450-4340-8b2d-4bb9816b6fd1" />


**Phân tích kết quả:**
- Deadline1 và Deadline2 của HD002 đã được dời về tương lai.
- Trạng thái trở về "Đang vay" — hợp đồng không còn bị tính lãi kép trong kỳ mới.
- Tiền lãi khách trả được ghi vào `ThanhToan_Log` với `LoaiThanhToan = 'Gia hạn'` để phân biệt với trả nợ thông thường.


L.*
