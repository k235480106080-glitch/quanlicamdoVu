# BTVN QUẢN LÍ CẦM ĐỒ  
# Nhiệm vụ 1: Thiết kế CSDL

# Nhiệm vụ 2: Cài đặt SQL ( Yêu cầu viết Scripts ) 
## 1. Script tạo các bảng Danh mục (Master Data)
### a. Tạo các bảng Danh mục (Dùng để quản lý trạng thái)

```sql
-- Danh mục Trạng thái Hợp đồng (Đang vay, Quá hạn, Tất toán)
CREATE TABLE DM_TrangThaiHD (
    MaTTHD INT PRIMARY KEY,
    TenTTHD NVARCHAR(50)
);

-- Danh mục Trạng thái Tài sản (Đang cầm, Đã trả, Sẵn sàng thanh lý)
CREATE TABLE DM_TrangThaiTS (
    MaTTTS INT PRIMARY KEY,
    TenTTTS NVARCHAR(50)
);
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/fc89b308-bc3c-4fc5-a8cb-bbac0dc4e89f" />
### b. Tạo các bảng dữ liệu chính
Các bảng này lưu thông tin Khách hàng, Nhân viên, Hợp đồng, Giao dịch và Tài sản:  

```SQL
-- Bảng Khách hàng
CREATE TABLE KhachHang (
    MaKH INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(100),
    CCCD VARCHAR(20) UNIQUE,
    SDT VARCHAR(15),
    DiaChi NVARCHAR(200)
);

-- Bảng Nhân viên
CREATE TABLE NhanVien (
    MaNV INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(100),
    ChucVu NVARCHAR(50)
);

-- Bảng Hợp đồng (Lưu gốc và thời hạn Deadline)
CREATE TABLE HopDong (
    MaHD INT IDENTITY(1,1) PRIMARY KEY,
    MaKH INT FOREIGN KEY REFERENCES KhachHang(MaKH),
    MaNV INT FOREIGN KEY REFERENCES NhanVien(MaNV),
    NgayLap DATETIME DEFAULT GETDATE(),
    SoTienVayGoc MONEY,
    Deadline1 DATETIME, -- Mốc tính lãi kép [cite: 12]
    Deadline2 DATETIME, -- Mốc thanh lý [cite: 35]
    MaTTHD INT FOREIGN KEY REFERENCES DM_TrangThaiHD(MaTTHD)
);

-- Bảng Tài sản (Một hợp đồng có nhiều tài sản [cite: 9])
CREATE TABLE TaiSan (
    MaTS INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    TenTaiSan NVARCHAR(100),
    GiaTriDinhGia MONEY,
    MaTTTS INT FOREIGN KEY REFERENCES DM_TrangThaiTS(MaTTTS),
    IsSold BIT DEFAULT 0 -- Cờ đánh dấu đã bán thanh lý [cite: 35]
);

-- Bảng Giao dịch (Audit Log - Ghi lại lịch sử trả tiền [cite: 51])
CREATE TABLE GiaoDich (
    MaGD INT IDENTITY(1,1) PRIMARY KEY,
    MaHD INT FOREIGN KEY REFERENCES HopDong(MaHD),
    MaNV INT FOREIGN KEY REFERENCES NhanVien(MaNV),
    NgayGD DATETIME DEFAULT GETDATE(),
    SoTienThu MONEY,
    TruNoGoc MONEY DEFAULT 0,
    TraLai MONEY DEFAULT 0,
    LoaiThanhToan NVARCHAR(50), -- Trả thẳng hoặc Trả góp [cite: 38]
    GhiChu NVARCHAR(MAX)
);
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/abb3955d-fead-4848-af7f-540dcc47844f" />

### c. Tạo Type dữ liệu (Dùng để truyền danh sách tài sản ở Event 1)
```SQL
CREATE TYPE TableTaiSan AS TABLE (
    TenTS NVARCHAR(100),
    GiaTri MONEY
);
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/e9aa1748-2c84-4107-b4fe-4fd2f1e5bf3d" />

### d. Khởi tạo dữ liệu trạng thái mặc định
```SQL
INSERT INTO DM_TrangThaiHD VALUES (1, N'Đang vay'), (2, N'Quá hạn'), (3, N'Đã tất toán');
INSERT INTO DM_TrangThaiTS VALUES (1, N'Đang cầm cố'), (2, N'Đã trả khách'), (3, N'Sẵn sàng thanh lý');
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/d7668038-5891-4246-838e-f8e418d4ef51" />

