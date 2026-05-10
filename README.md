# SQL_Server_2025_3
 **Họ và tên: Dương Đình Kiền**
 
  **Lớp: K59KMT**
  
  **MSSV: k235480106109**

## Đề bài: Hệ Thống Quản Lý Cầm Đồ


## Phần 1: Mô tả bài toán

- Xây dựng CSDL quản lý hệ thống cầm đồ, bao gồm: Quản lý khách hàng, hợp đồng vay thế chấp tài sản, tính lãi linh hoạt (lãi đơn trước hạn 1, lãi kép sau hạn 1), xử lý trả nợ từng phần, hoàn tài sản theo điều kiện thế chấp, và tự động chuyển trạng thái khi quá hạn/thanh lý.

## Phần 2: Các quy tắc nghiệm vụ

- Khách hàng và tài sản: 1 khách hàng có thể có nhiều hợp đồng cầm cố và mỗi hợp đồng có thể nhiều tài sản thế chấp khác nhau.
- Lãi đơn: 5.000đ/1tr/ngày
- Lãi kép: Tính trên gốc + lãi đơn
- Trả nợ từng phần, hoàn tài sản khi Giá trị tài sản lớn hơn hoặc bằng nhau dư nợ
- Trạng thái hợp đồng: Đang vay, Quá hạn, Đã thanh toán, Đã thanh lý, Đang trả góp
- Theo dõi lịch sử biến động tiền/Trạng thái

## Phần 3: Các yêu cầu nhiệm vụ
### 3.1: Thiết kế CSDL
  Sơ đồ ERD:
(Thiếu hình ảnh sơ đồ)

**Bước 1: Xác định thực thể.**

- KHACH_HANG (KHách hàng)
- HOP_DONG (Hợp đồng)
- CHI_TIET_CAM_DO (Tài sản cầm cố)
- LICH_SU_GIAO_DICH (Log giao dịch)

**Bước 2: Xác định thuộc tính**
**KHACH_HANG**
- MaKH (Khóa chính - PK)
- TenKH
- SDT
- CCCD

**HOP_DONG**
- MaHD (PK)
- MaKH (Khóa ngoại - FK)
- SoTienVay
- NgayLayTien
- DEADLINE 1
- DEADLINE 2
- TrangThaiHD

**CHI_TIET_CAM_DO**
- MaCT (PK)
- MaHD (FK)
- TenTaiSan
- GiaTriDinhGia
- TrangThaiTS

**LICH_SU_GIAO_DICH**
- MaGD (PK)
- MaHD (FK)
- NgayGD
- SoTienTra
- NguoiThu

**Bước 3: Xác định mối quan hệ**
- 1 Khách hàng → có N Hợp đồng (1-N)
- 1 Hợp đồng → có N Tài sản (1-N)
- 1 Hợp đồng → có N Lịch sử giao dịch (1-N)


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/ffb82105-820b-446d-b613-639d94c0dcd0" />
Hình ảnh đã tạo thành công các bảng

## 3.2 Cài đặt SQL
- 1. Đăng ký hợp đồng mới (Vay tiền)

