# BÁO CÁO BÀI TẬP SỐ 3: HỆ QUẢN TRỊ CSDL SQL  QUẢN LÝ CẦM ĐỒ

## Thông tin sinh viên:

**Họ và tên**: Đàm Ngọc Sơn

**Lớp**: K59.KMT.K01

**MSSV**: K23548016061

---

# Phần 1: Thiết kế CSDL

**1**. Vẽ sơ đồ thực thể liên kết ERD.

<P><img width="975" height="548" alt="image" src="https://github.com/user-attachments/assets/b7e3630a-f435-4ba7-8d4f-42e651d5ed96" />
 <I>Hình 1: Sơ đồ thực thể liên kết ERD
</I></P>

**2**. Bảng chuẩn hóa 3NF.
**Chuẩn hóa thực thể Khách hàng**

- Mục tiêu: Tách thông tin cá nhân khách hàng ra khỏi hợp đồng để tránh lặp lại dữ liệu.
<p><img width="972" height="233" alt="image" src="https://github.com/user-attachments/assets/3750d090-8fc4-46c0-9774-3c789c5a7ef9" />
<i>Hình 2: Chuẩn hóa thực thể Khách hàng</i></p>

**Chuẩn hóa thực thể Hợp đồng**

- Mục tiêu: Chỉ lưu trữ các thông tin cốt lõi của khoản vay và tham chiếu tới khách hàng.
<p><img width="972" height="406" alt="image" src="https://github.com/user-attachments/assets/c26efb33-154a-4a6d-a527-7984087f363c" />
<i>Hình 3: Chuẩn hóa thực thể Hợp đồng</i></p>

**Chuẩn hóa thực thể Tài sản**

- Mục tiêu: Quản lý danh mục nhiều tài sản trên một hợp đồng (quan hệ 1-N).
<p><img width="975" height="294" alt="image" src="https://github.com/user-attachments/assets/b5f2e502-5917-4d9c-80e7-9a2c99f4d708" />
<i>Hình 4: Chuẩn hóa thực thể Tài sản</i></p>

**Chuẩn hóa thực thể Nhật ký Giao dịch (Audit Log)**

- Mục tiêu: Lưu trữ biến động trạng thái và tiền nợ mà không làm ảnh hưởng đến dữ liệu gốc của hợp đồng.  
<p><img width="973" height="339" alt="image" src="https://github.com/user-attachments/assets/0fb177d1-91cf-49c6-9c7d-bc7373bb285f" />
<i>Hình 5: Chuẩn hóa thực thể Nhật ký Giao dịch</i></p>

**Chuẩn hóa CSDL Quẩn Lý Cầm Đồ**

<P><img width="623" height="1159" alt="image" src="https://github.com/user-attachments/assets/593222e9-d7b5-4bc6-a5a9-34eaef286098" />
<I>Hình 6: Bảng chuẩn hóa CSDL quản lý cầm đồ</I></P>

# Phần 2: Cài đặt SQL

**Event 1**: Đăng ký hợp đồng mới (Vay tiền)

**Store Procedure** xử lý việc tiếp nhận hợp đồng mới