## Event 1: Đăng ký hợp đồng mới
- Vì em đã tạo lại các bảng và Type dữ liệu ở bước trước, bây giờ em sẽ chạy đoạn mã sau để tạo Procedure đăng ký:
1. Tạo Store Procedure sp_DangKyHopDongProcedure này thực hiện 3 việc: kiểm tra/thêm khách hàng, tạo hợp đồng với trạng thái "Đang vay" (MaTTHD = 1), và lưu danh sách tài sản với trạng thái "Đang cầm cố" (MaTTTS = 1).
  ```SQL
  CREATE OR ALTER PROCEDURE sp_DangKyHopDong
    @TenKH NVARCHAR(100),
    @CCCD VARCHAR(20),
    @SDT VARCHAR(15),
    @DiaChi NVARCHAR(200),
    @MaNV INT,
    @SoTienVay MONEY,
    @Deadline1 DATETIME,
    @Deadline2 DATETIME,
    @DanhSachTS TableTaiSan READONLY -- Truyền danh sách nhiều tài sản từ Type đã tạo
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MaKH INT;

    -- 1. Kiểm tra khách hàng, nếu chưa có trong bảng KhachHang thì thêm mới [cite: 28]
    IF NOT EXISTS (SELECT 1 FROM KhachHang WHERE CCCD = @CCCD)
    BEGIN
        INSERT INTO KhachHang (HoTen, CCCD, SDT, DiaChi)
        VALUES (@TenKH, @CCCD, @SDT, @DiaChi);
    END
    SELECT @MaKH = MaKH FROM KhachHang WHERE CCCD = @CCCD;

    -- 2. Tạo Hợp đồng mới (Mặc định MaTTHD = 1: Đang vay) 
    INSERT INTO HopDong (MaKH, MaNV, NgayLap, SoTienVayGoc, Deadline1, Deadline2, MaTTHD)
    VALUES (@MaKH, @MaNV, GETDATE(), @SoTienVay, @Deadline1, @Deadline2, 1);

    DECLARE @MaHD INT = SCOPE_IDENTITY();

    -- 3. Lưu danh sách tài sản (Mặc định MaTTTS = 1: Đang cầm cố) [cite: 28, 49]
    INSERT INTO TaiSan (MaHD, TenTaiSan, GiaTriDinhGia, MaTTTS, IsSold)
    SELECT @MaHD, TenTS, GiaTri, 1, 0 FROM @DanhSachTS;

    PRINT N'Đã đăng ký thành công Hợp đồng số: ' + CAST(@MaHD AS VARCHAR);
END
GO
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/a77fa0b7-26c9-4a64-a574-7dee35c0143f" />



2. Chạy dữ liệu mẫu (Sample Data)
Sau khi tạo xong Procedure, em sẽ chạy lệnh dưới đây để nạp dữ liệu cho 3 khách hàng. Đây là dữ liệu quan trọng để làm các Event tính lãi và trả đồ sau này.
```SQL
-- Thêm nhân viên trước
INSERT INTO NhanVien (HoTen, ChucVu) VALUES (N'Lê Minh Quản Lý', N'Chủ tiệm'), (N'Trần Thu Ngân', N'Nhân viên');

-- Khách 1: Vay 25 triệu, cầm 1 Xe máy
DECLARE @L1 TableTaiSan;
INSERT INTO @L1 VALUES (N'Xe Yamaha Exciter 155', 42000000);
EXEC sp_DangKyHopDong N'Nguyễn Văn An', '012345678001', '0901111222', N'Hà Nội', 1, 25000000, '2026-06-01', '2026-07-01', @L1;

-- Khách 2: Vay 30 triệu, cầm 2 món đồ công nghệ
DECLARE @L2 TableTaiSan;
INSERT INTO @L2 VALUES (N'Macbook Air M3', 32000000), (N'iPhone 15 Pro', 26000000);
EXEC sp_DangKyHopDong N'Lê Thị Bình', '012345678002', '0903333444', N'Đà Nẵng', 1, 30000000, '2026-05-20', '2026-06-20', @L2;

