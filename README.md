# baitap3
NHIỆM VỤ 1: THIẾT KẾ VÀ CÀI ĐẶT CƠ SỞ DỮ LIỆU
1.1. Sơ đồ thực thể mối quan hệ (ERD)

Hệ thống quản lý tiệm cầm đồ được thiết kế nhằm quản lý chặt chẽ thông tin khách hàng, các khoản vay và tài sản thế chấp. Cấu trúc CSDL bao gồm 4 bảng chính với các mối quan hệ sau:

Khách hàng - Hợp đồng: Quan hệ 1-n (Một khách hàng có thể lập nhiều hợp đồng vay).

Hợp đồng - Tài sản: Quan hệ 1-n (Một hợp đồng có thể cầm cố nhiều tài sản để đảm bảo khoản vay).

Hợp đồng - Log: Quan hệ 1-n (Lưu vết mọi biến động trả tiền và thay đổi trạng thái của hợp đồng).

<img width="960" height="540" alt="1" src="https://github.com/user-attachments/assets/9d22a804-bbb0-43cb-b0d6-ef1c87fac7a4" />


1.2. Mô tả các bảng dữ liệu (Table Schemas)

Dưới đây là chi tiết cấu trúc các bảng trong hệ thống:

Bảng KhachHang: Lưu trữ thông tin cá nhân của khách hàng (Tên, Số điện thoại).

Bảng HopDong: Lưu thông tin khoản vay, ngày lập và các mốc Deadline để tính lãi suất.

Bảng TaiSan: Quản lý chi tiết tài sản thế chấp và giá trị định giá.

Bảng Log: Ghi lại nhật ký mỗi lần giao dịch, giúp theo dõi dòng tiền chính xác.

1.3. Script cài đặt CSDL (SQL Code)

Sử dụng mã SQL dưới đây để khởi tạo cơ sở dữ liệu và các bảng:

    -- Tạo Database
    CREATE DATABASE QuanLyCamDo;
    GO
    USE QuanLyCamDo;
    
    -- 1. Tạo bảng Khách hàng
    CREATE TABLE KhachHang (
        ID INT PRIMARY KEY IDENTITY(1,1),
        TenKH NVARCHAR(100),
        SDT VARCHAR(15)
    );
    
    -- 2. Tạo bảng Hợp đồng
    CREATE TABLE HopDong (
        ID INT PRIMARY KEY IDENTITY(1,1),
        CustomerID INT FOREIGN KEY REFERENCES KhachHang(ID),
        SoTienVay DECIMAL(18,2),
        NgayLap DATE,
        Deadline1 DATE, -- Mốc bắt đầu tính lãi kép
        Deadline2 DATE, -- Mốc thanh lý tài sản
        TrangThai NVARCHAR(50) DEFAULT N'Đang vay'
    );
    
    -- 3. Tạo bảng Tài sản
    CREATE TABLE TaiSan (
        ID INT PRIMARY KEY IDENTITY(1,1),
        ContractID INT FOREIGN KEY REFERENCES HopDong(ID),
        TenTaiSan NVARCHAR(255),
        GiaTriDinhGia DECIMAL(18,2),
        IsSold BIT DEFAULT 0 -- 0: Còn kho, 1: Đã bán
    );
    
    -- 4. Tạo bảng Log (Nhật ký giao dịch)
    CREATE TABLE Log (
        ID INT PRIMARY KEY IDENTITY(1,1),
        ContractID INT FOREIGN KEY REFERENCES HopDong(ID),
        NgayTra DATE,
        SoTienTra DECIMAL(18,2),
        NoiDung NVARCHAR(255)
    );

<img width="960" height="540" alt="{40D389F1-198A-475D-9B96-7F790ACC47C9}" src="https://github.com/user-attachments/assets/a36ccf64-37c9-406b-ad76-b886c31b8073" />

EVENT 1: ĐĂNG KÝ HỢP ĐỒNG MỚI