```
CREATE TYPE TABLE_TaiSanType AS TABLE (
    TenTS NVARCHAR(100),
    GiaTriDinhGia DECIMAL(18, 2)
);
GO
CREATE PROCEDURE sp_RegisterNewContract
    @HoTen NVARCHAR(100),
    @SoDienThoai VARCHAR(15),
    @DiaChi NVARCHAR(255),
    @SoTienVayGoc DECIMAL(18, 2),
    @Deadline1 DATETIME,
    @Deadline2 DATETIME,
    @DanhSachTaiSan TABLE_TaiSanType READONLY 
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRANSACTION;

    BEGIN TRY
        DECLARE @MaKH INT;
        SELECT @MaKH = MaKH FROM KhachHang WHERE SoDienThoai = @SoDienThoai;
        IF @MaKH IS NULL
        BEGIN
            INSERT INTO KhachHang (HoTen, SoDienThoai, DiaChi)
            VALUES (@HoTen, @SoDienThoai, @DiaChi);
            SET @MaKH = SCOPE_IDENTITY();
        END
        DECLARE @MaHD INT;
        INSERT INTO HopDong (MaKH, SoTienVayGoc, NgayLap, Deadline1, Deadline2, TrangThai)
        VALUES (@MaKH, @SoTienVayGoc, GETDATE(), @Deadline1, @Deadline2, N'Đang vay');    
        SET @MaHD = SCOPE_IDENTITY(); 
        INSERT INTO TaiSan (MaHD, TenTS, GiaTriDinhGia, TrangThaiTS, IsSold)
        SELECT @MaHD, TenTS, GiaTriDinhGia, N'Đang cầm cố', 0
        FROM @DanhSachTaiSan;
        INSERT INTO AuditLog (MaHD, NgayTra, SoTienTra, NguoiThu, GhiChu)
        VALUES (@MaHD, GETDATE(), 0, N'Hệ thống', N'Khởi tạo hợp đồng - Vay gốc: ' + CAST(@SoTienVayGoc AS NVARCHAR(50)));
        COMMIT TRANSACTION;
        PRINT N'Đăng ký hợp đồng thành công!';
    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        RAISERROR(@ErrorMessage, 16, 1);
    END CATCH
END;
GO
```

Logic xử lý:
- Tính toàn vẹn dữ liệu: Sử dụng BEGIN TRANSACTION để đảm bảo nếu có lỗi khi lưu tài sản, thông tin khách hàng hoặc hợp đồng sẽ không bị lưu "nửa vời".
- Quản lý khách hàng: Tự động nhận diện khách quen dựa trên số điện thoại để tránh tạo trùng bản ghi trong bảng KhachHang.
- Thiết lập thời hạn: Hai mốc Deadline1 và Deadline2 được truyền trực tiếp vào để làm căn cứ tính lãi đơn và lãi kép cho các bước tiếp theo.
- Ghi nhận lịch sử: Ngay khi tạo hợp đồng, một bản ghi log được sinh ra để đánh dấu dòng tiền khởi đầu của hệ thống.

<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/908bc801-c5a0-4300-bb0b-f75eec6bbcbd" />
<i>Hình 7: Thực thi lệnh thành công </i></p>

**Event 2**: Tính toán công nợ thời gian thực

**Function 1**: fn_CalcMoneyTransaction

- Hàm này tính toán số tiền của một giao dịch cụ thể (thường là khoản vay gốc trong hợp đồng) tính đến ngày mục tiêu.

```
CREATE FUNCTION fn_CalcMoneyTransaction
(
    @MaHD INT, 
    @TargetDate DATETIME
)
RETURNS DECIMAL(18, 2)
AS
BEGIN
    DECLARE @Goc DECIMAL(18, 2);
    DECLARE @DL1 DATETIME, @NgayLap DATETIME;
    DECLARE @TongTien DECIMAL(18, 2);
    DECLARE @SoNgayLaiDon INT, @SoNgayLaiKep INT;
    DECLARE @TiLeLaiHangNgay FLOAT = 0.005; 
    SELECT @Goc = SoTienVayGoc, @DL1 = Deadline1, @NgayLap = NgayLap 
    FROM HopDong WHERE MaHD = @MaHD;
    IF @TargetDate <= @DL1
    BEGIN
        SET @SoNgayLaiDon = DATEDIFF(DAY, @NgayLap, @TargetDate);
        IF @SoNgayLaiDon < 0 SET @SoNgayLaiDon = 0;
        SET @TongTien = @Goc + (@Goc * @TiLeLaiHangNgay * @SoNgayLaiDon);
    END
    ELSE
    BEGIN
        SET @SoNgayLaiDon = DATEDIFF(DAY, @NgayLap, @DL1);
        SET @TongTien = @Goc + (@Goc * @TiLeLaiHangNgay * @SoNgayLaiDon);
        SET @SoNgayLaiKep = DATEDIFF(DAY, @DL1, @TargetDate);
        SET @TongTien = @TongTien * POWER((1 + @TiLeLaiHangNgay), @SoNgayLaiKep);
    END

    RETURN @TongTien;
END;
GO
```
<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4e49fbb0-379f-4e6e-ad82-6f061ef839b8" />
<i>Hình 8: Hàm tính toán số tiền của một giao dịch cụ thể</i></p>

