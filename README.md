# Charybdis Nano Dual Ball handwire

Cấu hình ZMK cho hai nửa BLE, mỗi nửa có 17 phím và một trackball. Nửa phải làm central, nhận ma trận cục bộ và PMW3360, rồi chuyển báo cáo bàn phím và chuột cho máy tính qua BLE hoặc USB. Nửa trái gửi phím và sự kiện PMW3610 qua BLE. Hai nửa có nguồn riêng. Cấu hình dongle ba bo trước khi chuyển sang hai bo có trong commit `6e04cd5`.

Hai target dùng `nice_nano//zmk`, mặc định revision 2.0.0 theo ZMK. Kiểm tra clone có pinout tương ứng và bootloader nhận UF2 được tạo cho nice!nano v2. Việc build chỉ xác nhận target phần mềm. Dùng nhãn và đo trên bo thật để xác nhận từng chân trước khi hàn.

## Ma trận phím

Dùng một ma trận 3×6 cho mỗi nửa. Chân MCU bên dưới dùng chung cho cả hai sơ đồ đấu dây, mỗi chân đi đến hàng hoặc cột của chính nửa đó.

| Dây ma trận | Nhãn bo | GPIO nRF52840 |
| --- | --- | --- |
| R0 | 006 | P0.06 |
| R1 | 008 | P0.08 |
| R2 | 017 | P0.17 |
| C0 | 011 | P0.11 |
| C1 | 104 | P1.04 |
| C2 | 031 | P0.31 |
| C3 | 029 | P0.29 |
| C4 | 002 | P0.02 |
| C5 | 115 | P1.15 |

Nối mỗi phím theo thứ tự **Cx → switch → anode diode → cathode có vạch → Ry**. Firmware quét `col2row`, với pull-down ở hàng. Nhìn từ phía keycap, mỗi dòng dưới đây được đọc từ trái sang phải. Hai layer Mac và Windows cùng dùng QWERTY. Các dấu minh họa theo bộ gõ US.

| Vị trí | Trái | Phải |
| --- | --- | --- |
| Hàng trên R0 | C0 C1 C2 C3 C4 · Q W E R T | C4 C3 C2 C1 C0 · Y U I O P |
| Hàng giữa R1 | C0 C1 C2 C3 C4 · A S D F G | C4 C3 C2 C1 C0 · H J K L ; |
| Hàng dưới R2 | C0 C1 C2 C3 C4 · Z X C V B | C4 C3 C2 C1 C0 · N M , . / |
| Hai thumb | R1/C5 R2/C5 · Space Tab | R1/C5 R2/C5 · Enter Backspace |

R0/C5 bỏ trống ở mỗi nửa. Nếu đã hàn, dùng đồng hồ xác nhận cột thực tế và hai tọa độ thumb trước khi đổi dây. Hai nửa dùng cùng thứ tự 34 vị trí của keymap. Transform cộng offset 6 cho cột nửa phải.

Vị trí B ở layer số giữ hành vi dấu nháy của Mac và Windows trước đây.

## Đấu tín hiệu cảm biến

Đọc chữ in trên header và xác nhận chiều nhìn của từng module. Bảng này ánh xạ tên tín hiệu đã xác nhận đến MCU. Thứ tự dòng trong bảng độc lập với thứ tự chân header.

| Nhãn bo MCU | GPIO | Trái · PMW3610M | Phải · module PAW3360/3389 |
| --- | --- | --- | --- |
| 020 | P0.20 | NCS | NCS |
| 022 | P0.22 | SCLK | SCLK |
| 024 | P0.24 | SDIO | MOSI |
| 100 | P1.00 | Để trống | MISO |
| 113 | P1.13 | MOT / MOTION | MOT / MOTION |
| 111 | P1.11 | Dự phòng, để trống | Dự phòng, để trống |
| GND | GND | GND cùng nửa | GND cùng nửa |

Để 009 và 010 trống. Hai driver dùng SPI mode 3 và overlay giới hạn SPI0 ở 2 MHz. PMW3610 dùng SDIO hai chiều, nên pinctrl gán cả MOSI và MISO vào P0.24 và chỉ có một dây SDIO. Bên phải dùng MOSI và MISO riêng. Cả hai bản build dùng ngắt MOT tác động mức thấp. I²C, UART phần cứng và SPI1 trên bo được tắt vì có chân trùng với cấu hình này.