2.1. Nội dung báo cáo 

2. Event 1: Thủ tục đăng ký hợp đồng mới (Store Procedure)

Mục tiêu: Tự động hóa quá trình tiếp nhận hồ sơ cầm đồ, bao gồm thông tin khách hàng, số tiền vay và tài sản thế chấp.

Logic xử lý: >   - Kiểm tra khách hàng đã tồn tại hay chưa dựa trên số điện thoại.

Tự động tính toán ngày Deadline 1 (30 ngày sau khi lập) và Deadline 2 (45 ngày sau khi lập).

Ràng buộc dữ liệu: Đảm bảo thông tin hợp đồng và tài sản phải được lưu trữ đồng thời (sử dụng Transaction).

2.2. SQL Script 


    CREATE PROCEDURE sp_RegisterContract
        @CustomerName NVARCHAR(100),
        @Phone VARCHAR(15),
        @LoanAmount DECIMAL(18,2),
        @AssetName NVARCHAR(255),
        @AssetValue DECIMAL(18,2)
    AS
    BEGIN
        BEGIN TRANSACTION;
        BEGIN TRY
            -- 1. Thêm khách hàng nếu chưa có trong hệ thống
            IF NOT EXISTS (SELECT ID FROM KhachHang WHERE SDT = @Phone)
                INSERT INTO KhachHang (TenKH, SDT) VALUES (@CustomerName, @Phone);
            
            DECLARE @CustID INT = (SELECT ID FROM KhachHang WHERE SDT = @Phone);
    
            -- 2. Tạo hợp đồng mới (Tự động tính Deadline)
            INSERT INTO HopDong (CustomerID, SoTienVay, NgayLap, Deadline1, Deadline2, TrangThai)
            VALUES (@CustID, @LoanAmount, GETDATE(), DATEADD(DAY, 30, GETDATE()), DATEADD(DAY, 45, GETDATE()), N'Đang vay');
    
            DECLARE @NewContractID INT = SCOPE_IDENTITY();
    
            -- 3. Thêm tài sản thế chấp
            INSERT INTO TaiSan (ContractID, TenTaiSan, GiaTriDinhGia)
            VALUES (@NewContractID, @AssetName, @AssetValue);
    
            COMMIT TRANSACTION;
            PRINT 'Register Success!';
        END TRY
        BEGIN CATCH
            ROLLBACK TRANSACTION;
            PRINT 'Error: ' + ERROR_MESSAGE();
        END CATCH
    END;

<img width="960" height="540" alt="{B0C9CD6B-4029-487C-AF4B-582B225BE07D}" src="https://github.com/user-attachments/assets/c4a566a8-1b73-40ec-8c32-146e825909c3" />

        EXEC sp_RegisterContract N'Panyasack Vilasack', '0901234567', 10000000, N'iPhone 15 Pro Max', 25000000;

<img width="960" height="540" alt="{2DC0E118-AEB6-4686-A94A-FD6481EAF41E}" src="https://github.com/user-attachments/assets/3929d535-e7fd-463f-8169-806c957e0b28" />

        SELECT * FROM HopDong; SELECT * FROM TaiSan;
        
<img width="960" height="540" alt="{17F0FAE6-2394-43F3-AD81-68C27529F9E1}" src="https://github.com/user-attachments/assets/e9ac0cc2-91e2-4538-b448-6c08f4a67020" />

EVENT 2: THUẬT TOÁN TÍNH LÃI SUẤT (LAI ĐƠN & LÃI KÉP)
3.1. Nội dung báo cáo 

3. Event 2: Xây dựng hàm tính toán công nợ (Scalar Function)

Mục tiêu: Tính toán chính xác tổng dư nợ (Gốc + Lãi) của một hợp đồng tại một thời điểm bất kỳ.

Cơ chế tính lãi: >   - Giai đoạn 1 (Lãi đơn): Từ ngày lập đến ngày Deadline 1. Mức lãi là 0.5%/ngày tính trên số tiền gốc.