**Function 2**: fn_CalcMoneyContract

- Hàm này tính tổng số tiền khách còn nợ sau khi đã trừ đi các khoản đã trả trong lịch sử giao dịch (Audit Log).

```
CREATE FUNCTION fn_CalcMoneyContract
(
    @MaHD INT,
    @TargetDate DATETIME
)
RETURNS DECIMAL(18, 2)
AS
BEGIN
    DECLARE @TongPhaiTra DECIMAL(18, 2);
    DECLARE @TongDaTra DECIMAL(18, 2);
    SET @TongPhaiTra = dbo.fn_CalcMoneyTransaction(@MaHD, @TargetDate);
    SELECT @TongDaTra = ISNULL(SUM(SoTienTra), 0) 
    FROM AuditLog 
    WHERE MaHD = @MaHD AND NgayTra <= @TargetDate;
    RETURN @TongPhaiTra - @TongDaTra;
END;
GO
```
  
<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/04953968-b602-4932-9c9c-c5facd64b8b0" />
<i>Hình 9: Hàm tính tổng số tiền khách còn nợ sau khi đã trừ đi các khoản đã trả trong lịch sử giao dịch</i></p>


Kiểm tra xem hai hàm fn_CalcMoneyTransaction và fn_CalcMoneyContract hoạt động không:

```
USE QuanLyCamDo;
GO

DECLARE @CheckDate DATETIME = GETDATE();

SELECT TOP 1
    MaHD,
    SoTienVayGoc,
    Deadline1,
    @CheckDate AS NgayKiemTra,
    dbo.fn_CalcMoneyTransaction(MaHD, @CheckDate) AS TongNoPhatSinh,
    dbo.fn_CalcMoneyContract(MaHD, @CheckDate) AS DuNoThucTe
FROM HopDong
WHERE Deadline1 > @CheckDate;

SELECT TOP 1
    MaHD,
    SoTienVayGoc,
    Deadline1,
    @CheckDate AS NgayKiemTra,
    dbo.fn_CalcMoneyTransaction(MaHD, @CheckDate) AS TongNoPhatSinh_CoLaiKep,
    dbo.fn_CalcMoneyContract(MaHD, @CheckDate) AS DuNoThucTe
FROM HopDong
WHERE Deadline1 < @CheckDate AND Deadline2 > @CheckDate;

DECLARE @FutureDate DATETIME = DATEADD(MONTH, 1, GETDATE());

SELECT TOP 1
    MaHD,
    SoTienVayGoc,
    @FutureDate AS NgayDuBao,
    dbo.fn_CalcMoneyContract(MaHD, @FutureDate) AS TongTienPhaiTra_Sau1Thang
FROM HopDong
WHERE TrangThai = N'Đang vay';

```

<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/4af9a5d6-b931-4fd8-850c-b52a226de3cc" />
<i>Hình 10: Kiểm thử thành công 2 hàm </i></p>


**Event 3**: Xử lý trả nợ và hoàn trả tài sản

**Store Procedure** xử lý khi khách mang tiền đến

