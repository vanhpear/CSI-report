---
title: "Cách CLI Python của ESPectre điều khiển thiết bị, thu thập CSI và biên dịch firmware"
author: null
date: 2026-11-19
project: "ESPectre"
status: draft
tags: [bao-cao, python, cli]
---

# Cách CLI Python của ESPectre điều khiển thiết bị, thu thập CSI và biên dịch firmware

**Người thực hiện:** Lê Việt Anh · **Ngày thực hiện:** 19/11/2026

## 1. Vấn đề và mục tiêu giải quyết

Từ một công cụ dòng lệnh (CLI) là `espectre`, người dùng có thể làm việc với bốn biến thể firmware và giao tiếp với cảm biến theo nhiều cách. Khi một phiên ghi dữ liệu gặp lỗi, người bảo trì cần xác định đoạn mã nào đã chọn thiết bị, kết nối nào đã truyền mẫu và tệp đã lưu có vượt qua bước kiểm tra chất lượng hay không.

## 2. Các yêu cầu cần thiết trước

| Công việc                                 | Điều kiện cần có                                                                                                                                                                                                                                                                                                       |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Đọc và kiểm tra mã nguồn                  | Cần bản sao kho mã và trình thông dịch Python. Báo cáo này dùng CPython 3.12.14 trên Windows để kiểm tra cú pháp.                                                                                                                                                                                                      |
| Khởi động CLI                             | Cần trình khởi chạy của dự án và các thư viện trong `requirements.txt`. Khi khởi động, mô-đun dùng chung nạp PyYAML, Colorama và python-dotenv.                                                                                                                                                                        |
| Tìm hoặc điều khiển thiết bị              | Để tìm thiết bị tự động, cần mạng LAN IPv4 hỗ trợ mDNS. Nếu đã biết địa chỉ Direct, có thể gửi yêu cầu HTTP đến địa chỉ đó. Với MQTT, thiết bị phải được cấu hình để dùng máy chủ trung gian (broker) mà CLI có thể kết nối. Cấu hình Wi-Fi qua Improv và theo dõi nhật ký nối tiếp đều cần cổng nối tiếp tương thích. |
| Ghi thông tin trạng thái kênh Wi-Fi (CSI) | Cần thiết bị có thể truy cập và có công bố khả năng truyền CSI thô qua HTTP phiên bản 1. Máy tính cần có công cụ thu dữ liệu và gửi được gói UDP theo thông số thiết bị công bố. Riêng `collect --info` đọc danh mục dữ liệu trên máy, kể cả khi chưa kết nối cảm biến.                                                |
| Biên dịch hoặc triển khai firmware        | Native, Matter và Micro dùng môi trường ESP-IDF cục bộ hoặc ảnh Docker đã cố định phiên bản. ESPHome dùng bộ công cụ Python riêng và tệp cấu hình YAML. Để triển khai ứng dụng Micro, cần `mpy-cross-v6.3`, `mpremote` và tệp cấu hình cục bộ.                                                                         |

## 3. Nội dung đã thực hiện

Các bảng dưới đây chỉ ra tệp mã nguồn phụ trách từng thao tác. Để đọc ví dụ lệnh, thêm `./espectre` ở đầu trên Unix hoặc `.\espectre.cmd` trên Windows. Các ví dụ minh họa cú pháp trong mã nguồn. Mục 4 ghi riêng những phép kiểm tra đã thực hiện.

### Điểm khởi chạy CLI và cấu hình dùng chung