Giai đoạn 2 (Lãi kép): Từ sau ngày Deadline 1. Lãi suất 0.5%/ngày tính trên tổng (Tiền gốc + Lãi đơn đã tích lũy).

Giải thuật: Sử dụng hàm DATEDIFF để tính số ngày và hàm POWER để tính lũy thừa cho lãi kép.

3.2. SQL Script 

      CREATE FUNCTION dbo.fn_CalcMoneyContract (@ContractID INT, @TargetDate DATE)
      RETURNS DECIMAL(18,2)
      AS
      BEGIN
          DECLARE @Goc DECIMAL(18,2), @NgayVay DATE, @DL1 DATE, @TongTien DECIMAL(18,2);
          
          -- Lấy thông tin gốc và mốc thời gian từ hợp đồng
          SELECT @Goc = SoTienVay, @NgayVay = NgayLap, @DL1 = Deadline1 
          FROM HopDong WHERE ID = @ContractID;
      
          -- 1. Tính lãi đơn (Áp dụng đến mốc Deadline 1)
          DECLARE @SoNgayDon INT = DATEDIFF(DAY, @NgayVay, CASE WHEN @TargetDate > @DL1 THEN @DL1 ELSE @TargetDate END);
          IF @SoNgayDon < 0 SET @SoNgayDon = 0;
          
          SET @TongTien = @Goc + (@Goc * 0.005 * @SoNgayDon); -- Lãi 0.5%/ngày
      
          -- 2. Tính lãi kép (Nếu ngày hiện tại vượt quá Deadline 1)
          IF @TargetDate > @DL1
          BEGIN
              DECLARE @SoNgayKep INT = DATEDIFF(DAY, @DL1, @TargetDate);
              -- Công thức: A = P * (1 + r)^n
              SET @TongTien = @TongTien * POWER(1 + 0.005, @SoNgayKep);
          END
      
          RETURN @TongTien;
      END;

<img width="960" height="540" alt="{20968766-2093-46AF-8689-0D020D8C3EA0}" src="https://github.com/user-attachments/assets/88140aad-2666-491f-8f23-b110e2b0fe79" />

        SELECT 
            ID, 
            SoTienVay AS TienGoc, 
            dbo.fn_CalcMoneyContract(ID, GETDATE()) AS NoHienTai,
            dbo.fn_CalcMoneyContract(ID, DATEADD(DAY, 40, NgayLap)) AS NoSau40Ngay
        FROM HopDong;

<img width="960" height="540" alt="{BD00A76B-4EFC-412B-B185-E4685D700CC2}" src="https://github.com/user-attachments/assets/e9203bda-9f84-491c-b48f-e985eac4384b" />

Sử dụng Scalar Function giúp tái sử dụng mã nguồn tại nhiều nơi (như báo cáo, trigger, thủ tục thanh toán) mà không cần viết lại công thức lãi suất phức tạp."

EVENT 3: QUẢN LÝ THANH TOÁN VÀ TRẢ TÀI SẢN
3.1. Nội dung báo cáo

3. Event 3: Thủ tục thanh toán và trả tài sản

Mục tiêu: Quản lý việc khách hàng trả tiền gốc/lãi và điều kiện để khách nhận lại tài sản thế chấp.

Quy tắc nghiệp vụ:

Ghi nhận lịch sử mỗi lần trả tiền vào bảng Log (không ghi đè tiền nợ).

Cập nhật trạng thái hợp đồng thành "Đã thanh toán" nếu dư nợ bằng 0.

Quy tắc trả hàng: Chỉ cho phép khách lấy lại tài sản nếu giá trị định giá của các tài sản còn lại vẫn lớn hơn hoặc bằng tổng dư nợ hiện tại.

