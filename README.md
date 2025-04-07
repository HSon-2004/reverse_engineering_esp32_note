# Study reverse engineering ESP32

Đây là một repo mình dùng để note lại quá trình tìm hiểu về reverse engineering ESP32. Mình sẽ cố gắng cập nhật thường xuyên và chia sẻ những gì mình đã học được.

## Tools used
- IDA Pro
- Ghidra
- esptool
- esp32_image_parse
- qemu
- ImHex

## Mở đầu
Đây có vẻ là bài viết đầu tiên của mình về reverse engineering ESP32 và cũng là bài viết đầu tiên của mình về reverse engineering nói chung 😂, vậy là sau hơn một năm học reverse engineering thì mới có một bài đầu tiên về lĩnh vực này. Let's go ta hãy đi vào chủ đề chính nào.

ESP32 là một dòng vi điều khiển rất phổ biến trong cộng đồng IoT và được sử dụng rộng rãi trong các sản phẩm thương mại. Thông thường các công cụ như IDA Pro hay Ghidra vẫn chưa hô trợ tốt cho việc phân tích file binary của ESP32 và firmware cũng khá hiếm gặp trong cái challenge CTF. Tuy nhiên, dạo gần đây thì mình thấy có một số challenge CTF về ESP32 trong các giải mà mình đã chơi như Codegate2025 hay ASISfinal2024. Do không giải được bài trong thời gian của cuộc thi CTF 😢, nên mình đã thử up solve lại. Nhờ vậy mình đã tìm hiểu được một chút về ESP32 và cách phân tích firmware của nó. 

## File binary của ESP32
ESP32 sử dụng định dạng file binary là ELF (Executable and Linkable Format), tuy nhiên file thực sự được nạp vào bộ nhớ là một file nhị phân có định dạng là BIN. File nhị phân này được tạo ra từ file ELF bằng cách sử dụng công cụ esptool. Công cụ này sẽ chuyển đổi file ELF thành file BIN và nén lại để tiết kiệm dung lượng bộ nhớ. File BIN này sẽ được nạp vào bộ nhớ của ESP32 khi khởi động.

Thông thường, nếu thử dùng lệnh file để kiểm tra file BIN thì sẽ không nhận được thông tin gì cả. Nếu thử mở file bằng IDA Pro thì có file IDA sẽ detect được file này là một file của ESP32 nhưng có file thì không thể phân tích được 😧 (ở đây mình lấy 2 challenge của codegate2025 và asisfinal2024). 

To be continued...