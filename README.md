# BTVN QUẢN LÍ CẦM ĐỒ  
# Nhiệm vụ 1: Thiết kế CSDL

# Nhiệm vụ 2: Cài đặt SQL ( Yêu cầu viết Scripts ) 
## 1. Script tạo các bảng Danh mục (Master Data)
``` SQL 
-- Tạo bảng danh mục Trạng thái Hợp đồng
CREATE TABLE DM_TrangThaiHD (
    MaTTHD INT PRIMARY KEY,
    TenTrangThai NVARCHAR(50) NOT NULL
);

INSERT INTO DM_TrangThaiHD VALUES 
(1, N'Đang vay'), 
(2, N'Quá hạn (Nợ xấu)'), 
(3, N'Đã tất toán'), 
(4, N'Đã thanh lý tài sản');

-- Tạo bảng danh mục Trạng thái Tài sản
CREATE TABLE DM_TrangThaiTS (
    MaTTTS INT PRIMARY KEY,
    TenTrangThai NVARCHAR(50) NOT NULL
);

INSERT INTO DM_TrangThaiTS VALUES 
(1, N'Đang cầm cố'), 
(2, N'Đã trả khách'), 
(3, N'Sẵn sàng thanh lý'), 
(4, N'Đã bán thanh lý');
```
<img width="2877" height="1794" alt="image" src="https://github.com/user-attachments/assets/ea21993e-0662-45dd-99a5-3f104b3a2a12" />

## 2. Script tạo các bảng Thực thể chính
```SQL
-- Bảng Khách hàng
CREATE TABLE KhachHang (
    MaKH INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(100) NOT NULL,
    SoCCCD VARCHAR(20) UNIQUE NOT NULL,
    SDT VARCHAR(15),
    DiaChi NVARCHAR(255)
);

-- Bảng Nhân viên
CREATE TABLE NhanVien (
    MaNV INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(100) NOT NULL,
    ChucVu NVARCHAR(50),
    SDT VARCHAR(15)
);

-- Bảng Hợp đồng
CREATE TABLE HopDong (
    MaHD INT IDENTITY(1,1) PRIMARY KEY,
    MaKH INT FOREIGN KEY REFERENCES KhachHang(MaKH),
    MaNV INT FOREIGN KEY REFERENCES NhanVien(MaNV),
    NgayLap DATETIME DEFAULT GETDATE(),
    SoTienVayGoc MONEY NOT NULL,
    Deadline1 DATETIME NOT NULL,
    Deadline2 DATETIME NOT NULL,
    MaTTHD INT FOREIGN KEY REFERENCES DM_TrangThaiHD(MaTTHD)
);

-- Bảng Tài sản
CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    TenTaiSan NVARCHAR(255) NOT NULL,
    GiaTriDinhGia MONEY NOT NULL,
    MaTTTS INT FOREIGN KEY REFERENCES DM_TrangThaiTS(MaTTTS),
    IsSold BIT DEFAULT 0
);
```

<img width="2872" height="1786" alt="image" src="https://github.com/user-attachments/assets/30f34d11-9b6d-4bfa-b2bd-61365aa597df" />

## 3. Script tạo bảng Nghiệp vụ (Trả góp & Lãi)
```SQL
-- Bảng Giao dịch (Xử lý cả Trả thẳng và Trả góp)
CREATE TABLE GiaoDich (
    MaGD INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    MaNV INT FOREIGN KEY REFERENCES NhanVien(MaNV),
    NgayGD DATETIME DEFAULT GETDATE(),
    LoaiThanhToan NVARCHAR(20), -- 'Trả thẳng' hoặc 'Trả góp'
    SoTienThu MONEY NOT NULL,
    TruNoGoc MONEY DEFAULT 0,
    TraLai MONEY DEFAULT 0,
    GhiChu NVARCHAR(MAX)
);
```

<img width="2879" height="1798" alt="image" src="https://github.com/user-attachments/assets/e4b64870-809f-44c2-958e-7bf35a311694" />

## Tạo bảng thành công 
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/f6e5a3a6-e8e7-41e7-9926-fa1c973727d9" />

## Event 1: Tiếp nhận Hợp đồng mới
### a. Viết Logic Đăng ký Hợp đồng (Stored Procedure)