3.2. SQL Script 

    CREATE PROCEDURE sp_Payment
        @ContractID INT,
        @Amount DECIMAL(18,2),
        @Note NVARCHAR(255)
    AS
    BEGIN
        -- 1. Ghi nhật ký vào bảng Log
        INSERT INTO Log (ContractID, NgayTra, SoTienTra, NoiDung)
        VALUES (@ContractID, GETDATE(), @Amount, @Note);
    
        -- 2. Tính toán dư nợ hiện tại (sử dụng hàm ở Event 2)
        DECLARE @CurrentDebt DECIMAL(18,2) = dbo.fn_CalcMoneyContract(@ContractID, GETDATE());
    
        -- 3. Cập nhật trạng thái nếu đã trả hết nợ
        IF @Amount >= @CurrentDebt
        BEGIN
            UPDATE HopDong SET TrangThai = N'Đã thanh toán' WHERE ID = @ContractID;
            PRINT N'Hợp đồng đã thanh toán đủ.';
        END
        ELSE
        BEGIN
            PRINT N'Đã ghi nhận thanh toán một phần. Dư nợ còn lại ước tính: ' + CAST((@CurrentDebt - @Amount) AS VARCHAR);
        END
    END;


<img width="960" height="540" alt="{FACA6363-91E8-479F-A099-3C3A20E758F6}" src="https://github.com/user-attachments/assets/a3beb0f7-b76c-486b-81c4-a972d434581b" />

        EXEC sp_Payment @ContractID = 1, @Amount = 2000000, @Note = N'Khách trả đợt 1';

<img width="960" height="540" alt="{92EEB552-2ED9-49FA-9DC2-8BE37458101B}" src="https://github.com/user-attachments/assets/9ccef49d-411a-4064-b3c1-0aaf3f979727" />

        Run: SELECT * FROM Log WHERE ContractID = 1;

<img width="960" height="540" alt="{DE652853-E9AB-470D-989C-E064BAF6EE55}" src="https://github.com/user-attachments/assets/7c255414-0f5a-4747-991f-2739930e50e4" />

"Hệ thống kiểm tra điều kiện trả đồ dựa trên tổng giá trị định giá của bảng TaiSan (IsSold = 0) so với kết quả trả về từ hàm fn_CalcMoneyContract. Nếu SUM(GiaTriDinhGia) >= fn_CalcMoneyContract, khách mới được rút bớt tài sản."

EVENT 4: TRUY VẤN DANH SÁCH NỢ XẤU (TABLE-VALUED FUNCTION)

4.1. Nội dung báo cáo 

4. Event 4: Truy vấn danh sách nợ xấu (Table-Valued Function)

Mục tiêu: Liệt kê các khách hàng đã quá hạn Deadline 1 nhưng chưa hoàn thành nghĩa vụ tài chính.

Tính năng: >   - Tính toán số ngày quá hạn của từng hợp đồng.

Sử dụng hàm fn_CalcMoneyContract (đã viết ở Event 2) để hiển thị tổng nợ thực tế (bao gồm cả lãi kép).

Dự báo tổng nợ trong 30 ngày tới để hỗ trợ việc ra quyết định thanh lý tài sản.

4.2. SQL Script (Code ສ້າງ Function)


    CREATE FUNCTION dbo.fn_GetBadDebtList()
    RETURNS TABLE 
    AS
    RETURN (
        SELECT 
            k.TenKH AS [Tên Khách Hàng], 
            k.SDT AS [Số Điện Thoại], 
            h.SoTienVay AS [Tiền Gốc],
            DATEDIFF(DAY, h.Deadline1, GETDATE()) AS [Số Ngày Quá Hạn],
            dbo.fn_CalcMoneyContract(h.ID, GETDATE()) AS [Tổng Nợ Hiện Tại],
            dbo.fn_CalcMoneyContract(h.ID, DATEADD(DAY, 30, GETDATE())) AS [Dự Báo Nợ Sau 30 Ngày]
        FROM KhachHang k
        JOIN HopDong h ON k.ID = h.CustomerID
        WHERE h.Deadline1 < GETDATE() -- Chỉ lấy những hợp đồng đã quá Deadline 1
          AND h.TrangThai <> N'Đã thanh toán' -- Và chưa thanh toán xong
    );