```
CREATE PROCEDURE sp_ProcessPaymentAndReturnAsset
    @MaHD INT,
    @SoTienKhachTra DECIMAL(18, 2),
    @NguoiThu NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @TargetDate DATETIME = GETDATE();
    DECLARE @TongNoHienTai DECIMAL(18, 2);
    DECLARE @IsAnyAssetSold BIT;
    DECLARE @Deadline2 DATETIME;
    SELECT @Deadline2 = Deadline2 FROM HopDong WHERE MaHD = @MaHD;
    SELECT @IsAnyAssetSold = CASE WHEN EXISTS (SELECT 1 FROM TaiSan WHERE MaHD = @MaHD AND IsSold = 1) THEN 1 ELSE 0 END;

    -- KIỂM TRA ĐIỀU KIỆN THANH LÝ 
    IF @TargetDate > @Deadline2 AND @IsAnyAssetSold = 1
    BEGIN
        PRINT N'Thông báo: Tài sản đã bị thanh lý. Không thu tiền, không trả đồ.';
        RETURN;
    END
    SET @TongNoHienTai = dbo.fn_CalcMoneyContract(@MaHD, @TargetDate);
    DECLARE @DuNoConLai DECIMAL(18, 2) = @TongNoHienTai - @SoTienKhachTra;
    INSERT INTO AuditLog (MaHD, NgayTra, SoTienTra, NguoiThu, GhiChu)
    VALUES (@MaHD, @TargetDate, @SoTienKhachTra, @NguoiThu, 
            N'Trả nợ. Dư nợ còn lại: ' + CAST(@DuNoConLai AS NVARCHAR(50)));

    IF @DuNoConLai <= 0
    BEGIN
        UPDATE HopDong SET TrangThai = N'Đã thanh toán đủ' WHERE MaHD = @MaHD;
        UPDATE TaiSan SET TrangThaiTS = N'Đã trả khách' WHERE MaHD = @MaHD AND IsSold = 0;
        PRINT N'Xác nhận: Đã thanh toán đủ. Đã làm thủ tục trả toàn bộ tài sản.';
    END
    ELSE
    BEGIN
        UPDATE HopDong SET TrangThai = N'Đang trả góp' WHERE MaHD = @MaHD;
        PRINT N'Xác nhận: Đã ghi nhận số tiền trả. Dư nợ còn lại: ' + CAST(@DuNoConLai AS NVARCHAR(50));
        PRINT N'--- DANH SÁCH TÀI SẢN CÓ THỂ TRẢ LẠI NGAY ---';
        
        SELECT MaTS, TenTS, GiaTriDinhGia
        FROM TaiSan
        WHERE MaHD = @MaHD 
          AND TrangThaiTS = N'Đang cầm cố'
          AND @DuNoConLai <= (
              SELECT SUM(GiaTriDinhGia) 
              FROM TaiSan TS2 
              WHERE TS2.MaHD = @MaHD 
                AND TS2.TrangThaiTS = N'Đang cầm cố' 
                AND TS2.MaTS <> TaiSan.MaTS 
          );
    END
END;
GO
```

Trình tự xử lý nghiệp vụ:
- Bước 1: Kiểm tra điều kiện thanh lý:  Hệ thống kiểm tra xem thời gian hiện tại đã vượt quá Deadline 2 chưa và tài sản đã có cờ IsSold (đã bán) chưa.  Nếu tài sản đã bán, hệ thống sẽ chặn và thông báo: "Không thu tiền, không trả đồ".
- Bước 2: Tính toán công nợ thực tế:  Sử dụng hàm fn_CalcMoneyContract để tính tổng nợ (Gốc + Lãi) tính đến đúng thời điểm khách mang tiền đến.
- Bước 3: Cập nhật lịch sử và Trạng thái:
  - Ghi nhận Log : Số tiền trả được ghi vào bảng AuditLog để đảm bảo không mất dấu vết dòng tiền.
  - Phân loại trạng thái:
    - Nếu trả hết: Cập nhật hợp đồng thành "Đã thanh toán đủ" và làm thủ tục trả hết đồ.
    - Nếu trả một phần: Cập nhật trạng thái thành "Đang trả góp" và tính dư nợ còn lại.