```


IF OBJECT_ID('sp_DangKyHopDong', 'P') IS NOT NULL
    DROP PROCEDURE sp_DangKyHopDong;
GO

CREATE PROCEDURE sp_DangKyHopDong
    @TenKH NVARCHAR(100),
    @SDT VARCHAR(15),
    @CCCD VARCHAR(12),
    @SoTienVay DECIMAL(15,2),
    @Deadline1 DATE,
    @Deadline2 DATE,
    @DanhSachTS NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @MaKH INT;
    DECLARE @MaHD INT;
    DECLARE @IsNewCustomer BIT = 0;
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- 1. Kiểm tra hoặc tạo Khách hàng
        SELECT @MaKH = MaKH FROM KHACHHANG WHERE CCCD = @CCCD;
        
        IF @MaKH IS NULL
        BEGIN
            INSERT INTO KHACHHANG (TenKH, SDT, CCCD)
            VALUES (@TenKH, @SDT, @CCCD);
            
            SET @MaKH = SCOPE_IDENTITY();
            SET @IsNewCustomer = 1;
        END
        
        -- 2. Tạo Hợp đồng
        INSERT INTO HOPDONG (MaKH, SoTienVay, Deadline1, Deadline2)
        VALUES (@MaKH, @SoTienVay, @Deadline1, @Deadline2);
        
        SET @MaHD = SCOPE_IDENTITY();
        
        -- 3. Insert tài sản từ JSON
        INSERT INTO CHI_TIET_CAMDO (MaHD, TenTaiSan, GiaTriDinhGia)
        SELECT 
            @MaHD,
            JSON_VALUE(value, '$.ten') AS TenTaiSan,
            CAST(JSON_VALUE(value, '$.gia') AS DECIMAL(15,2)) AS GiaTriDinhGia
        FROM OPENJSON(@DanhSachTS);
        
        COMMIT TRANSACTION;
        
        
        -- Thông báo kết quả
        SELECT 
            @MaHD AS MaHopDongMoi,
            CASE WHEN @IsNewCustomer = 1 THEN N'Tạo mới khách hàng và hợp đồng thành công'
                 ELSE N'Khách hàng đã tồn tại, tạo hợp đồng mới thành công'
            END AS ThongBao;
        
        -- Thông tin khách hàng
        SELECT 
            MaKH,
            TenKH,
            SDT,
            CCCD,
            DiaChi,
            NgayDangKy,
            CASE WHEN @IsNewCustomer = 1 THEN N'Khách hàng mới' ELSE N'Khách hàng cũ' END AS LoaiKhach
        FROM KHACHHANG
        WHERE MaKH = @MaKH;
        
        -- Thông tin hợp đồng
        SELECT 
            hd.MaHD,
            hd.MaKH,
            kh.TenKH,
            hd.SoTienVay,
            hd.Deadline1,
            hd.Deadline2,
            hd.TrangThaiHD,
            hd.NgayLap,
            DATEDIFF(DAY, GETDATE(), hd.Deadline1) AS SoNgayConLaiDenHan1
        FROM HOPDONG hd
        INNER JOIN KHACHHANG kh ON hd.MaKH = kh.MaKH
        WHERE hd.MaHD = @MaHD;
        
        -- Danh sách tài sản cầm cố
        SELECT 
            MaCT,
            MaHD,
            TenTaiSan,
            GiaTriDinhGia,
            TrangThaiTS,
            FORMAT(GiaTriDinhGia, 'N0', 'vi-vn') AS GiaTriDinhGiaFormat
        FROM CHI_TIET_CAMDO
        WHERE MaHD = @MaHD;
        
        -- 5. Tổng hợp thông tin
        SELECT 
            @MaHD AS MaHopDong,
            (SELECT TenKH FROM KHACHHANG WHERE MaKH = @MaKH) AS TenKhachHang,
            @SoTienVay AS SoTienVay,
            (SELECT COUNT(*) FROM CHI_TIET_CAMDO WHERE MaHD = @MaHD) AS SoLuongTaiSan,
            (SELECT SUM(GiaTriDinhGia) FROM CHI_TIET_CAMDO WHERE MaHD = @MaHD) AS TongGiaTriTaiSan,
            @Deadline1 AS HanThanhToan1,
            @Deadline2 AS HanThanhToan2;
            
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR (@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END



```

<img width="1916" height="1079" alt="image" src="https://github.com/user-attachments/assets/7e842672-5a07-470b-a850-6a4c33b65e2e" />

Hình ảnh kết quả chạy thành công

<img width="1918" height="1076" alt="image" src="https://github.com/user-attachments/assets/30e92b10-3618-482f-a4d8-dbfe897ef566" />

Hình ảnh thêm dữ liệu người cấm cố tài sản