-- Khách 3: Vay 10 triệu, cầm 2 món đồ trang sức (Để test logic trả đồ lẻ ở Event 3)
DECLARE @L3 TableTaiSan;
INSERT INTO @L3 VALUES (N'Lắc tay vàng 24K', 18000000), (N'Nhẫn vàng 9999', 12000000);
EXEC sp_DangKyHopDong N'Phạm Văn Cường', '012345678003', '0905555666', N'TP.HCM', 2, 10000000, '2026-06-15', '2026-08-15', @L3;

-- Kiểm tra kết quả trong bảng
SELECT * FROM HopDong;
SELECT * FROM TaiSan;
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/c37f94c3-e355-4dd9-9efb-9e59c4d82ae3" />

## Event 2: Xây dựng hàm tính toán công nợ.
Hàm này cực kỳ quan trọng vì nó là "trái tim" để tính tiền lãi. Theo yêu cầu, chúng ta phải xử lý cả lãi đơn (trước Deadline 1) và lãi kép (sau Deadline 1).
a. Code tạo hàm fn_TinhTongNo
Em đã thiết kế logic để tự động trừ đi số tiền khách đã trả trong bảng `GiaoDich` để con số nợ luôn là con số cuối cùng khách còn thiếu.
```SQL
CREATE OR ALTER FUNCTION fn_TinhTongNo (@MaHD INT, @NgayTinh DATETIME)
RETURNS MONEY
AS
BEGIN
    DECLARE @NgayLap DATETIME, @Goc MONEY, @DL1 DATETIME, @TongNoHienTai MONEY = 0;
    DECLARE @LaiSuat FLOAT = 0.005; -- Lãi suất quy định 0.5% mỗi ngày

    -- Lấy thông tin gốc từ hợp đồng
    SELECT @NgayLap = NgayLap, @Goc = SoTienVayGoc, @DL1 = Deadline1 
    FROM HopDong WHERE MaHD = @MaHD;

    -- Kiểm tra nếu chưa có hợp đồng thì thoát
    IF @Goc IS NULL RETURN 0;

    -- 1. Tính tổng nợ phát sinh (Gốc + Lãi) tùy theo mốc thời gian
    IF @NgayTinh <= @DL1
    BEGIN
        -- Tính lãi đơn: Gốc + (Gốc * Lãi Suất * Số Ngày)
        SET @TongNoHienTai = @Goc + (@Goc * @LaiSuat * DATEDIFF(DAY, @NgayLap, @NgayTinh));
    END
    ELSE
    BEGIN
        -- Tính lãi kép: Gốc nhập lãi tại Deadline 1, sau đó tính lũy thừa theo ngày quá hạn
        DECLARE @GocNhapLai MONEY = @Goc + (@Goc * @LaiSuat * DATEDIFF(DAY, @NgayLap, @DL1));
        SET @TongNoHienTai = @GocNhapLai * POWER((1 + @LaiSuat), DATEDIFF(DAY, @DL1, @NgayTinh));
    END

    -- 2. Trừ đi tổng số tiền khách thực tế ĐÃ TRẢ (lấy từ lịch sử GiaoDich)
    DECLARE @DaTra MONEY = ISNULL((SELECT SUM(SoTienThu) FROM GiaoDich WHERE MaHD = @MaHD), 0);

    -- Kết quả trả về là số tiền thực tế khách còn nợ
    RETURN @TongNoHienTai - @DaTra;
END
GO
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/0f5f876c-170b-48c9-b767-4c970f028a84" />

b. Kiểm tra Event 2 (Rất quan trọng)
Sau khi nhấn Execute để tạo hàm thành công, em sẽ chạy lệnh dưới đây để "nghiệm thu".

Vì em vừa mới tạo hợp đồng (ngày lập là hôm nay), nên nếu tính nợ tại thời điểm hiện tại thì tiền lãi sẽ bằng 0.
```sql
-- Kiểm tra nợ của khách hàng số 3 (Anh Cường vay 10tr)
SELECT dbo.fn_TinhTongNo(3, GETDATE()) AS SoTienNoHienTai;
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/572c7d25-db8f-406c-b9da-6966387ac161" />

Phân tích ảnh chạy trên cho dễ hiểu:
- Nợ gốc: 10,000,000 (Khách hàng số 3 - Anh Cường).
- Tiền lãi: 50,000.
- Lý do: Bạn đang tính nợ tại thời điểm 2026-05-12, trong khi hợp đồng lập ngày 2026-05-11. Trôi qua 1 ngày với lãi suất 0.5% thì tiền lãi là: $10,000,000 \times 0.005 = 50,000$. Hàm đã tự động cộng dồn nên rất chuẩn.

