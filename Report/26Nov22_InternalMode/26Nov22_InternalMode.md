
# Tìm hiểu về Internal Mode của espectre

## 1. Khái niệm
- Internal Mode (Chế độ nội bộ): là chế độ mà **ESP32 tự chủ động tạo lưu lượng mạng Wi-Fi để thu thập dữ liệu CSI**, thay vì phải phụ thuộc vào lưu lượng do thiết bị bên ngoài tạo ra (sinh ra là để đảm bảo esp32 sẽ luôn nhận được gói tin để đo CSI)
## 2. Cách thức hoạt động
![](ảnh%202.png)
- **Quy trình gồm 4 bước**:

	a. ESP32 kết nối với router Wi-Fi.
	    
	b. ESP32 chủ động gửi các gói tin, ví dụ 100 gói/giây.
	    
	c. Router phản hồi, ESP32 nhận gói và trích xuất CSI.
	    
	d. ESPectre phân tích sự biến đổi CSI để phát hiện chuyển động.
	
## 3. **Internal Mode dùng giao thức gì?** 

- ESPectre hỗ trợ 4 kiểu tạo lưu lượng nội bộ:![](ảnh%203.1.png)
- Hiểu đơn giản là:
- ![895](ảnh%203.2.png)
## 4. Phương pháp nào tối ưu nhất?

- Airtime(thuật ngữ): các thiết bị cùng truy cập vào wifi phải chia sẻ thời gian kênh truyền cho nhau (gói tin dài ngắn, hàng đợi, bộ đệm buffer lưu trữ gói tin)
- Bảng so sánh các phương pháp của internal mode đc sử dụng khi nào:![](ảnh%204.1.png)
###  **Kết luận**

- **Ping**: Lựa chọn mặc định vì đơn giản, dễ triển khai.
    
- **DNS UDP:** Thay thế khi Ping không hoạt động nhưng DNS UDP được hỗ trợ.
    
- **DNS TCP:** Hữu ích khi cần sử dụng DNS qua kết nối TCP.
    
- **Wi-Fi Raw:** Tiết kiệm airtime nhất, giúp giảm tải cho kênh Wi-Fi nếu phần cứng tương thích.

![](ảnh%204.2.png)
## **5. Kết quả: Chúng ta đã giải quyết được vấn đề gì?**

Ban đầu, ESP32 phải phụ thuộc vào lưu lượng mạng có sẵn nên không kiểm soát được thời điểm nhận những gói tin phù hợp để đo CSI.

Sau khi triển khai Internal Mode:

- ESP32 **chủ động tạo lưu lượng Wi-Fi** theo tốc độ cấu hình.
    
- Nhờ phản hồi từ router/AP, ESP32 có thể thu thập CSI **đều đặn** hơn.
    
- Hệ thống giảm phụ thuộc vào việc có người khác đang sử dụng mạng.
    
- CSI sau đó được xử lý để phục vụ phát hiện chuyển động.