Driver PMW3610 đọc qua `spi_transceive_dt()` với ánh xạ chung chân theo hướng dẫn của badjeff. Driver không tự điều khiển GPIO để đổi hướng SDIO. Khi thử, kiểm tra Product ID và mức SDIO trong pha đọc. Nếu dùng mạch chuyển mức, xác nhận phía MCU nhả đường truyền đúng lúc để hai đầu tránh cùng lái SDIO.

Firmware để NRES/NRESET rời khỏi MCU. Xác minh mạch reset của đúng module giữ mức idle hợp lệ. Nếu chưa có mức idle, đối chiếu sơ đồ và datasheet để chọn mạch giữ mức trước khi chạy thử. Đọc mã trên chip sau khi đối chiếu revision PCB, hoặc đo Product ID ở thanh ghi `0x00` bằng thiết bị đọc SPI hay firmware chẩn đoán có log. PMW3360DM-T2QU dự kiến trả `0x42`. PMW3610DM-SUDU dự kiến trả `0x3E`. Với module phải, nếu mã chip hoặc ID khác, dừng và chọn driver theo chip thực tế. Chữ PAW3360/3389 trên PCB chỉ xác định loại PCB.

Driver PMW3360 ở revision đã ghim thực hiện reset qua thanh ghi và đọc Product ID để ghi log debug. Driver bỏ qua bước nạp SROM và tiếp tục khởi tạo dù ID khác giá trị dự kiến. Datasheet PMW3360 đưa nạp SROM vào trình tự khởi tạo chuẩn. Vì vậy cần xác minh đúng chip trước khi chạy firmware bên phải với module đã cắm. Khả năng chạy ổn định với driver này cần được xác nhận trên bàn. Nếu module cần SROM để hoạt động đúng, bản firmware này cần driver có hỗ trợ nạp SROM và firmware cảm biến phù hợp.

### Nguồn

Xác định bằng sơ đồ nhà bán hoặc truy vết khi đã tháo USB và pin. Kiểm tra GND, VDD, VDDIO/VDIO, reset, vị trí chân số 1 và linh kiện trên module. Xác định từng chân nguồn ở header là đầu vào regulator, rail đi thẳng vào IC hay rail đã được cấp trên module. Phép đo thông mạch giúp truy đường mạch. Dùng mã linh kiện và sơ đồ để xác định điện áp đầu vào.

| Chip và điều kiện áp dụng | VDD | VDDIO |
| --- | --- | --- |
| PMW3610DM-SUDU, nếu header đi thẳng vào rail IC | 1,70–2,10 V, điển hình 1,80 V | 1,70–3,30 V |
| PMW3360DM-T2QU, nếu header đi thẳng vào rail IC | 1,80–2,10 V, điển hình 1,90 V | 1,80–3,60 V và ≥ VDD |

Nếu cấp trực tiếp vào IC PMW3360, dùng regulator khoảng 1,9 V cho VDD và đối chiếu điện áp dưới tải, dòng và tụ theo datasheet. VDD và VDDIO phải lên trong vòng 100 ms của nhau. Chọn logic 3,3 V khi phù hợp với bo và module. Với PMW3610, 3,30 V là giới hạn trên của dải làm việc VDDIO. Tính dung sai và đo ở cả chế độ USB lẫn pin trước khi nối trực tiếp. Nếu module dùng logic 1,8 V, dùng giải pháp chuyển mức đáp ứng tốc độ SPI, hướng các chân và SDIO hai chiều của PMW3610. Đảm bảo MOT và điện trở kéo lên thuộc miền điện áp phù hợp.

Hai nửa tách điện. Mỗi module chung GND với MCU cùng nửa. LiPo 1S 3,7 V danh định chỉ nối B+/B− đã xác minh cực tính và mạch sạc của clone. Cấm cấp trực tiếp RAW, VBUS, 5 V hoặc B+ vào VDD/VDDIO cảm biến. Dùng nguồn cảm biến ổn định suốt thời gian MCU hoạt động. Nếu nguồn cảm biến đi qua VCC điều khiển được, xác minh thêm GPIO không cấp ngược qua chân tín hiệu khi VCC tắt, và cảm biến được khởi tạo lại khi VCC bật.

## Kiểm tra trên bàn