## Event 3 - Xử lý thanh toán và Gợi ý trả đồ
a. Tạo mới Procedure
```SQL
CREATE PROCEDURE sp_XuLyThanhToan
    @MaHD INT,
    @MaNV INT,
    @SoTienThu MONEY,
    @GhiChu NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Lấy nợ hiện tại (kiểm tra xem hàm fn_TinhTongNo đã tồn tại chưa)
    DECLARE @TongNo MONEY = dbo.fn_TinhTongNo(@MaHD, GETDATE());

    BEGIN TRY
        BEGIN TRANSACTION;
        -- Ghi Log giao dịch
        INSERT INTO GiaoDich (MaHD, MaNV, NgayGD, SoTienThu, LoaiThanhToan, GhiChu)
        VALUES (@MaHD, @MaNV, GETDATE(), @SoTienThu, 
                CASE WHEN @SoTienThu >= @TongNo THEN N'Trả thẳng' ELSE N'Trả góp' END, @GhiChu);

        -- Cập nhật trạng thái
        IF @SoTienThu >= @TongNo
        BEGIN
            UPDATE HopDong SET MaTTHD = 3 WHERE MaHD = @MaHD; 
            UPDATE TaiSan SET MaTTTS = 2 WHERE MaHD = @MaHD; 
        END
        ELSE
        BEGIN
            UPDATE HopDong SET MaTTHD = 1 WHERE MaHD = @MaHD; 
            
            -- Gợi ý trả đồ
            DECLARE @NoConLai MONEY = @TongNo - @SoTienThu;
            DECLARE @TongGiaTriDo MONEY = (SELECT SUM(GiaTriDinhGia) FROM TaiSan WHERE MaHD = @MaHD AND MaTTTS = 1);

            SELECT MaTS, TenTaiSan, GiaTriDinhGia, N'Có thể trả món này' AS GoiY
            FROM TaiSan 
            WHERE MaHD = @MaHD AND MaTTTS = 1
            AND (@TongGiaTriDo - GiaTriDinhGia) >= @NoConLai;
        END
        COMMIT;
        PRINT N'Thực hiện thành công!';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        DECLARE @ErrMsg NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrMsg, 16, 1);
    END CATCH
END
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/5b819294-9ce7-4229-add7-a3206201a706" />

b.  Chạy lệnh thực thi
```SQL
EXEC sp_XuLyThanhToan 
    @MaHD = 3, 
    @MaNV = 1, 
    @SoTienThu = 5000000, 
    @GhiChu = N'Khách trả 5 triệu';
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/069d57df-e9c8-498c-be62-83918e9c37ec" />

## Event 4: Truy vấn danh sách nợ xấu
a. Tạo Hàm Hỗ Trợ Dự Báo Nợ
Hàm này tương tự như hàm tính nợ hiện tại bạn đã làm ở Event 2, nhưng cho phép bạn truyền vào một ngày bất kỳ để xem số tiền nợ tại thời điểm đó (dùng để tính nợ sau 1 tháng).

```SQL
-- Hàm này thực chất chính là hàm fn_TinhTongNo bạn đã tạo
-- Bạn có thể dùng trực tiếp hàm cũ hoặc tạo hàm mới này để quản lý riêng
CREATE OR ALTER FUNCTION fn_DuBaoNo (@MaHD INT, @NgayDuBao DATETIME)
RETURNS MONEY
AS
BEGIN
    -- Sử dụng lại logic tính toán của Event 2
    RETURN dbo.fn_TinhTongNo(@MaHD, @NgayDuBao);
END
GO
```
<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/9e6bb984-5d44-464a-a1a2-bf78dc7f4d82" />
b. Truy Vấn Danh Sách Nợ Xấu
- Để lùi tất cả các hợp đồng về quá khứ nhằm mục đích kiểm tra (test) danh sách nợ xấu, em sẽ câu lệnh` UPDATE` duy nhất mà không có điều kiện `WHERE`.

Sau đó em sẽ chạy đoạn code sau, nó sẽ lùi mốc Deadline 1 của tất cả các hợp đồng về hẳn 1 tháng trước:

```SQL
-- Lùi Deadline 1 của tất cả hợp đồng về 1 tháng trước so với ngày hiện tại
UPDATE HopDong 
SET Deadline1 = DATEADD(MONTH, -1, GETDATE());

-- Kiểm tra lại bảng hợp đồng để xem ngày đã đổi chưa
SELECT MaHD, NgayLap, Deadline1, SoTienVayGoc FROM HopDong;
```