- 2. Tính toán công nợ thời gian thực








  ```





-- Tính số tiền cần trả cho hợp đồng tại ngày mục tiêu
IF OBJECT_ID('fn_CalcMoneyContract', 'FN') IS NOT NULL
    DROP FUNCTION fn_CalcMoneyContract;
GO

CREATE FUNCTION fn_CalcMoneyContract(
    @MaHD INT, 
    @TargetDate DATE
) 
RETURNS DECIMAL(15,2)
AS
BEGIN
    DECLARE @Goc DECIMAL(15,2);
    DECLARE @Deadline1 DATE;
    DECLARE @NgayLap DATE;
    DECLARE @D1_days INT;
    DECLARE @After_D1_days INT;
    DECLARE @LaiDon DECIMAL(15,2);
    DECLARE @TienCanTra DECIMAL(15,2);
    DECLARE @Rate DECIMAL(5,4) = 0.005; -- 0.5%/ngày
    
    -- Lấy thông tin hợp đồng
    SELECT @Goc = SoTienVay, @Deadline1 = Deadline1, @NgayLap = NgayLap
    FROM HOPDONG WHERE MaHD = @MaHD;
    
    -- Nếu không tìm thấy hợp đồng, trả về 0
    IF @Goc IS NULL 
        RETURN 0; 
    
    -- Tính số ngày từ ngày lập đến min(TargetDate, Deadline1)
    SET @D1_days = DATEDIFF(DAY, @NgayLap, 
        CASE WHEN @TargetDate < @Deadline1 THEN @TargetDate ELSE @Deadline1 END);
    
    IF @D1_days < 0 
        SET @D1_days = 0; 
    
    -- Tính lãi đơn cho giai đoạn 1
    SET @LaiDon = @Goc * @Rate * @D1_days;
    
    -- Nếu TargetDate vượt Deadline1 -> tính lãi kép cho giai đoạn 2
    IF @TargetDate > @Deadline1 
    BEGIN
        SET @After_D1_days = DATEDIFF(DAY, @Deadline1, @TargetDate);
        SET @TienCanTra = (@Goc + @LaiDon) * POWER(1 + @Rate, @After_D1_days);
    END
    ELSE
    BEGIN
        SET @TienCanTra = @Goc + @LaiDon;
    END
    
    RETURN ROUND(@TienCanTra, 2);
END
GO


------------------------------------------------------------------------------------------------------------


-- FUNCTION 2: fn_CalcMoneyTransaction
-- Wrapper gọi fn_CalcMoneyContract cho đồng nhất nghiệp vụ
IF OBJECT_ID('fn_CalcMoneyTransaction', 'FN') IS NOT NULL
    DROP FUNCTION fn_CalcMoneyTransaction;
GO

CREATE FUNCTION fn_CalcMoneyTransaction(
    @TransactionID INT, 
    @TargetDate DATE
) 
RETURNS DECIMAL(15,2)
AS
BEGIN
    -- Gọi function chính để tính toán
    RETURN dbo.fn_CalcMoneyContract(@TransactionID, @TargetDate);
END
GO








```





<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/af930d0e-b6c3-43d2-b104-93a37cee3fa8" />


Hình ảnh chương trình tính lãi chạy thành công






```






SELECT dbo.fn_CalcMoneyContract(81, GETDATE()) AS SoTienCanTra;
                                 
SELECT dbo.fn_CalcMoneyContract(81, '2026-08-01') AS SoTienCanTra_01_08;


--===================================================================================

SELECT dbo.fn_CalcMoneyContract(80, GETDATE()) AS SoTienCanTra;
                                 
SELECT dbo.fn_CalcMoneyContract(80, '2026-08-10') AS SoTienCanTra_10_08;


SELECT 
    MaHD,
    SoTienVay,
    NgayLap,
    Deadline1,
    GETDATE() AS NgayHienTai,
    DATEDIFF(DAY, NgayLap, GETDATE()) AS SoNgayDaQua,
    dbo.fn_CalcMoneyContract(MaHD, GETDATE()) AS SoTienCanTraHienTai
FROM HOPDONG;




```




<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/b1a95f40-9aa7-4044-97bf-de8b09b9aac1" />

Hình ảnh tính lãi từng ngày theo gợi ý (Gốc + Lãi đơn + Lãi kép)


- 3.3 Xử lý trả nợ và hoàn trả tài sản