- Bước 4: Logic gợi ý trả tài sản:
  - Hệ thống thực hiện so sánh: (Tổng giá trị các món đồ đang cầm - Giá trị món đồ định trả) $\ge$ Dư nợ còn lại.
  - Chỉ những món đồ nào thỏa mãn điều kiện bảo đảm trên mới hiện ra trong danh sách gợi ý để trả cho khách.  
<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/15660cc1-aa72-43f3-b790-9d21cf7049e6" />
<i>Hình 11: Thực thi lệnh thành công </i></p>


Kiểm thử hệ thống:
Yêu cầu 1: Khách hàng đến thanh toán 1 phần
```
DECLARE @MaHD_Run INT = 53; 
DECLARE @SoTienTra DECIMAL(18,2) = 5000000; 
DECLARE @NguoiThucHien NVARCHAR(100) = N'Sơn Admin';

EXEC sp_ProcessPaymentAndReturnAsset 
    @MaHD = @MaHD_Run, 
    @SoTienKhachTra = @SoTienTra, 
    @NguoiThu = @NguoiThucHien;

SELECT * FROM AuditLog WHERE MaHD = @MaHD_Run;
```
<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/a5ddacf9-a27d-456f-8945-910ea36ad1af" />

  <i>Hình 12: Khách hàng thực hiện thanh toán 1 phần</i>
<p/>

Yêu cầu 2: Khách hàng đến tất toán nợ( Trả hết nợ gốc và lãi)

```
DECLARE @MaHD_Run INT = 53;
DECLARE @NguoiThucHien NVARCHAR(100) = N'Sơn Admin';
DECLARE @TongNoToanBo DECIMAL(18,2);

SET @TongNoToanBo = dbo.fn_CalcMoneyContract(@MaHD_Run, GETDATE());

EXEC sp_ProcessPaymentAndReturnAsset 
    @MaHD = @MaHD_Run, 
    @SoTienKhachTra = @TongNoToanBo, 
    @NguoiThu = @NguoiThucHien;

SELECT MaHD, TrangThai FROM HopDong WHERE MaHD = @MaHD_Run;
SELECT MaTS, TenTS, TrangThaiTS FROM TaiSan WHERE MaHD = @MaHD_Run;
```
<p><i>Hình 13: Khách hàng đến tất toán nợ, hệ thống ghi nhận và đã trả lại hết đồ cho khách</i></p>

**Event 4**: Truy vấn danh sách nợ xấu (Nợ khó đòi)

Xuất danh sách các khách hàng đã quá hạn Deadline 1 mà chưa thanh toán xong hợp đồng. Báo cáo yêu cầu hiển thị các thông tin:

- Tên khách hàng và Số điện thoại liên lạc.

- Số tiền vay gốc ban đầu.

- Số ngày đã quá hạn so với mốc Deadline 1.

- Tổng số tiền phải trả thực tế ở thời điểm hiện tại (bao gồm cả lãi kép phát sinh).

- Dự báo tổng số tiền phải trả nếu khách hàng tiếp tục nợ thêm 1 tháng nữa.

Hệ thống cần một công cụ lọc nhanh các hợp đồng "đỏ" để bộ phận thu hồi nợ làm việc. Điểm mấu chốt là phải tính được số tiền nợ tăng lên theo từng ngày (lãi kép) và đưa ra con số dự báo tương lai để nhân viên có thể dùng làm áp lực/cảnh báo khi đàm phán với khách hàng.