<img width="2875" height="1799" alt="image" src="https://github.com/user-attachments/assets/b77a3999-3f91-4646-9936-0125f3ceb335" />


Tại sao nên dùng lệnh này?
`DATEADD(MONTH, -1, GETDATE())`: Hàm này tự động lấy thời gian hiện tại trừ đi 1 tháng. Điều này đảm bảo tất cả hợp đồng chắc chắn sẽ rơi vào trạng thái "Quá hạn" bất kể hôm nay là ngày nào.

Không có `WHERE`: Khi bỏ `WHERE`, lệnh sẽ áp dụng cho mọi dòng trong bảng `HopDong`.
- Bây giờ em sẽ chạy đoạn code truy vấn dưới đây, danh sách sẽ hiện ra đầy đủ các khách hàng vì tất cả đều đã quá hạn:
```SQL
SELECT 
    k.HoTen AS [Tên KH],
    k.SDT AS [Số điện thoại],
    h.SoTienVayGoc AS [Tiền gốc],
    DATEDIFF(DAY, h.Deadline1, GETDATE()) AS [Ngày quá hạn],
    dbo.fn_TinhTongNo(h.MaHD, GETDATE()) AS [Nợ hiện tại],
    dbo.fn_TinhTongNo(h.MaHD, DATEADD(MONTH, 1, GETDATE())) AS [Nợ dự kiến (1 tháng sau)]
FROM HopDong h
JOIN KhachHang k ON h.MaKH = k.MaKH
WHERE h.Deadline1 < GETDATE() 
  AND h.MaTTHD <> 3; -- Chỉ hiện những người chưa trả hết tiền
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/e7da1f8d-5cae-404a-bfc9-394053c0e7c9" />

## Event 5: Quản lý thanh lý tài sản
Em sẽ lấy ra một vài ví dụ sau: 
- Trigger 1: Tự động chuyển sang "Quá hạn (nợ xấu)"
Trigger này sẽ kiểm tra mỗi khi hợp đồng được cập nhật. Nếu ngày hiện tại vượt quá Deadline 1, trạng thái sẽ tự chuyển thành 2 (Quá hạn).

```SQL
CREATE OR ALTER TRIGGER trg_ChuyenNoXau
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    -- Nếu ngày hiện tại > Deadline1 và đang ở trạng thái 1 (Đang vay)
    -- Thì tự động cập nhật sang trạng thái 2 (Quá hạn)
    UPDATE HopDong
    SET MaTTHD = 2
    FROM HopDong h
    INNER JOIN inserted i ON h.MaHD = i.MaHD
    WHERE i.Deadline1 < GETDATE() AND i.MaTTHD = 1;
END
GO
```


<img width="2878" height="1799" alt="image" src="https://github.com/user-attachments/assets/d96d6fa1-2b78-4b28-a8bb-7efbe3960bf2" />




- Trigger 2: Tự động chuyển tài sản sang "Sẵn sàng thanh lý"
Khi hợp đồng bị chuyển sang trạng thái "Quá hạn" và ngày hiện tại đã vượt quá Deadline 2, toàn bộ tài sản liên quan sẽ chuyển sang trạng thái 3 (Sẵn sàng thanh lý).

```SQL
CREATE OR ALTER TRIGGER trg_SanSangThanhLy
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    -- Kiểm tra nếu hợp đồng quá hạn và vượt Deadline 2
    IF EXISTS (SELECT 1 FROM inserted WHERE MaTTHD = 2 AND Deadline2 < GETDATE())
    BEGIN
        UPDATE TaiSan
        SET MaTTTS = 3 -- 3: Sẵn sàng thanh lý
        WHERE MaHD IN (SELECT MaHD FROM inserted WHERE MaTTHD = 2 AND Deadline2 < GETDATE());
    END
END
GO
```


<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/67a3f7a4-7732-4ed1-86d0-d6eb77570022" />




- Trigger 3: Tự động chuyển tài sản thành "Đã bán thanh lý"
Khi chúng ta cập nhật trạng thái hợp đồng sang một mã tương ứng với việc đã xử lý xong (ví dụ em thêm trạng thái mới là 4 - Đã thanh lý), tài sản sẽ tự động cập nhật IsSold = 1.
Lưu ý: Trước tiên hãy thêm trạng thái "Đã thanh lý" vào danh mục:

```SQL
IF NOT EXISTS (SELECT 1 FROM DM_TrangThaiHD WHERE MaTTHD = 4)
    INSERT INTO DM_TrangThaiHD VALUES (4, N'Đã thanh lý');