```




-- STORED PROCEDURE: sp_XuLyTraNo
-- Xử lý thanh toán nợ và cập nhật trạng thái

IF OBJECT_ID('sp_XuLyTraNo', 'P') IS NOT NULL
    DROP PROCEDURE sp_XuLyTraNo;
GO

CREATE PROCEDURE sp_XuLyTraNo
    @MaHD INT,
    @SoTienTra DECIMAL(15,2),
    @NguoiThu NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    
    DECLARE @DuNoHienTai DECIMAL(15,2);
    DECLARE @TongNoCanTra DECIMAL(15,2);
    DECLARE @DuSauKhiTra DECIMAL(15,2);
    DECLARE @TrangThai NVARCHAR(50);
    DECLARE @TongGiaTriTaiSan DECIMAL(15,2);
    
    BEGIN TRANSACTION;
    
    BEGIN TRY
        -- 1. Kiểm tra nếu đã thanh lý
        SELECT @TrangThai = TrangThaiHD FROM HOPDONG WHERE MaHD = @MaHD;
        
        IF @TrangThai = N'Đã thanh lý tài sản'
        BEGIN
            RAISERROR(N'Tài sản đã bị thanh lý. Không thu tiền, không trả đồ.', 16, 1);
            ROLLBACK TRANSACTION;
            RETURN;
        END
        
        -- 2. Tính tổng nợ đến hôm nay
        SET @TongNoCanTra = dbo.fn_CalcMoneyContract(@MaHD, CAST(GETDATE() AS DATE));
        SET @DuSauKhiTra = @TongNoCanTra - @SoTienTra;
        
        IF @DuSauKhiTra < 0 
            SET @DuSauKhiTra = 0;
        
        -- 3. Ghi log giao dịch
        INSERT INTO LICH_SU_GIAO_DICH (MaHD, SoTienTra, NguoiThu, DuNoConLai, GhiChu)
        VALUES (@MaHD, @SoTienTra, @NguoiThu, @DuSauKhiTra, N'Thanh toán nợ');
        
        -- 4. Cập nhật trạng thái & hoàn tài sản nếu đủ điều kiện
        IF @DuSauKhiTra = 0
        BEGIN
            UPDATE HOPDONG SET TrangThaiHD = N'Đã thanh toán' WHERE MaHD = @MaHD;
            UPDATE CHI_TIET_CAMDO SET TrangThaiTS = N'Đã trả khách' WHERE MaHD = @MaHD;
            
            SELECT N'Đã thanh toán đủ. Trả toàn bộ tài sản.' AS ThongBao;
        END
        ELSE
        BEGIN
            UPDATE HOPDONG SET TrangThaiHD = N'Đang trả góp' WHERE MaHD = @MaHD;
            
            -- 5. Tính tổng giá trị tài sản đang cầm cố
            SELECT @TongGiaTriTaiSan = SUM(GiaTriDinhGia) 
            FROM CHI_TIET_CAMDO 
            WHERE MaHD = @MaHD AND TrangThaiTS = N'Đang cầm cố';
            
            -- Gợi ý tài sản được trả lại nếu tổng giá trị ≥ dư nợ
            IF @TongGiaTriTaiSan >= @DuSauKhiTra
            BEGIN
                SELECT 
                    c.MaCT,
                    c.TenTaiSan,
                    c.GiaTriDinhGia,
                    FORMAT(c.GiaTriDinhGia, 'N0', 'vi-vn') AS GiaTriFormat
                FROM CHI_TIET_CAMDO c
                WHERE c.MaHD = @MaHD 
                    AND c.TrangThaiTS = N'Đang cầm cố';
            END
            ELSE
            BEGIN
                SELECT N'Chưa đủ giá trị tài sản để trả lại. Tiếp tục cầm cố toàn bộ.' AS ThongBao;
            END
        END
        
        -- 6. Hiển thị thông tin cập nhật
        SELECT 
            @MaHD AS MaHopDong,
            @TongNoCanTra AS TongNoTruocKhiTra,
            @SoTienTra AS SoTienVuaTra,
            @DuSauKhiTra AS DuNoConLai,
            FORMAT(@DuSauKhiTra, 'N0', 'vi-vn') AS DuNoConLai_VND,
            CASE WHEN @DuSauKhiTra = 0 THEN N'Đã thanh toán' ELSE N'Đang trả góp' END AS TrangThaiMoi;
        
        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR (@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END
GO


```


<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/8960936c-1cc9-40d6-9f20-9f5b54ad3ccc" />


Hình ảnh chạy chương trình thành công


<img width="1916" height="1079" alt="image" src="https://github.com/user-attachments/assets/a0af6c5d-29c0-493a-82c5-d020ecda8eda" />

Hình ảnh kết quả trả nợ 1 phần và thanh toán toàn bộ số gốc và lãi


- 3.4 Truy vấn danh sách nợ xấu (Nợ Khó đòi)

**Thêm thông tin người vay**