```
USE QuanLyCamDo;
GO

DECLARE @NgayHienTai DATETIME = GETDATE();
DECLARE @MotThangSau DATETIME = DATEADD(MONTH, 1, GETDATE());

SELECT 
    KH.HoTen AS [Tên Khách Hàng],
    KH.SoDienThoai AS [Số Điện Thoại],
    HD.SoTienVayGoc AS [Tiền Vay Gốc],
    DATEDIFF(DAY, HD.Deadline1, @NgayHienTai) AS [Số Ngày Quá Hạn],
    CAST(dbo.fn_CalcMoneyContract(HD.MaHD, @NgayHienTai) AS DECIMAL(18,0)) AS [Tổng Nợ Hiện Tại],
    CAST(dbo.fn_CalcMoneyContract(HD.MaHD, @MotThangSau) AS DECIMAL(18,0)) AS [Tổng Nợ Sau 1 Tháng]
FROM KhachHang KH
JOIN HopDong HD ON KH.MaKH = HD.MaKH
WHERE 
    HD.Deadline1 < @NgayHienTai
    AND HD.TrangThai <> N'Đã thanh toán đủ'
ORDER BY [Số Ngày Quá Hạn] DESC;
GO
```
Luồng xử lý chi tiết:
- Bước 1: Thiết lập mốc thời gian: Hệ thống lấy ngày giờ hiện tại của hệ thống (GETDATE) và tính toán mốc thời gian của 30 ngày tới (DATEADD) để phục vụ dự báo.

- Bước 2: Kết hợp dữ liệu (Join): Thực hiện nối bảng KhachHang và HopDong dựa trên mã khách hàng để lấy thông tin định danh và thông tin tài chính cùng lúc.

- Bước 3: Lọc điều kiện quá hạn: Chỉ lấy các bản ghi có ngày Deadline1 nhỏ hơn ngày hiện tại. Đồng thời loại trừ những người đã trả hết nợ (TrangThai <> N'Đã thanh toán đủ').

- Bước 4: Tính toán nợ động:
  - Sử dụng hàm DATEDIFF để trừ ngày, ra kết quả số ngày trễ hạn.
  - Gọi hàm fn_CalcMoneyContract để truyền vào MaHD và Ngày kiểm tra. Hàm này sẽ tự động tính lãi đơn (nếu trong hạn) hoặc lãi kép (nếu quá hạn).

- Bước 5: Định dạng và Sắp xếp: Sử dụng CAST để làm tròn tiền về số nguyên cho dễ nhìn và sắp xếp (ORDER BY) theo số ngày quá hạn giảm dần để đưa những khách hàng nợ lâu nhất lên đầu danh sách.

 <p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f5f59f5c-0798-4a17-9462-18e9a9793f06" />
<i>Hình 14: Danh sách các khách hàng đã quá hạn Deadline 1 mà chưa thanh toán xong hợp đồng </i></p>

**Event 5**: Quản lý thanh lý tài sản

Xây dựng cơ chế tự động cập nhật trạng thái của Hợp đồng và Tài sản dựa trên các mốc thời gian và điều kiện nghiệp vụ:

- Tự động chuyển Hợp đồng sang "Quá hạn (nợ xấu)" khi vượt quá Deadline 1.

- Tự động chuyển Tài sản sang "Sẵn sàng thanh lý" khi Hợp đồng đã quá hạn nợ xấu và vượt mốc Deadline 2.

- Tự động cập nhật Tài sản thành "Đã bán thanh lý" ngay khi trạng thái Hợp đồng chuyển sang "Đã thanh lý".

```
USE QuanLyCamDo;
GO

CREATE TRIGGER trg_UpdateNoxauOnDeadline1
ON HopDong
AFTER UPDATE, INSERT
AS
BEGIN
    UPDATE HopDong
    SET TrangThai = N'Quá hạn (nợ xấu)'
    FROM HopDong
    INNER JOIN inserted i ON HopDong.MaHD = i.MaHD
    WHERE HopDong.TrangThai = N'Đang vay' 
      AND GETDATE() > HopDong.Deadline1;
END;
GO

CREATE TRIGGER trg_ReadyToLiquidateOnDeadline2
ON HopDong
AFTER UPDATE
AS
BEGIN
    UPDATE TaiSan
    SET TrangThaiTS = N'Sẵn sàng thanh lý'
    FROM TaiSan
    INNER JOIN inserted i ON TaiSan.MaHD = i.MaHD
    WHERE i.TrangThai = N'Quá hạn (nợ xấu)' 
      AND GETDATE() > i.Deadline2
      AND TaiSan.TrangThaiTS = N'Đang cầm cố';
END;
GO

CREATE TRIGGER trg_MarkAsSoldOnLiquidated
ON HopDong
AFTER UPDATE
AS
BEGIN
    IF UPDATE(TrangThai)
    BEGIN
        UPDATE TaiSan
        SET TrangThaiTS = N'Đã bán thanh lý',
            IsSold = 1
        FROM TaiSan
        INNER JOIN inserted i ON TaiSan.MaHD = i.MaHD
        WHERE i.TrangThai = N'Đã thanh lý'
          AND TaiSan.TrangThaiTS = N'Sẵn sàng thanh lý';
    END
END;
GO

```
Luồng xử lý chi tiết:

- Trigger 1 (trg_UpdateNoxauOnDeadline1):

  - Kích hoạt: Khi có hợp đồng mới được thêm vào hoặc cập nhật.

  - Xử lý: Kiểm tra nếu ngày hiện tại đã vượt qua Deadline 1 mà hợp đồng vẫn ở trạng thái "Đang vay", nó sẽ tự động chuyển sang "Quá hạn (nợ xấu)". Đây là tiền đề để hệ thống bắt đầu tính lãi kép ở Event 2.

- Trigger 2 (trg_ReadyToLiquidateOnDeadline2):

  - Kích hoạt: Khi trạng thái hợp đồng được cập nhật.

  - Xử lý: Nếu hợp đồng đã là nợ xấu và ngày hiện tại vượt quá Deadline 2, toàn bộ tài sản liên quan đang ở trạng thái "Đang cầm cố" sẽ bị đổi thành "Sẵn sàng thanh lý". Lúc này, nhân viên có thể thấy các món đồ này trong danh mục hàng thanh lý.

- Trigger 3 (trg_MarkAsSoldOnLiquidated):

  - Kích hoạt: Khi quản lý quyết định bán đồ và chuyển trạng thái hợp đồng sang "Đã thanh lý".

  - Xử lý: Sử dụng hàm UPDATE(TrangThai) để tối ưu hiệu năng (chỉ chạy khi cột trạng thái thay đổi). Nó sẽ quét toàn bộ tài sản của hợp đồng đó, chuyển trạng thái sang "Đã bán thanh lý" và bật cờ IsSold = 1.

  - Kết quả: Event 3 sẽ tự động chặn mọi nỗ lực nộp thêm tiền của khách hàng vì đồ đã bán xong.

<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c2b4ac29-b2c8-49e0-9cf9-2a9ee8f875c6" />
<i>Hình 15: Thực thi lệnh </i></p>

Kịch bản:
- Em sẽ chọn một hợp đồng đang bình thường, sau đó "ép" ngày Deadline về quá khứ để Trigger hiểu rằng hợp đồng này đã quá hạn. Tiếp theo, em sẽ chuyển trạng thái hợp đồng để xem tài sản có tự động bị "khóa" hoặc "thanh lý" theo không.

```
USE QuanLyCamDo;
GO

DECLARE @MaHD_Test INT = 55;

UPDATE HopDong 
SET Deadline1 = DATEADD(DAY, -5, GETDATE()), 
    TrangThai = N'Đang vay' 
WHERE MaHD = @MaHD_Test;

SELECT MaHD, Deadline1, TrangThai FROM HopDong WHERE MaHD = @MaHD_Test;

UPDATE HopDong 
SET Deadline2 = DATEADD(DAY, -1, GETDATE()) 
WHERE MaHD = @MaHD_Test;

SELECT MaHD, MaTS, TenTS, TrangThaiTS 
FROM TaiSan 
WHERE MaHD = @MaHD_Test;

UPDATE HopDong 
SET TrangThai = N'Đã thanh lý' 
WHERE MaHD = @MaHD_Test;

SELECT MaHD, MaTS, TenTS, TrangThaiTS, IsSold 
FROM TaiSan 
WHERE MaHD = @MaHD_Test;
GO

```