| Mô-đun                                                | Chức năng chính                                                                                                                                                                                               | Lệnh hoặc nơi sử dụng                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| [__init__.py](../src/python/espectre_cli/__init__.py) | Khai báo và mô tả gói CLI chạy trên máy tính. Trình khởi chạy của dự án nạp hàm điều phối lệnh từ `app.py`.                                                                                                   | Khi Python nạp gói                                           |
| [app.py](../src/python/espectre_cli/app.py)           | Đăng ký 13 lệnh cấp cao nhất với `argparse`, gắn từng lệnh với hàm xử lý và chuyển kết quả của hàm thành mã thoát của tiến trình.                                                                             | `--help`                                                     |
| [about.py](../src/python/espectre_cli/about.py)       | Tạo nhãn phiên bản CLI từ mã Git rút gọn của bản mã nguồn hiện tại. Khi thiếu thông tin Git, mô-đun hiển thị `workspace checkout`.                                                                            | `version`, `about`                                           |
| [common.py](../src/python/espectre_cli/common.py)     | Đặt các đường dẫn dùng chung, nạp cấu hình môi trường, khai báo tham số kết nối MQTT và chọn cổng nối tiếp theo biến thể firmware, chip và thao tác. Nếu còn nhiều cổng phù hợp, CLI yêu cầu người dùng chọn. | `monitor`, `provision` và các lệnh làm việc với firmware     |
| [targets.py](../src/python/espectre_cli/targets.py)   | Ánh xạ tên rút gọn của chip sang tệp YAML của ESPHome hoặc sang ứng dụng và đích biên dịch của ESP-IDF. Native hỗ trợ cả ESP32-S2. Bảng đích của Matter gồm ESP32 đời đầu, C3, C5, C6 và S3.                  | `native build`, `matter build` và bước chọn cấu hình ESPHome |

### Truy cập thiết bị và thu thập dữ liệu

| Mô-đun                                                                | Chức năng chính                                                                                                                                                                                                                | Lệnh hoặc nơi sử dụng                                                    |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| [device_discovery.py](../src/python/espectre_cli/device_discovery.py) | Tìm dịch vụ `_espectre._tcp.local.`, kiểm tra địa chỉ IPv4 và thông tin DNS-SD, chuẩn hóa mã định danh thiết bị rồi trả về địa chỉ Direct mà thiết bị công bố.                                                                 | `devices --frontend native --json`                                       |
| [device_control.py](../src/python/espectre_cli/device_control.py)     | Lấy mật khẩu Wi-Fi từ biến môi trường hoặc yêu cầu người dùng nhập ẩn để cấu hình thiết bị qua Improv Serial. Với Direct, mô-đun xác định địa chỉ đích, kiểm tra dữ liệu JSON đầu vào rồi in phản hồi của thiết bị.            | `provision --ssid MyNetwork`, `direct get health --frontend native`      |
| [device_transport.py](../src/python/espectre_cli/device_transport.py) | Mã hóa và phân tích khung Improv, giới hạn thời gian chờ khi cấu hình thiết bị, kiểm tra khả năng hỗ trợ giao thức Direct, xác thực phản hồi HTTP và tiếp nhận luồng sự kiện từ máy chủ (SSE).                                 | `ImprovSerialClient` và `DirectClient`                                   |
| [mqtt_shell.py](../src/python/espectre_cli/mqtt_shell.py)             | Chọn thiết bị qua máy chủ MQTT, gợi ý lệnh dựa trên bản tin về khả năng của thiết bị mà máy chủ lưu giữ, gửi lệnh ở dạng JSON và ghép phản hồi với lệnh đang chờ.                                                              | `mqtt`                                                                   |
| [serial_monitor.py](../src/python/espectre_cli/serial_monitor.py)     | Hiển thị dữ liệu từ cổng nối tiếp dưới dạng văn bản UTF-8 hoặc xuất nguyên các byte. Theo mặc định, mở trình theo dõi sẽ giữ phiên hiện tại của thiết bị. Tùy chọn khởi động lại yêu cầu chip và giao diện nối tiếp có hỗ trợ. | `monitor --port COM3`                                                    |
| [host.py](../src/python/espectre_cli/host.py)                         | Chọn thiết bị hỗ trợ CSI thô, điều phối việc gửi gói UDP và nhận mẫu qua HTTP, kiểm tra bộ phát hiện chuyển động đã sẵn sàng chưa, lưu bản ghi có nhãn theo từng thiết bị rồi gọi bộ kiểm tra chất lượng.                      | `collect --info`, `collect --frontend native --label wave --duration 45` |

### Biên dịch và triển khai firmware