1. Rút USB và tháo pin trước khi hàn hoặc cắm/rút module. Kiểm tra chập VDD–GND, VDDIO–GND và giữa hai rail, cầu thiếc, cực tính và chiều diode bằng thang đo phù hợp.
2. Thử hai bo và ma trận với USB khi cảm biến còn ngắt. Dùng USB nửa phải để nhận báo cáo. Cho hai nửa kết nối BLE. Thử riêng 17 phím mỗi nửa, gồm hai thumb mỗi bên.
3. Đo các rail ở đầu nối trống khi dùng USB và khi dùng pin. Cắt điện trước khi cắm từng module. Thử từng nửa với nguồn giới hạn dòng khi có và xác minh mã chip hoặc Product ID. Ngắt cấp nguồn ngay nếu nóng, dòng tăng bất thường hoặc rail sụt. Với module phải, xác minh PMW3360 trước khi chạy bản firmware này. Theo dõi việc khởi tạo và báo cáo chuyển động.
4. Lăn mỗi bi riêng, rồi cả hai cùng lúc. Kiểm tra X/Y, chiều cuộn khi dùng layer scroll và việc BLE tự kết nối lại sau khi tắt/bật nguồn. Hai bộ chỉnh trục nằm trong `split_input_common.dtsi`. `trackball_listener` là PMW3360 bên phải, `trackball_listener2` là PMW3610 bên trái. Chỉnh riêng `input-processors` theo hướng lắp thực tế.
5. ZMK sleep được tắt trong hai bản build. Nếu bật sleep hoặc thêm cơ chế ngắt VCC, kiểm tra thức dậy, cấp lại rail và khởi tạo lại cảm biến, gồm SROM của PMW3360.

## Build

GitHub Actions dùng `build.yaml` với hai target Charybdis. `config/west.yml` ghim ZMK và hai driver theo commit. ZMK nhập Zephyr `v4.1.0+zmk-fixes`. Khi đã cài west, các công cụ Zephyr và Zephyr SDK, chạy các lệnh dưới đây từ thư mục repo. Workspace build dùng thư mục tạm và đọc cấu hình hiện tại, kể cả thay đổi chưa commit.

```sh
repo="$(pwd -P)"
zmk_workspace="$(mktemp -d)"
mkdir "$zmk_workspace/config"
cp "$repo/config/west.yml" "$zmk_workspace/config/west.yml"
west init -l "$zmk_workspace/config"
cd "$zmk_workspace"
west update

for side in left right; do
  west build -b nice_nano//zmk -s zmk/app -d "$zmk_workspace/build-$side" -- \
    -DSHIELD="charybdis_$side" \
    -DZMK_CONFIG="$repo/config" \
    -DZMK_EXTRA_MODULES="$repo"
done
```

Mỗi build tạo `build-left/zephyr/zmk.uf2` hoặc `build-right/zephyr/zmk.uf2` trong `$zmk_workspace`. Xem `zephyr/zephyr.dts` và `zephyr/.config` trong mỗi thư mục để kiểm tra GPIO, SPI0, offset ma trận, vai trò BLE, driver được bật và đường input split. Cả hai DTS có input split `reg = <1>`; bên trái tham chiếu PMW3610, bên phải tham chiếu node split qua listener. Listener PMW3360 bên phải tham chiếu trực tiếp cảm biến cục bộ.

## Nguồn đối chiếu

- [ZMK Pointing Devices](https://zmk.dev/docs/hardware-integration/pointing)
- [Pinout và sơ đồ nice!nano](https://nicekeyboards.com/docs/nice-nano/pinout-schematic/)
- [Driver badjeff/PMW3610](https://github.com/badjeff/zmk-pmw3610-driver)
- [Mã khởi tạo driver PMW3360 đã ghim](https://github.com/george-norton/zmk-driver-pmw3360/blob/0dfa2f0411b795613a35f222f993e45a1e4cc32a/drivers/input/input_pmw3360.c)
- [Datasheet PMW3610DM-SUDU](https://trackballs.eu/media/Nakabayashi/Digio2/PMW3610DM-SUDU.pdf)
- [Datasheet PMW3360DM-T2QU](https://datasheet.lcsc.com/datasheet/pdf/c018c3e1d470821feefadd37e2f785d5.pdf?productCode=C20612443)