<img width="960" height="540" alt="{4E207E90-7050-4F57-ADF4-4F096E5B928A}" src="https://github.com/user-attachments/assets/2f0d66b8-d272-4bf1-9ae9-b95a4916b212" />


        SELECT * FROM dbo.fn_GetBadDebtList();

<img width="960" height="540" alt="{B9E28D20-16D9-4791-950F-04AE4CB49283}" src="https://github.com/user-attachments/assets/6312194f-1a44-47e8-b713-beb0dd4906f2" />

"Hàm này giúp bộ phận quản lý nhận diện nhanh các hợp đồng rủi ro cao. Việc sử dụng dự báo nợ sau 30 ngày giúp tiệm cầm đồ đánh giá xem giá trị tài sản thế chấp còn đủ để bao phủ khoản nợ trong tương lai hay không, từ đó đưa ra quyết định thanh lý tài sản kịp thời."

EVENT 5: QUẢN LÝ THANH LÝ TÀI SẢN (TRIGGER)

5.1. Nội dung báo cáo 

5. Event 5: Cơ chế tự động hóa trạng thái (Triggers)

Mục tiêu: Đảm bảo tính khách quan và kịp thời trong việc phân loại nợ xấu và xử lý tài sản thế chấp mà không cần thao tác thủ công.

Các Trigger được cài đặt:
- Trạng thái hợp đồng: Tự động chuyển từ "Đang vay" sang "Quá hạn (nợ xấu)" ngay khi ngày hiện tại vượt quá mốc Deadline 1.
- 
- Trạng thái tài sản: Tự động chuyển sang "Sẵn sàng thanh lý" nếu hợp đồng đã ở trạng thái nợ xấu và vượt quá mốc Deadline 2.
- 
- Ghi chú: Hệ thống vẫn theo dõi chặt chẽ từng tài sản theo các trạng thái: đang cầm cố, đã trả khách hoặc đã bán thanh lý.  

5.2. SQL Script (Code ສ້າງ Trigger)


    -- 1. Trigger tự động cập nhật trạng thái Nợ xấu khi quá Deadline 1
    CREATE TRIGGER trg_AutoOverdue
    ON HopDong
    AFTER UPDATE, INSERT
    AS
    BEGIN
        UPDATE HopDong
        SET TrangThai = N'Quá hạn (nợ xấu)'
        FROM HopDong h
        INNER JOIN inserted i ON h.ID = i.ID
        WHERE h.Deadline1 < GETDATE() 
          AND h.TrangThai = N'Đang vay';
    END;
    GO
    
    -- 2. Trigger chuẩn bị thanh lý tài sản khi quá Deadline 2
    CREATE TRIGGER trg_ReadyToSell
    ON HopDong
    AFTER UPDATE
    AS
    BEGIN
        IF EXISTS (SELECT 1 FROM inserted WHERE Deadline2 < GETDATE())
        BEGIN
            UPDATE TaiSan
            SET TenTaiSan = TenTaiSan + N' (Sẵn sàng thanh lý)'
            FROM TaiSan ts
            INNER JOIN inserted i ON ts.ContractID = i.ID
            WHERE i.Deadline2 < GETDATE() AND ts.IsSold = 0;
        END
    END;
    GO

<img width="960" height="540" alt="{8B707BF6-112C-4FBF-A173-D89A09CCB910}" src="https://github.com/user-attachments/assets/48d771cd-8071-4f3d-ae37-153a79460951" />



        UPDATE HopDong SET Deadline1 = '2023-01-01' WHERE ID = 1;
        SELECT * FROM HopDong WHERE ID = 1;

<img width="960" height="540" alt="{D8C6CDC4-6BB3-46AF-A45D-6823F37228FC}" src="https://github.com/user-attachments/assets/03eb8a75-0791-47df-839e-585f1c689973" />