```SQL

-- Tạo Type để truyền danh sách tài sản
CREATE TYPE TableTaiSan AS TABLE (
    TenTS NVARCHAR(255),
    GiaTri MONEY
);
GO

-- Store Procedure chính cho Event 1
CREATE PROCEDURE sp_DangKyHopDong
    @HoTen NVARCHAR(100),
    @SoCCCD VARCHAR(20),
    @SDT VARCHAR(15),
    @DiaChi NVARCHAR(255),
    @MaNV INT,
    @SoTienVay MONEY,
    @Deadline1 DATETIME,
    @Deadline2 DATETIME,
    @DS_TaiSan TableTaiSan READONLY
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MaKH INT, @MaHD INT;

    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Kiểm tra/Thêm khách hàng
        SELECT @MaKH = MaKH FROM KhachHang WHERE SoCCCD = @SoCCCD;
        IF @MaKH IS NULL
        BEGIN
            INSERT INTO KhachHang (HoTen, SoCCCD, SDT, DiaChi) 
            VALUES (@HoTen, @SoCCCD, @SDT, @DiaChi);
            SET @MaKH = SCOPE_IDENTITY();
        END

        -- Tạo Hợp đồng
        INSERT INTO HopDong (MaKH, MaNV, NgayLap, SoTienVayGoc, Deadline1, Deadline2)
        VALUES (@MaKH, @MaNV, GETDATE(), @SoTienVay, @Deadline1, @Deadline2);
        SET @MaHD = SCOPE_IDENTITY();

        -- Thêm danh sách tài sản
        INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia)
        SELECT @MaHD, TenTS, GiaTri FROM @DS_TaiSan;

        COMMIT TRANSACTION;
        PRINT N'Thành công! Đã tạo Hợp đồng ID: ' + CAST(@MaHD AS VARCHAR);
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT N'Lỗi: ' + ERROR_MESSAGE();
    END CATCH
END
GO
```


<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/50785fd9-1922-4349-b43b-e76e4c02a1c7" />

### b. Lấy ví dụ cho 3 khách hàng 

```SQL
-- Thêm nhân viên quản lý
INSERT INTO NhanVien (HoTen, ChucVu) VALUES (N'Lê Minh Quản Lý', N'Chủ tiệm'), (N'Trần Thu Ngân', N'Nhân viên');

-- KHÁCH 1: Cầm 1 Xe máy
DECLARE @List1 TableTaiSan;
INSERT INTO @List1 VALUES (N'Xe Yamaha Exciter 155', 42000000);
EXEC sp_DangKyHopDong N'Nguyễn Văn An', '012345678001', '0901111222', N'Hà Nội', 1, 25000000, '2026-06-01', '2026-07-01', @List1;

-- KHÁCH 2: Cầm 2 món đồ công nghệ
DECLARE @List2 TableTaiSan;
INSERT INTO @List2 VALUES (N'Macbook Air M3', 32000000), (N'iPhone 15 Pro', 26000000);
EXEC sp_DangKyHopDong N'Lê Thị Bình', '012345678002', '0903333444', N'Đà Nẵng', 1, 30000000, '2026-05-20', '2026-06-20', @List2;

-- KHÁCH 3: Cầm 2 món đồ trang sức
DECLARE @List3 TableTaiSan;
INSERT INTO @List3 VALUES (N'Lắc tay vàng 24K', 18000000), (N'Nhẫn vàng 9999', 12000000);
EXEC sp_DangKyHopDong N'Phạm Văn Cường', '012345678003', '0905555666', N'TP.HCM', 2, 10000000, '2026-06-15', '2026-08-15', @List3;

-- TRUY VẤN KIỂM TRA
SELECT * FROM KhachHang;
SELECT * FROM HopDong;
SELECT * FROM TaiSan;
```

<img width="2878" height="1797" alt="image" src="https://github.com/user-attachments/assets/8ff7071e-3eb8-48f6-b480-ccc9285e4005" />