Luồng xử lý chi tiết:

- Bước 1 (Kích hoạt Trigger 1): Khi bạn chạy lệnh UPDATE đưa Deadline1 về 5 ngày trước, trg_UpdateNoxauOnDeadline1 sẽ ngay lập tức bắt được sự kiện này. Kết quả lệnh SELECT đầu tiên sẽ thấy TrangThai tự nhảy sang "Quá hạn (nợ xấu)" .
- Bước 2 (Kích hoạt Trigger 2): Tiếp theo, em đưa Deadline2 về quá khứ. Lúc này trg_ReadyToLiquidateOnDeadline2 sẽ kiểm tra: Hợp đồng đã là nợ xấu + Đã quá Deadline 2 $\rightarrow$ Nó sẽ tự động quét bảng TaiSan và đổi trạng thái đồ thành "Sẵn sàng thanh lý". 
- Bước 3 (Kích hoạt Trigger 3): Cuối cùng, em giả định đã bán xong đồ bằng cách cập nhật Hợp đồng thành "Đã thanh lý". Ngay lập tức, trg_MarkAsSoldOnLiquidated sẽ chốt hạ bằng cách đổi trạng thái tài sản thành "Đã bán thanh lý" và bật cờ IsSold = 1.


<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/75ff5526-a8ec-4f71-b1a9-961ca2f028f7" />
<i>Hình 17: Trạng thái tài sản tự động thanh lý khi đã quá hạn không thể liên lạc được với khách hàng</i></p>


- Khi khách hàng [maHD55] đến thanh toán nợ nhưng món đồ đó đã bị thanh lý:
```
USE QuanLyCamDo;
GO

DECLARE @MaHD_Cam INT = 55; 
DECLARE @SoTienNop DECIMAL(18,2) = 10000000;
DECLARE @NguoiNhan NVARCHAR(100) = N'Sơn Admin';

UPDATE HopDong SET Deadline2 = DATEADD(DAY, -1, GETDATE()), TrangThai = N'Quá hạn (nợ xấu)' WHERE MaHD = @MaHD_Cam;
UPDATE TaiSan SET TrangThaiTS = N'Đã bán thanh lý', IsSold = 1 WHERE MaHD = @MaHD_Cam;

EXEC sp_ProcessPaymentAndReturnAsset 
    @MaHD = @MaHD_Cam, 
    @SoTienKhachTra = @SoTienNop, 
    @NguoiThu = @NguoiNhan;

SELECT * FROM AuditLog WHERE MaHD = @MaHD_Cam;
GO
```
Luồng xử lý chi tiết:
- Bước 1: Hai lệnh UPDATE đầu tiên sẽ đưa hợp đồng về trạng thái "đã thanh lý" và quan trọng nhất là bật cờ IsSold = 1 trong bảng tài sản.

- Bước 2: Khi lệnh EXEC được gọi, Store Procedure sẽ chạy đoạn kiểm tra điều kiện ngay đầu tiên: IF @TargetDate > @Deadline2 AND @IsSold = 1.

- Bước 3: Vì điều kiện trên thỏa mãn, hệ thống sẽ in ra thông báo lỗi và thực hiện lệnh RETURN.

- Bước 4: Lệnh SELECT * FROM AuditLog cuối cùng sẽ trả về kết quả trống hoặc không có dòng tiền mới nào được thêm vào. Điều này chứng minh giao dịch thu tiền đã bị chặn thành công.

<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/09b1ffde-54e2-4003-9ec0-031df2293726" />
<i>Hình 18: Khi khách hàng đến nộp tiền vào lúc này, hệ thống phải chặn lại vì món đồ không còn trong kho để trả cho khách </i></p>

<p><img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/41615b66-a0a6-4efe-b689-a9bdc414e317" />
<i>Hình 19: Hệ thống báo tài sản đã bị thanh lý, không thể nộp tiền, không còn đồ để trả</i></p>