| Mô-đun                                                              | Chức năng chính                                                                                                                                                                                                                                                  | Lệnh hoặc nơi sử dụng                                                          |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| [idf.py](../src/python/espectre_cli/idf.py)                         | Chọn môi trường chạy ESP-IDF, biên dịch Native hoặc Matter cho chip đã chọn và đưa các tệp đã biên dịch thành công vào thư mục dành cho bước nạp. Với Matter, mô-đun đọc dữ liệu mã QR và mã ghép đôi thủ công từ đầu ra nối tiếp.                               | `doctor`, `native build --chip c3`, `matter qr --chip c6`                      |
| [idf_container.py](../src/python/espectre_cli/idf_container.py)     | Kiểm tra dịch vụ Docker và ảnh cần dùng, áp dụng quy tắc tải ảnh, rồi gắn thư mục mã nguồn và bộ nhớ đệm biên dịch vào vùng chứa. Mã băm SHA-256 xác định chính xác ảnh ESP-IDF 5.5.6 được dùng.                                                                 | `native build --chip c3 --backend docker`                                      |
| [build_artifacts.py](../src/python/espectre_cli/build_artifacts.py) | Đặt tên tệp nhị phân theo mã băm nội dung. Chỉ sau khi các tệp mới đã sẵn sàng, mô-đun mới thay `flasher_args.json`, đồng thời giữ các tệp cũ để phiên nạp đang chạy có thể hoàn tất. Mô-đun cũng xuất đường dẫn, kích thước và mã SHA-256 của tệp đã biên dịch. | Các lệnh biên dịch có `--json` và bước chuẩn bị tệp nạp cho Native hoặc Matter |
| [esptool_runner.py](../src/python/espectre_cli/esptool_runner.py)   | Đọc địa chỉ nạp và các tùy chọn từ tệp mô tả của ESP-IDF, sắp xếp tệp theo địa chỉ rồi chọn tốc độ truyền và chế độ khởi động lại phù hợp với chip.                                                                                                              | Nạp qua cổng nối tiếp hoặc thực hiện yêu cầu khởi động lại thiết bị            |
| [esphome.py](../src/python/espectre_cli/esphome.py)                 | Xác định tệp YAML để ESPHome kiểm tra cấu hình, biên dịch, đọc nhật ký hoặc nạp qua mạng. Khi nạp qua cổng nối tiếp, mô-đun dùng công cụ nạp chung với ảnh firmware xuất xưởng hoặc các tệp đã biên dịch.                                                        | `esphome config --chip c3`                                                     |
| [micro_firmware.py](../src/python/espectre_cli/micro_firmware.py)   | Tạo ảnh firmware nền từ các phiên bản MicroPython và micropython-lib đã được cố định. Mô-đun áp dụng bản vá cho CSI và bo mạch, chuẩn bị các tệp hỗ trợ và khóa thư mục biên dịch dùng chung để tránh xung đột giữa các phiên.                                   | `micro build`, `micro flash`                                                   |
| [micro.py](../src/python/espectre_cli/micro.py)                     | Điều phối biên dịch và nạp firmware nền. Khi triển khai, mô-đun biên dịch mã ứng dụng và cấu hình cục bộ thành tệp `.mpy` rồi chép lên bo mạch. Bước xác minh kiểm tra các phương thức CSI, bộ phát hiện chuyển động và bộ lấy mẫu tích hợp trong firmware.      | `micro deploy`, `micro run`, `micro verify`                                    |

### Từ lệnh CLI đến kết quả

**Đọc trạng thái thiết bị.** Khi chạy `direct get health --frontend native`, CLI chuyển yêu cầu đến hàm xử lý Direct và dùng cơ chế tìm thiết bị để lấy địa chỉ kết nối. Trước khi yêu cầu tài nguyên `health`, máy tính đọc `capabilities` để kiểm tra phiên bản giao thức, danh sách tài nguyên và danh sách thao tác. CLI in kết quả dưới dạng JSON. Giao diện dòng lệnh MQTT dùng cùng quy ước dữ liệu của ứng dụng và trao đổi qua các chủ đề trên máy chủ MQTT. [Đặc tả API](../docs/API.md)