4. CÁC SỰ KIỆN BỔ SUNG (EVENT BỔ SUNG)
4.1. Sự kiện Gia hạn hợp đồng (Gia hạn thời gian vay)

Mục tiêu: Cho phép khách hàng gia hạn hợp đồng bằng cách thanh toán toàn bộ lãi phát sinh tính đến thời điểm hiện tại. Sau khi gia hạn, các mốc Deadline 1 và Deadline 2 sẽ được thiết lập mới để giúp khách hàng tránh bị tính lãi kép.

Logic xử lý (Store Procedure):

Tính toán tổng tiền lãi hiện tại bằng hàm fn_CalcMoneyContract.

Ghi nhận giao dịch trả lãi vào bảng Log.

Cập nhật NgayLap về ngày hiện tại và tính toán lại Deadline 1, Deadline 2.

Chuyển trạng thái hợp đồng trở lại "Đang vay".


    CREATE PROCEDURE sp_RenewContract
        @ContractID INT,
        @NguoiThu NVARCHAR(100)
    AS
    BEGIN
        DECLARE @TongNo DECIMAL(18,2) = dbo.fn_CalcMoneyContract(@ContractID, GETDATE());
        DECLARE @TienGoc DECIMAL(18,2) = (SELECT SoTienVay FROM HopDong WHERE ID = @ContractID);
        DECLARE @TienLai DECIMAL(18,2) = @TongNo - @TienGoc;
    
        -- 1. Ghi nhận việc đóng lãi vào bảng Log
        INSERT INTO Log (ContractID, NgayTra, SoTienTra, NoiDung)
        VALUES (@ContractID, GETDATE(), @TienLai, N'Đóng lãi gia hạn. Người thu: ' + @NguoiThu);
    
        -- 2. Cập nhật lại mốc thời gian cho hợp đồng
        UPDATE HopDong
        SET NgayLap = GETDATE(),
            Deadline1 = DATEADD(DAY, 30, GETDATE()),
            Deadline2 = DATEADD(DAY, 45, GETDATE()),
            TrangThai = N'Đang vay'
        WHERE ID = @ContractID;
    
        PRINT N'Gia hạn thành công. Đã thu lãi: ' + CAST(@TienLai AS VARCHAR);
    END;

<img width="960" height="540" alt="{72AD67DB-59DF-428C-BD72-4AA1DEA13C35}" src="https://github.com/user-attachments/assets/ff3f8101-3b58-45e1-9260-cd413257574d" />


4.2. Lịch sử hợp đồng (Audit Log)

Mục tiêu: Đảm bảo tính minh bạch trong quản lý dòng tiền. Thay vì ghi đè số dư nợ trực tiếp vào bảng Hợp đồng, mọi giao dịch thanh toán (dù là trả một phần hay trả hết) đều được lưu lại chi tiết tại bảng Log.

Ý nghĩa nghiệp vụ: > - Lưu trữ chính xác: Ngày trả, Số tiền trả và Người thu tiền.

Tránh mất dấu vết dòng tiền (Audit Trail), giúp kế toán đối soát dữ liệu dễ dàng.


    -- Truy vấn lịch sử trả tiền của một hợp đồng cụ thể
    SELECT 
        L.NgayTra AS [Ngày giao dịch],
        L.SoTienTra AS [Số tiền đóng],
        L.NoiDung AS [Ghi chú/Người thu],
        H.SoTienVay AS [Số tiền gốc ban đầu]
    FROM Log L
    JOIN HopDong H ON L.ContractID = H.ID
    WHERE H.ID = 1 -- Thay bằng ID hợp đồng cần kiểm tra
    ORDER BY L.NgayTra DESC;

<img width="960" height="540" alt="{22803FB1-9CAE-43DB-93F1-18233B2828FA}" src="https://github.com/user-attachments/assets/16715dc3-982a-47cd-8889-f77272aa1ff2" />