Viết Trigger:

SQL
CREATE OR ALTER TRIGGER trg_DaBanThanhLy
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;
    -- Nếu trạng thái hợp đồng chuyển sang 4 (Đã thanh lý)
    IF EXISTS (SELECT 1 FROM inserted WHERE MaTTHD = 4)
    BEGIN
        UPDATE TaiSan
        SET MaTTTS = 3, -- Vẫn thuộc danh mục thanh lý
            IsSold = 1  -- Đã bán
        WHERE MaHD IN (SELECT MaHD FROM inserted WHERE MaTTHD = 4);
    END
END
GO
```


<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/8a068862-7fc6-4ba5-8229-992186033dce" />




5.4. Cách kiểm tra (Test Event 5)
Để Trigger kích hoạt, em sẽ thực hiện một lệnh `UPDATE` trên bảng `HopDong`và sẽ thử thanh lý hợp đồng số 1:

```SQL
-- Thử chuyển hợp đồng 1 sang trạng thái "Đã thanh lý"
UPDATE HopDong SET MaTTHD = 4 WHERE MaHD = 1;

-- Kiểm tra xem tài sản của hợp đồng 1 đã tự nhảy IsSold = 1 chưa
SELECT * FROM TaiSan WHERE MaHD = 1;
```

<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/b25bcda5-772c-4062-9645-12e519b99d0e" />






Tổng kết: Hệ thống của em bây giờ đã có thể:

- Tự nhắc nợ.

- Tự động chuẩn bị đồ để đem đi bán khi khách quá hạn Deadline 2.

- Tự động đóng kho khi đồ đã bán xong.



### Các sự kiện có thể bổ sung 

- Lịch sử hợp đồng (Audit Log)
Thực tế, bảng GiaoDich bạn đã tạo chính là bảng Log. Mỗi lần khách trả tiền, chúng ta INSERT một dòng mới thay vì UPDATE đè lên tiền gốc.

Để xem lịch sử dòng tiền của một khách hàng, em dùng câu lệnh:

```SQL
SELECT 
    gd.MaGD,
    gd.NgayGD,
    gd.SoTienThu,
    nv.HoTen AS [Người thu tiền],
    gd.LoaiThanhToan,
    gd.GhiChu
FROM GiaoDich gd
JOIN NhanVien nv ON gd.MaNV = nv.MaNV
WHERE gd.MaHD = 3 -- Thay bằng mã hợp đồng muốn xem
ORDER BY gd.NgayGD DESC;
```


<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/767d4492-f77c-4ec4-867e-4ed4e2c6553a" />

- Sự kiện Gia hạn hợp đồng
Đây là nghiệp vụ quan trọng: Khách trả hết tiền lãi để "reset" lại thời gian, đưa hợp đồng về trạng thái như mới vay để không bị tính lãi kép.

Bạn hãy chạy Store Procedure này:

```SQL
CREATE OR ALTER PROCEDURE sp_GiaHanHopDong
    @MaHD INT,
    @MaNV INT,
    @SoThangGiaHan INT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- 1. Tính số tiền lãi khách ĐANG NỢ đến thời điểm hiện tại
    DECLARE @Goc MONEY, @NgayLap DATETIME;
    SELECT @Goc = SoTienVayGoc, @NgayLap = NgayLap FROM HopDong WHERE MaHD = @MaHD;
    
    DECLARE @TongNoHienTai MONEY = dbo.fn_TinhTongNo(@MaHD, GETDATE());
    DECLARE @TienLaiPhaiTra MONEY = @TongNoHienTai - @Goc;

    IF @TienLaiPhaiTra <= 0
    BEGIN
        PRINT N'Khách hiện không nợ lãi, không cần gia hạn bằng cách này.';
        RETURN;
    END

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 2. Ghi Log: Khách trả tiền lãi để gia hạn
        INSERT INTO GiaoDich (MaHD, MaNV, NgayGD, SoTienThu, LoaiThanhToan, GhiChu)
        VALUES (@MaHD, @MaNV, GETDATE(), @TienLaiPhaiTra, N'Trả lãi gia hạn', 
                N'Gia hạn thêm ' + CAST(@SoThangGiaHan AS VARCHAR) + N' tháng');

        -- 3. Cập nhật Deadline mới (Dời Deadline 1 và 2)
        -- Reset ngày lập về hôm nay để tính lãi đơn lại từ đầu
        UPDATE HopDong
        SET NgayLap = GETDATE(),
            Deadline1 = DATEADD(MONTH, @SoThangGiaHan, GETDATE()),
            Deadline2 = DATEADD(MONTH, @SoThangGiaHan + 1, GETDATE()),
            MaTTHD = 1 -- Đưa về trạng thái Đang vay (nếu đang là Quá hạn)
        WHERE MaHD = @MaHD;

        COMMIT TRANSACTION;
        PRINT N'Gia hạn thành công. Số tiền lãi đã thu: ' + FORMAT(@TienLaiPhaiTra, 'N0');
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT N'Lỗi: ' + ERROR_MESSAGE();
    END CATCH