## Event 2: Tính toán công nợ thời gian thực
- Đây là phần quan trọng nhất, xử lý "bộ não" của hệ thống để tính toán tiền lãi đơn và lãi kép dựa trên quy tắc: $5.000đ/1.000.000đ$ gốc mỗi ngày.
- Trong SQL Server, cách tốt nhất để tái sử dụng logic này là viết một Scalar-Valued Function.
### a. Ý tưởng thuật toán
- Lãi suất hàng ngày ($r$): $5.000 / 1.000.000 = 0.005$ (0.5%/ngày).
- Giai đoạn 1 (Lãi đơn): Từ ngày lập đến Deadline1.
- Công thức: $Gốc + (Gốc \times r \times Số\ngày)$.
- Giai đoạn 2 (Lãi kép): Từ sau Deadline1 đến ngày hiện tại. Phần lãi đơn tích lũy được sẽ nhập vào gốc.
- Công thức: $Gốc\mới \times (1 + r)^{Sốngàyquáhạn}$.
- Công nợ cuối cùng: Tổng tiền (Gốc + Lãi) trừ đi số tiền khách đã trả (lấy từ bảng GiaoDich).
### b. Mã SQL: Hàm tính tổng nợ (fn_TinhTongNo)
```SQL
CREATE FUNCTION fn_TinhTongNo (@MaHD INT, @NgayTinh DATETIME)
RETURNS MONEY
AS
BEGIN
    DECLARE @NgayLap DATETIME, @GocBanDau MONEY, @Deadline1 DATETIME;
    DECLARE @TongTienPhaiTra MONEY = 0;
    DECLARE @LaiSuatDaily FLOAT = 0.005; -- 0.5% mỗi ngày
    
    -- 1. Lấy thông tin gốc của hợp đồng
    SELECT 
        @NgayLap = NgayLap, 
        @GocBanDau = SoTienVayGoc, 
        @Deadline1 = Deadline1
    FROM HopDong WHERE MaHD = @MaHD;

    -- 2. Tính toán tiền lãi theo các mốc thời gian
    IF @NgayTinh <= @Deadline1
    BEGIN
        -- TRƯỜNG HỢP LÃI ĐƠN (Trong hạn)
        DECLARE @SoNgayVay INT = DATEDIFF(DAY, @NgayLap, @NgayTinh);
        IF @SoNgayVay < 0 SET @SoNgayVay = 0;
        
        SET @TongTienPhaiTra = @GocBanDau + (@GocBanDau * @LaiSuatDaily * @SoNgayVay);
    END
    ELSE
    BEGIN
        -- TRƯỜNG HỢP LÃI KÉP (Quá hạn Deadline1)
        -- Bước A: Tính tổng tiền (gốc + lãi đơn) tại mốc Deadline1
        DECLARE @SoNgayLaiDon INT = DATEDIFF(DAY, @NgayLap, @Deadline1);
        DECLARE @GocNhapLai MONEY = @GocBanDau + (@GocBanDau * @LaiSuatDaily * @SoNgayLaiDon);
        
        -- Bước B: Tính lãi kép từ Deadline1 đến NgayTinh
        DECLARE @SoNgayQuaHan INT = DATEDIFF(DAY, @Deadline1, @NgayTinh);
        -- Công thức: A = P * (1 + r)^n
        SET @TongTienPhaiTra = @GocNhapLai * POWER((1 + @LaiSuatDaily), @SoNgayQuaHan);
    END

    -- 3. Trừ đi tổng số tiền khách đã từng trả (Trả góp/Trả thẳng)
    DECLARE @DaTra MONEY;
    SELECT @DaTra = ISNULL(SUM(SoTienThu), 0) 
    FROM GiaoDich 
    WHERE MaHD = @MaHD AND NgayGD <= @NgayTinh;

    RETURN @TongTienPhaiTra - @DaTra;
END
GO
```

<img width="2877" height="1793" alt="image" src="https://github.com/user-attachments/assets/98e53920-0679-42f2-b670-b18a8acf7231" />




### c. Cách sử dụng để kiểm tra nợ của 3 khách hàng đã thêm

```SQL
-- Xem danh sách nợ hiện tại của tất cả hợp đồng
SELECT 
    h.MaHD, 
    k.HoTen, 
    h.SoTienVayGoc AS [Gốc gốc],
    h.NgayLap,
    h.Deadline1,
    dbo.fn_TinhTongNo(h.MaHD, GETDATE()) AS [Tổng nợ hiện tại (Gốc + Lãi)],
    dbo.fn_TinhTongNo(h.MaHD, '2026-12-31') AS [Nợ dự tính cuối năm 2026]
FROM HopDong h
JOIN KhachHang k ON h.MaKH = k.MaKH;
```

<img width="2879" height="1796" alt="image" src="https://github.com/user-attachments/assets/7ff0eac1-e6f3-44a5-a73f-265f4bdfae2f" />



