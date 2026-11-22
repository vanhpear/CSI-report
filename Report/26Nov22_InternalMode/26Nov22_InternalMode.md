
# Tìm hiểu về Internal Mode của espectre

## 1. Khái niệm
- Internal Mode (Chế độ nội bộ): là chế độ mà **ESP32 tự chủ động tạo lưu lượng mạng Wi-Fi để thu thập dữ liệu CSI**, thay vì phải phụ thuộc vào lưu lượng do thiết bị bên ngoài tạo ra (sinh ra là để đảm bảo esp32 sẽ luôn nhận được gói tin để đo CSI)
## 2. Cách thức hoạt động
 ![540](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(237%2C%20237%2C%20237\)%22%20viewBox%3D%220%200%20340%20182%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%229%22%20y%3D%2216%22%20width%3D%22118%22%20height%3D%2271%22%20rx%3D%2210%22%20fill%3D%22%23E0F2FE%22%20stroke%3D%22%230284C7%22%2F%3E%3Ctext%20x%3D%2268%22%20y%3D%2243%22%20font-size%3D%2215%22%20font-weight%3D%22bold%22%20text-anchor%3D%22middle%22%20fill%3D%22%23075985%22%3EESP32%3C%2Ftext%3E%3Ctext%20x%3D%2268%22%20y%3D%2262%22%20font-size%3D%2211%22%20text-anchor%3D%22middle%22%20fill%3D%22%23075985%22%3EESPectre%3C%2Ftext%3E%3Crect%20x%3D%22212%22%20y%3D%2216%22%20width%3D%22118%22%20height%3D%2271%22%20rx%3D%2210%22%20fill%3D%22%23DCFCE7%22%20stroke%3D%22%2316A34A%22%2F%3E%3Ctext%20x%3D%22271%22%20y%3D%2244%22%20font-size%3D%2215%22%20font-weight%3D%22bold%22%20text-anchor%3D%22middle%22%20fill%3D%22%23166534%22%3ERouter%3C%2Ftext%3E%3Ctext%20x%3D%22271%22%20y%3D%2262%22%20font-size%3D%2211%22%20text-anchor%3D%22middle%22%20fill%3D%22%23166534%22%3EWi-Fi%20AP%3C%2Ftext%3E%3Cpath%20d%3D%22M130%2031%20H205%22%20stroke%3D%22%230284C7%22%20stroke-width%3D%222%22%20fill%3D%22none%22%2F%3E%3Cpath%20d%3D%22M198%2026%20L205%2031%20L198%2036%22%20stroke%3D%22%230284C7%22%20stroke-width%3D%222%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22168%22%20y%3D%2223%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EPing%3C%2Ftext%3E%3Cpath%20d%3D%22M209%2071%20H134%22%20stroke%3D%22%2316A34A%22%20stroke-width%3D%222%22%20fill%3D%22none%22%2F%3E%3Cpath%20d%3D%22M141%2066%20L134%2071%20L141%2076%22%20stroke%3D%22%2316A34A%22%20stroke-width%3D%222%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22168%22%20y%3D%2285%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3EReply%3C%2Ftext%3E%3Cpath%20d%3D%22M68%2089%20V112%20H170%20V130%22%20stroke%3D%22%2364748B%22%20stroke-width%3D%221.6%22%20stroke-dasharray%3D%224%204%22%20fill%3D%22none%22%2F%3E%3Cpath%20d%3D%22M165%20124%20L170%20130%20L175%20124%22%20stroke%3D%22%2364748B%22%20stroke-width%3D%221.6%22%20fill%3D%22none%22%2F%3E%3Crect%20x%3D%2289%22%20y%3D%22133%22%20width%3D%22162%22%20height%3D%2237%22%20rx%3D%228%22%20fill%3D%22%23F3E8FF%22%20stroke%3D%22%239333EA%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22148%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20font-weight%3D%22bold%22%20fill%3D%22%236B21A8%22%3EPh%C3%A2n%20t%C3%ADch%20CSI%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22161%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22%236B21A8%22%3EPh%C3%A1t%20hi%E1%BB%87n%20chuy%E1%BB%83n%20%C4%91%E1%BB%99ng%3C%2Ftext%3E%3C%2Fsvg%3E)
- **Quy trình gồm 4 bước**:

	a. ESP32 kết nối với router Wi-Fi.
	    
	b. ESP32 chủ động gửi các gói tin, ví dụ 100 gói/giây.
	    
	c. Router phản hồi, ESP32 nhận gói và trích xuất CSI.
	    
	d. ESPectre phân tích sự biến đổi CSI để phát hiện chuyển động.
	
## 3. **Internal Mode dùng giao thức gì?** 

- ESPectre hỗ trợ 4 kiểu tạo lưu lượng nội bộ:![[Pasted image 20261122223724.png]]Hiểu đơn giản là ![[Pasted image 20261122224648.png]]
## 4. Phương pháp nào tối ưu nhất?

- Airtime(thuật ngữ): các thiết bị cùng truy cập vào wifi phải chia sẻ thời gian kênh truyền cho nhau (gói tin dài ngắn, hàng đợi, bộ đệm buffer lưu trữ gói tin)
- Bảng so sánh các phương pháp của internal mode đc sử dụng khi nào:![[Pasted image 20261122231621.png]]
###  **Kết luận**

- **Ping**: Lựa chọn mặc định vì đơn giản, dễ triển khai.
    
- **DNS UDP:** Thay thế khi Ping không hoạt động nhưng DNS UDP được hỗ trợ.
    
- **DNS TCP:** Hữu ích khi cần sử dụng DNS qua kết nối TCP.
    
- **Wi-Fi Raw:** Tiết kiệm airtime nhất, giúp giảm tải cho kênh Wi-Fi nếu phần cứng tương thích.

![[Pasted image 20261122232337.png]]
## **5. Kết quả: Chúng ta đã giải quyết được vấn đề gì?**

Ban đầu, ESP32 phải phụ thuộc vào lưu lượng mạng có sẵn nên không kiểm soát được thời điểm nhận những gói tin phù hợp để đo CSI.

Sau khi triển khai Internal Mode:

- ESP32 **chủ động tạo lưu lượng Wi-Fi** theo tốc độ cấu hình.
    
- Nhờ phản hồi từ router/AP, ESP32 có thể thu thập CSI **đều đặn** hơn.
    
- Hệ thống giảm phụ thuộc vào việc có người khác đang sử dụng mạng.
    
- CSI sau đó được xử lý để phục vụ phát hiện chuyển động.