**Ghi CSI trong 45 giây.** Với ví dụ `collect` có nhãn ở bảng trên, máy tính kiểm tra phiên bản giao thức CSI và dấu nhận diện gói tin mà thiết bị công bố. Sau đó, máy tính chuyển thiết bị sang chế độ nhận lưu lượng từ bên ngoài và đọc lại cấu hình để xác nhận. Máy tính gửi gói UDP để tạo lưu lượng Wi-Fi, còn cảm biến gửi mẫu CSI về qua HTTP. Phiên ghi chỉ bắt đầu khi bộ phát hiện đã sẵn sàng và chỉ số chuyển động duy trì dưới ngưỡng đủ lâu, mặc định là ba giây. Khoảng 45 giây được tính từ lúc bắt đầu ghi. Sau khi kết thúc phiên, thiết bị vẫn giữ chế độ nhận lưu lượng từ bên ngoài. [Quy trình thu thập CSI](../docs/CLI.md#collect)

## 4. Kết quả và đánh giá

### Đã đạt được

- Đã đọc và lập danh mục toàn bộ 18 tệp Python, gồm 9.024 dòng tính cả chú thích và dòng trống. CPython 3.12.14 đã phân tích từng tệp bằng `ast.parse` và biên dịch cây cú pháp ngay trong bộ nhớ. Cả 18 tệp đều vượt qua bước kiểm tra này.
- Đã xác định các mô-đun phụ trách đọc trạng thái thiết bị, ghi dữ liệu có nhãn và chuẩn bị tệp firmware để nạp. Riêng ở bước kiểm tra chất lượng bản ghi, nếu bộ kiểm tra xác định tệp không đạt, CLI giữ tệp trên đĩa và trả về mã lỗi. Nếu không nạp được bộ kiểm tra hoặc có ngoại lệ trong khi kiểm tra tệp, CLI in cảnh báo nhưng vẫn có thể trả về mã thành công. Vì vậy, cần đọc thông báo kiểm tra chất lượng để biết tệp đã thực sự được kiểm tra hay chưa. [Mã kiểm tra chất lượng bản ghi](../src/python/espectre_cli/host.py)

### Chưa đạt / hạn chế

- Trên bản sao dự án dùng cho báo cáo này, lệnh `.\espectre.cmd --help` trên Windows trả về mã thoát 1 vì trình khởi chạy không tìm thấy trình thông dịch Python dùng được. Khi gọi trực tiếp tập lệnh của dự án bằng CPython 3.12.14 có sẵn, chương trình dừng ở bước nạp mô-đun `yaml` do thiếu thư viện PyYAML. Cú pháp lệnh trong báo cáo đã được đối chiếu với mã phân tích đối số và tài liệu CLI.
- Phạm vi xác minh gồm cấu trúc mã nguồn và cú pháp Python. Đợt rà soát này chưa chạy trên bo mạch thật, kết nối máy chủ MQTT, thu luồng CSI trực tiếp hoặc biên dịch firmware. Chưa có số liệu thực nghiệm về khả năng tương thích khi chạy, tốc độ thu gói thực tế và hiệu năng phát hiện chuyển động.

## 5. Việc làm tiếp theo / Action items

Chưa có công việc tiếp theo được chỉ định.

## 6. Các link, nguồn tài liệu tham khảo

- Mục 3 dẫn đến từng tệp trong số 18 tệp mã nguồn đã rà soát. [Thư mục mã nguồn CLI](../src/python/espectre_cli/)
- Cách chọn trình thông dịch, chuyển lệnh đến CLI và xác định các thư viện cần trên máy tính. [Trình khởi chạy Python](../espectre) · [Trình khởi chạy Windows](../espectre.cmd) · [Danh sách thư viện](../requirements.txt)
- Tổng quan dự án, cú pháp lệnh, các thao tác Direct được hỗ trợ và thông tin thiết bị công bố qua DNS-SD. [Giới thiệu ESPectre](../README.md) · [Tài liệu CLI](../docs/CLI.md) · [Đặc tả API](../docs/API.md) · [Đặc tả tìm thiết bị](../docs/DISCOVERY.md)
- Cấu trúc và quy tắc đặt tên báo cáo. [Hướng dẫn viết báo cáo kỹ thuật](../skills/technical-report/SKILL.md) · [Mẫu báo cáo](../TemplateReport/M%E1%BA%ABu%20b%C3%A1o%20c%C3%A1o.md) · [Quy ước chỉnh sửa mẫu](../TemplateReport/change.md)
