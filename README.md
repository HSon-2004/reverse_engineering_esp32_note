# Study reverse engineering ESP32

Đây là một repo mình dùng để note lại quá trình tìm hiểu về reverse engineering ESP32. Mình sẽ cố gắng cập nhật thường xuyên và chia sẻ những gì mình đã học được.

## Tools used
- IDA Pro
- Ghidra
- esptool
- esp32_image_parse
- qemu
- ImHex
- mkspiffs

## Mở đầu
Đây có vẻ là bài viết đầu tiên của mình về reverse engineering ESP32 và cũng là bài viết đầu tiên của mình về reverse engineering nói chung 😂, vậy là sau hơn một năm học reverse engineering thì mới có một bài viết đầu tiên về lĩnh vực này (lý do là mình quá lười để viết 😠). Let's go ta hãy đi vào chủ đề chính nào.

ESP32 là một dòng vi điều khiển rất phổ biến trong cộng đồng IoT và được sử dụng rộng rãi trong các sản phẩm thương mại. Thông thường các công cụ như IDA Pro hay Ghidra vẫn chưa hỗ trợ tốt cho việc phân tích file binary của ESP32 và firmware cũng khá hiếm gặp trong các challenge CTF. Tuy nhiên, dạo gần đây thì mình thấy có một số challenge CTF về ESP32 trong các giải mà mình đã chơi như Codegate2025 hay ASISfinal2024. Do không giải được bài trong thời gian của cuộc thi CTF 😢, nên mình đã thử up solve lại. Nhờ vậy mình đã tìm hiểu được một chút về ESP32 và cách phân tích firmware của nó. 

## File binary của ESP32
ESP32 sử dụng định dạng file binary là ELF (Executable and Linkable Format), tuy nhiên file thực sự được nạp vào bộ nhớ là một file nhị phân có định dạng là BIN. File nhị phân này được tạo ra từ file ELF bằng cách sử dụng công cụ esptool. Công cụ này sẽ chuyển đổi file ELF thành file BIN và nén lại để tiết kiệm dung lượng bộ nhớ. File BIN này sẽ được nạp vào bộ nhớ của ESP32 khi khởi động.

Thông thường, nếu thử dùng lệnh file để kiểm tra file BIN thì sẽ không nhận được thông tin gì cả. Nếu thử mở file bằng IDA Pro thì có file IDA sẽ detect được file này là một file của ESP32 nhưng có file thì không thể detect được 😧 (ở đây mình lấy 2 challenge của codegate2025 và asisfinal2024). Tương tự nếu dùng esptool.py thì file `Torando.bin` sẽ detect được là esp32-c3, còn file `flash.bin` thì báo lỗi. Mở file `flash.bin` trong ImHex thì có thể thấy được giữa các partitions sẽ là các byte `0xFF`, còn file `Torando.bin` thì các byte `0xff` sẽ được thêm vào cuối file. Sở dĩ như vậy vì flash của esp32 sẽ là 4MB nên khi nạp file vào mạch thì phải chèn `0xff` vào để đủ 4MB, cách chèn `0xff` vào giữa các partitions hay vào cuối file phụ thuộc vào tool nạp hoặc config của người dùng.
Do vậy để có thể phân tích được file `flash.bin` thì có thể dùng công cụ `esp32_image_parse` để phân tích file này. 
```bash
python esp32_image_parser.py show_partitions ../flash.bin 

reading partition table...
entry 0:
  label      : nvs
  offset     : 0x9000
  length     : 20480
  type       : 1 [DATA]
  sub type   : 2 [WIFI]

entry 1:
  label      : otadata
  offset     : 0xe000
  length     : 8192
  type       : 1 [DATA]
  sub type   : 0 [OTA]

entry 2:
  label      : app0
  offset     : 0x10000
  length     : 1310720
  type       : 0 [APP]
  sub type   : 16 [ota_0]

...

```
## Dump and Parse firmware 

Thông thường code nằm trong partition `app0` và `app1`, còn các partition khác sẽ là các dữ liệu như OTA hay NVS. Để dump được code trong partition `app0` thì có thể dùng lệnh sau:
```bash
python esp32_image_parser.py dump_partition flash.bin -partition app0 -output app0_out.bin

```

Sau khi dump thì để có thể đọc được file bằng Ghidra(IDA hiện tai vẫn chưa decompile được) thì cần parse lại file này về định dạng ELF. Để parse lại file thì có thể dùng công cụ `esp32_image_parse` với lệnh sau:
```bash
python3 esp32_image_parser.py create_elf flash.bin -partition app0 -output app0_out.elf
``` 
Thường khi chạy lệnh trên thì sẽ xuất hiện lỗi do mã nguồn đã cũ và không được update. Để fix lỗi này thì có thể sửa lại một chút trong file `esp32_image_parser.py` như sau thêm `BYTE_ACCESSIBLE, DRAM' : '.dram0.data',` tại dòng 58 và thêm lib `from esptool.bin_image import *` (có thể check thêm các cách sửa lỗi khác trong pr trên github của esp32_image_parser). Sau khi sửa xong thì có thể chạy lại lệnh trên và sẽ có file `app0_out.elf` được tạo ra. Thực ra thì với file `app0_out.bin` cũng có thể mở được bằng Ghidra hoặc IDA nhưng sẽ không có các thông tin như symbol hay function name. Do vậy để có thể phân tích được dễ hơn thì nên parse lại file này về định dạng ELF. Tương tự với file `Torando.bin` thì cũng cần sửa lại source code của `esp32_image_parser.py` để có thể parse được file này (`Torando.bin` là file build cho esp32c3 xài risc-v).

## Reverse engineering

Sau khi đã có file ELF thì có thể mở file này bằng Ghidra hoặc IDA Pro để phân tích. Sẽ có một vài hàm có sẵn sysmbol, còn các hàm khác sẽ không có symbol. Trong trường hợp này thì mình có thể xài IDA FLIRT Signature để tìm kiếm các hàm có sẵn trong thư viện ESP32.

Trong esp32 thì hàm app_main sẽ giống như hàm main trong các chương trình C khác. Hàm này thường sẽ chứa các hàm khác như setup hay loop. Trong hàm setup sẽ thường chứa các hàm khởi tạo như khởi tạo wifi, khởi tạo các task hay khởi tạo các biến toàn cục. Còn trong hàm loop sẽ thường chứa các hàm xử lý chính của chương trình. Do vậy nếu tìm được hàm app_main thì có thể tìm được các hàm khác trong chương trình. Nếu chương trình có sử dụng FreeRTOS thì có thể tìm được các task trong hàm setup. Ngoài ra các chương trình ngắt cũng thường được định nghĩa trong hàm setup. 



To be continued...