```


DECLARE @json2 NVARCHAR(MAX) = N'[{"ten": "Điện thoại Samsung A54", "gia": 5000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Trần Thị Bích',
    @SDT = '0912223344',
    @CCCD = '079088002222',
    @SoTienVay = 15000000,
    @Deadline1 = '2026-03-15',
    @Deadline2 = '2026-09-15',
    @DanhSachTS = @json2,
    @NgayDangKy = '2025-09-15';


DECLARE @json3 NVARCHAR(MAX) = N'[{"ten": "Laptop Dell Inspiron", "gia": 12000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Lê Văn Cường',
    @SDT = '0923334455',
    @CCCD = '012345678999',
    @SoTienVay = 18000000,
    @Deadline1 = '2026-04-20',
    @Deadline2 = '2026-10-20',
    @DanhSachTS = @json3,
    @NgayDangKy = '2025-10-20';



DECLARE @json4 NVARCHAR(MAX) = N'[{"ten": "TV Sony 55 inch", "gia": 10000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Phạm Thị Dung',
    @SDT = '0934445566',
    @CCCD = '036077003333',
    @SoTienVay = 25000000,
    @Deadline1 = '2026-08-15',
    @Deadline2 = '2027-02-15',
    @DanhSachTS = @json4,
    @NgayDangKy = '2026-02-15';


DECLARE @json5 NVARCHAR(MAX) = N'[{"ten": "Tủ lạnh Panasonic", "gia": 8000000}, {"ten": "Máy giặt LG", "gia": 7000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Hoàng Văn Em',
    @SDT = '0945556677',
    @CCCD = '040088004444',
    @SoTienVay = 12000000,
    @Deadline1 = '2026-09-20',
    @Deadline2 = '2027-03-20',
    @DanhSachTS = @json5,
    @NgayDangKy = '2026-03-20';




DECLARE @json7 NVARCHAR(MAX) = N'[{"ten": "Xe máy SH Mode", "gia": 60000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Đỗ Văn Giang',
    @SDT = '0967778899',
    @CCCD = '064011006666',
    @SoTienVay = 50000000,
    @Deadline1 = '2026-11-11',
    @Deadline2 = '2027-05-11',
    @DanhSachTS = @json7,
    @NgayDangKy = '2026-05-11';



DECLARE @json8 NVARCHAR(MAX) = N'[{"ten": "Vàng SJC 5 chỉ", "gia": 250000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Bùi Thị Hương',
    @SDT = '0978889900',
    @CCCD = '075022007777',
    @SoTienVay = 50000000,
    @Deadline1 = '2026-12-01',
    @Deadline2 = '2027-06-01',
    @DanhSachTS = @json8,
    @NgayDangKy = '2026-06-01';

DECLARE @json9 NVARCHAR(MAX) = N'[{"ten": "Nhẫn kim cương", "gia": 30000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Ngô Văn Inh',
    @SDT = '0989990011',
    @CCCD = '083033008888',
    @SoTienVay = 40000000,
    @Deadline1 = '2027-01-25',
    @Deadline2 = '2027-07-25',
    @DanhSachTS = @json9,
    @NgayDangKy = '2026-07-25';

DECLARE @json10 NVARCHAR(MAX) = N'[{"ten": "MacBook Air M2", "gia": 20000000}, {"ten": "iPad Pro", "gia": 15000000}]';
EXEC sp_DangKyHopDong 
    @TenKH = N'Mai Thị Khanh',
    @SDT = '0990001122',
    @CCCD = '091044009999',
    @SoTienVay = 35000000,
    @Deadline1 = '2027-02-28',
    @Deadline2 = '2027-08-28',
    @DanhSachTS = @json10,
    @NgayDangKy = '2026-08-28';




```



<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/ad7891ed-1085-47db-ae9b-4027f90cc0e6" />

Hình ảnh thêm danh sách có người vay thanh toán 1 phần, đã thanh toán và quá hạn thanh toán




```



SELECT 
    k.TenKH AS [Tên KH],
    k.SDT AS [Số điện thoại],
    FORMAT(h.SoTienVay, 'N0', 'vi-vn') AS [Tiền vay gốc (VNĐ)],
    
    
    GETDATE() AS [Thời gian kiểm tra],
    
    
    CASE 
        WHEN h.Deadline1 < CAST(GETDATE() AS DATE) 
        THEN DATEDIFF(DAY, h.Deadline1, CAST(GETDATE() AS DATE))
        ELSE 0 
    END AS [Số ngày quá hạn],
    
    
    FORMAT(dbo.fn_CalcMoneyContract(h.MaHD, CAST(GETDATE() AS DATE)), 'N0', 'vi-vn') AS [Tổng nợ hiện tại (VNĐ)],
    
   
    FORMAT(dbo.fn_CalcMoneyContract(h.MaHD, DATEADD(MONTH, 1, CAST(GETDATE() AS DATE))), 'N0', 'vi-vn') AS [Tổng nợ sau 1 tháng (VNĐ)],
    
   
    h.TrangThaiHD AS [Trạng thái],
    h.Deadline1 AS [Hạn thanh toán 1]
    
FROM HOPDONG h
INNER JOIN KHACHHANG k ON h.MaKH = k.MaKH
WHERE h.Deadline1 < CAST(GETDATE() AS DATE)
  AND h.TrangThaiHD NOT IN (N'Đã thanh toán', N'Đã thanh lý tài sản');



````


<img width="1915" height="1079" alt="image" src="https://github.com/user-attachments/assets/409104f0-b1c1-477c-a82b-204b6f433beb" />

Hình ảnh kết quả danh sách những người vay quá hạn và số tiền cần phải thanh toán sau 1 tháng