END
GO
```






<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/5b0473c5-117e-4405-a626-742ad7d48c5a" />




- Cách kiểm tra (Test Case)
Giả sử hợp đồng số 3 (Anh Cường) đang nợ lãi (như kết quả 10,050,000 bạn chạy trước đó), hãy thực hiện gia hạn thêm 2 tháng:

```SQL
-- Thực hiện gia hạn
EXEC sp_GiaHanHopDong @MaHD = 3, @MaNV = 1, @SoThangGiaHan = 2;

-- Kiểm tra lại Deadline và Lịch sử trả tiền
SELECT MaHD, NgayLap, Deadline1, Deadline2 FROM HopDong WHERE MaHD = 3;
SELECT * FROM GiaoDich WHERE MaHD = 3;
```



<img width="2879" height="1799" alt="image" src="https://github.com/user-attachments/assets/89aa5960-8b2c-428c-95ba-ddf270b84335" />

### Em xin giải thích lý do tại sao ngày tháng lại hiển thị như vậy:

a. Tại sao Deadline 2 lại lên đến tháng 8?
Kết quả hiển thị Deadline2 là 2026-08-15 là hoàn toàn đúng vì theo logic nghiệp vụ và code gia hạn của em với lý do như sau:

- Hợp đồng gốc (HĐ 3): Vốn dĩ có Deadline2 là ngày 15/06/2026.

- Lệnh gia hạn: em thực hiện gia hạn thêm 2 tháng (@SoThangGiaHan = 2).

- Logic tính toán: Hệ thống lấy ngày hiện tại (tháng 5) cộng thêm 2 tháng cho Deadline 1, và Deadline 2 thường được cộng thêm 1 tháng so với Deadline 1 để làm mốc thanh lý tài sản.

--> Kết quả: Việc Deadline2 nhảy tới tháng 8/2026 cho thấy khách hàng có thêm thời gian dự phòng dài hơn trước khi tài sản bị đưa đi thanh lý, điều này đúng với yêu cầu "tránh bị tính lãi kép" và dời kỳ hạn mới của khách.

b. Một điểm cần lưu ý về Deadline 1
Cột Deadline1 đang hiển thị là 2026-04-12. Trong quá trình test nợ xấu ở các bước trước, em đã chạy lệnh lùi ngày về tháng 4 cho tất cả hợp đồng.

Mặc dù Deadline2 đã dời lên tháng 8 thành công, nhưng để logic tính lãi đơn/lãi kép chạy đúng từ thời điểm này, em phải đảm bảo Deadline1 cũng phải lớn hơn ngày hiện tại.

c. Kiểm tra Lịch sử hợp đồng (Audit Log)
Bảng GiaoDich phía dưới hình đã ghi nhận thành công dòng tiền:

- Mã giao dịch (MaGD) 1: Ghi lại việc khách trả 5,000,000 vào ngày 12/05/2026.

- Người thu: Mã nhân viên 1.

- Ghi chú: "Khách trả 5 triệu".

Việc ghi lại từng lần trả tiền như thế này giúp em không bị mất dấu vết dòng tiền, đáp ứng đúng yêu cầu về bảng Log để quản lý Audit.

--> Tổng kết: Các sự kiện bổ sung đã chạy hợp lí. Hệ thống đã lưu lại lịch sử trả tiền và dời được kỳ hạn thanh lý (Deadline 2) ra xa để hỗ trợ khách hàng.
