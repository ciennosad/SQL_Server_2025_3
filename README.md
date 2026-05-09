# SQL_Server_2025_3
## Họ và tên: Dương Đình Kiền
## Lớp: K59KMT
## MSSV: k235480106109

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
### Nhiệm vụ 1: Thiết kế CSDL
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



