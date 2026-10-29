# Đặc tả đề xuất OTFS trên FPGA — v0.1

Ngày: 28/10/2026. Trạng thái: đề xuất thiết kế, chưa có kết quả tổng hợp RTL hoặc đo trên board.

## 1. Quyết định đề xuất

- Tận dụng DE2-115 để triển khai bộ điều chế/giải điều chế OTFS băng gốc và đánh giá số học fixed-point.
- Dùng USRP làm đầu thu/phát RF qua máy tính ở giai đoạn đầu. Model USRP hiện chưa xác định.
- Chốt cấu hình đầu tiên: SISO, QPSK, lưới delay–Doppler 128 × 32, tốc độ mẫu phức 2 MS/s, CP 16 mẫu cho mỗi subsymbol.
- Hoàn thành TX, RX transforms và loopback trước; đưa detector có xét Doppler lên FPGA là mốc riêng, phụ thuộc kết quả tổng hợp và ngân sách chu kỳ.
- Nếu đã có X310, cân nhắc triển khai trực tiếp trong FPGA của X310 bằng RFNoC, bỏ đường truyền trung gian DE2-115 khi cần chạy độc lập.

Đây là nguyên mẫu nghiên cứu OTFS hướng tới ứng dụng 6G/high mobility. Không gọi là thiết bị tuân thủ chuẩn 6G. ITU đã công bố khung IMT-2030; khung đó không xác lập OTFS là dạng sóng bắt buộc [S1].

## 2. Phạm vi chức năng

### Bắt buộc

1. Mô hình tham chiếu floating-point và fixed-point dùng cùng quy ước trục, chuẩn hóa FFT, CP, thứ tự mẫu.
2. TX FPGA: bit source → QPSK → pilot/data mapping → OTFS modulation → CP → I/Q output.
3. RX FPGA: CP removal và OTFS demodulation; hard decision trong AWGN/kênh đơn giản.
4. Bộ thu tham chiếu trên PC có đồng bộ, ước lượng kênh và detector để đánh giá kênh đa đường biến thiên theo thời gian.
5. So sánh BER, EVM, tài nguyên và throughput; OFDM dùng cùng băng thông, năng lượng bit và cách tính overhead.
6. Thử nghiệm RF bằng USRP sau khi hoàn thành loopback số.

### Mở rộng có điều kiện

- Detector thưa có xét Doppler trên FPGA; 16-QAM; lưới 256 × 64; FEC.
- Truyền liên tục với PC bridge sau khi có Ethernet MAC/UDP, FIFO, bộ đếm mất mẫu và đo tải thực tế.
- MIMO, LDPC, mmWave và băng thông hàng trăm MHz không thuộc cấu hình ban đầu.

## 3. Thông số PHY ban đầu

| Hạng mục | Giá trị / quy định |
|---|---|
| Biến thể | OTFS dựa trên CP-OFDM, xung chữ nhật, CP ở mỗi subsymbol |
| Antenna | SISO, 1 TX + 1 RX |
| Lưới DD | M = 128 delay bins, N = 32 Doppler bins |
| Điều chế | QPSK Gray; 16-QAM ở bước mở rộng |
| Sample rate | Fs = 2,000,000 mẫu phức/s |
| Khoảng cách sóng mang | Δf = Fs/M = 15,625 Hz |
| Thời gian hữu ích/subsymbol | M/Fs = 64 µs |
| CP | Lcp = 16 mẫu = 8 µs, ở từng subsymbol |
| Thời gian subsymbol có CP | Ts = 72 µs |
| Mẫu/frame | N(M+Lcp) = 4,608, chưa gồm preamble và khoảng nghỉ |
| Thời gian frame | 2.304 ms, chưa gồm preamble và khoảng nghỉ |
| Độ phân giải delay | 1/(MΔf) = 0.5 µs |
| Độ phân giải Doppler | 1/(N Ts) ≈ 434.03 Hz |
| Bit/frame tối đa | 128 × 32 × 2 = 8,192 bit trước pilot/guard/FEC |
| Tốc độ bit tối đa với CP | ≈ 3.556 Mbit/s, trước pilot/guard/FEC/preamble |
| I/Q giao tiếp | signed int16 I + signed int16 Q, 32 bit/mẫu |
| Băng thông lưới danh định | MΔf = 2 MHz; không đồng nhất với băng thông chiếm dụng RF đo được |
| Tần số RF | Chọn sau khi xác định USRP và daughterboard; không ảnh hưởng chức năng băng gốc |

Chọn 2 MS/s để dễ bắt đầu với nhiều hệ USRP; vẫn phải đọc lại sample rate thực mà UHD chấp nhận. Đây là numerology nghiên cứu, không phải cấu hình NR 15 kHz. Nếu bắt buộc Δf = 15 kHz thì Fs = 1.92 MS/s và phải xem xét sample-rate conversion, nhất là trên N210 có clock cố định 100 MHz [S5].

Mô hình đầu tiên dùng đầy đủ lưới để kiểm chứng thuật toán. Pilot, guard DD và mask/DC/guard trong miền TF phải có định nghĩa riêng. Không thêm mask TF rồi mặc nhiên giữ nguyên đường rút gọn toán học chưa kiểm chứng. Budget ban đầu dành tối đa khoảng 25% vị trí DD cho pilot/guard, nhưng tỷ lệ cuối cùng phải tính theo delay/Doppler support và detector. Nếu tỷ lệ dữ liệu là 75%, tốc độ trước FEC/preamble là khoảng 2.667 Mbit/s; với code rate 1/2 còn khoảng 1.333 Mbit/s.

## 4. Kiến trúc và thuật toán

### 4.1 TX/RX tham chiếu

TX:

`bits → QPSK → X_DD[M,N] → ISFFT → X_TF → IFFT_M mỗi cột → CP → preamble/burst → I/Q`

RX:

`I/Q → phát hiện frame/timing/CFO → bỏ CP → FFT_M → SFFT → channel estimation/detection → bits`

Quy ước X_DD[m,n]: hàng m là delay, cột n là Doppler. Dùng biến đổi unitary trong mô hình toán; RTL dùng scaling stage rõ ràng và bù scale ở đầu ra. Với xung chữ nhật, không có xử lý TF xen giữa, FFT_M trong ISFFT và IFFT_M của OFDM có thể triệt tiêu; TX có thể chỉ cần M phép IFFT_N cộng bộ nhớ đổi thứ tự. RX tương ứng dùng FFT_N. Phải so sánh hai đường với cùng chuẩn hóa và ordering trước khi chọn bản rút gọn [S2].

CP mỗi subsymbol được chọn nhất quán trong mô hình kênh và receiver. Không dùng nguyên công thức equalizer dành cho reduced-CP hoặc zero-padded OTFS nếu chưa chuyển đổi mô hình.

### 4.2 Detector

- Mốc A: AWGN, perfect timing và perfect CSI; kiểm tra transform và QPSK.
- Mốc B: kênh đa đường có Doppler; detector LMMSE tham chiếu chạy trên PC, có thể dùng mô hình thưa và solver lặp.
- Mốc C: ước lượng kênh từ training/pilot, bổ sung fractional Doppler.
- Mốc D: đưa solver thưa hoặc detector lặp phù hợp biến thể CP lên FPGA; khởi đầu giới hạn 4 đường truyền, 4 vòng lặp rồi đánh giá tăng lên.

Không coi FFT/SFFT + QPSK slicer là một receiver OTFS hoàn chỉnh cho kênh high-Doppler. One-tap equalizer chỉ là baseline cho điều kiện phù hợp. Fractional Doppler làm năng lượng trải ra nhiều bin, nên số tap tính toán có thể lớn hơn số đường truyền vật lý.

Không lưu ma trận kênh đặc 4096 × 4096 trong BRAM: riêng ma trận complex float32 đã khoảng 128 MiB. Ưu tiên danh sách tap và phép nhân ma trận–vector ngầm.

## 5. Kiến trúc phần cứng DE2-115

DE2-115 dùng Cyclone IV E EP4CE115, 114,480 LE, khoảng 3.9 Mbit embedded RAM và 266 bộ nhân 18 × 18; có GbE và HSMC [S3, S4]. Số bộ nhân này không đồng nghĩa với 266 phép nhân phức mỗi chu kỳ.

| Khối | Đề xuất |
|---|---|
| Clock xử lý | Mục tiêu 100 MHz; chỉ xác nhận sau place-and-route và timing analysis |
| FFT | FFT/IFFT 32 điểm dùng lại theo hàng cho bản rút gọn; bản tham chiếu có thêm biến đổi 128 điểm |
| Số học | I/Q vào/ra 16 bit; nội bộ khởi đầu 18 bit, tích lũy rộng hơn; width/scaling chốt bằng sweep fixed-point |
| Overflow | Rounding, saturation và bộ đếm clipping; không cho wraparound âm thầm |
| Buffer | Ping-pong RAM, địa chỉ chuyển thứ tự hàng/cột, FIFO qua clock domain |
| Giao diện nội bộ | valid/ready, start/end-of-frame, frame_id, error flags |
| Debug | SignalTap, bộ đếm cycle/frame, FIFO watermark, overflow/underflow, CRC |
| Host link giai đoạn đầu | JTAG/UART dùng điều khiển và chuyển frame offline |
| Host link liên tục | Gigabit Ethernet với MAC + packetization/UDP; HDL/IP/license phải được kiểm tra |

Một lưới 4096 mẫu phức 16+16 bit cần 16 KiB; hai buffer cần 32 KiB. Một buffer waveform có CP cần 18 KiB. Nếu lưu nội bộ 18+18 bit thì lưới cần 18 KiB. Đây chỉ là payload storage, chưa gồm packing M9K, FFT buffers, FIFO, hệ số và trạng thái detector.

Ngân sách thiết kế sơ bộ: giữ tổng LE, RAM và multiplier dưới khoảng 70% để còn chỗ routing và tích hợp. Đây là trần mục tiêu, không phải tài nguyên đã đo. DE2-115 có cơ sở để làm modem transforms nhỏ; full receiver thời gian thực cần kết quả tổng hợp thực tế.

Ở 100 MHz có 230,400 chu kỳ/frame. Initiation interval của pipeline phải ≤ 2.304 ms để theo kịp input liên tục; latency toàn frame có thể lớn hơn nếu đủ buffering. Mục tiêu riêng cho xử lý sau khi đã nhận đủ frame: ≤ 2.304 ms. Nếu TX/RX dùng chung engine thì tổng lịch xử lý phải đáp ứng cả hai chiều.

## 6. Kết nối với USRP

### Phương án 1 — khuyến nghị để bắt đầu

`DE2-115 ⇄ PC (test harness + UHD) ⇄ USRP ⇄ kênh RF`

1. FPGA sinh waveform; chuyển file/frame về PC.
2. PC phát waveform bằng USRP; thu I/Q bằng USRP.
3. PC đưa frame nhận về DE2-115 để kiểm thử RX; detector phức tạp có thể chạy PC trong mốc đầu.
4. Báo cáo rõ đây là kiểm thử phần cứng theo frame, chưa phải modem RF độc lập thời gian thực.

Sau đó mới thay chuyển frame offline bằng Ethernet streaming. 2 MS/s × 32 bit = 64 Mbit/s = 8 MB/s mỗi chiều, chưa gồm packet headers. Khi cùng truyền/thu, PC phải xử lý cả hai luồng và bộ đệm; bandwidth danh định của GbE không tự bảo đảm latency hoặc không mất mẫu.

Không xem cổng USB-Blaster là đường USB sample streaming. Cắm Ethernet từ DE2-115 vào USRP cũng không tự tạo liên kết I/Q; phải có logic giao thức và điều khiển UHD tương ứng. DE2-115 có HSMC, trong khi FMCOMMS3 dùng FMC; chúng không cắm trực tiếp được.

### Phương án 2 — OTFS chạy trong FPGA của USRP

| Model phát hiện được | Hướng đề xuất |
|---|---|
| B200/B210 | Dùng làm RF front end qua USB 3.0. B210 có Spartan-6 XC6SLX150, dùng ISE; RFNoC không được hỗ trợ như các dòng mới. Sửa FPGA được nhưng không ưu tiên cho mốc đầu [S6]. |
| N200/N210 | Dùng RF front end qua GbE; cần biết daughterboard. N210 thuộc thế hệ FPGA cũ, không phải lựa chọn ưu tiên cho detector OTFS mới [S5, S7]. |
| X300/X310 | Phù hợp triển khai block OTFS trực tiếp qua RFNoC. X310 dùng Kintex-7 XC7K410T, nhưng phải trừ tài nguyên của radio/transport shell [S6, S7]. |
| E320/N310 | Cũng có thể dùng RFNoC; lựa chọn phụ thuộc thiết bị có sẵn và mức độ cần chạy embedded [S6, S8]. |

Một USRP đủ thử loopback TX–RX nếu thiết bị/daughterboard hỗ trợ đồng thời. Thử RF có dây dùng cáp và suy hao tính theo công suất phát, gain và giới hạn đầu vào của đúng model; không nối TX công suất cao trực tiếp vào RX. Hai nút vô tuyến độc lập thường cần hai RF transceiver và kế hoạch clock/timing riêng.

## 7. Nếu cần mua board

| Lựa chọn | Khi nên chọn | Điều kiện |
|---|---|---|
| Giữ DE2-115 + USRP hiện có | Đồ án tập trung HDL, fixed-point, tài nguyên, thuật toán | Làm theo frame trước; chưa cần mua FPGA |
| USRP X310 + RF daughterboard phù hợp | Muốn modem FPGA nối RF trực tiếp, khai thác UHD/RFNoC | Có kinh phí, toolchain phù hợp; xác định ngân sách tài nguyên còn trống |
| ZC706 + AD-FMCOMMS3-EBZ | Muốn thiết kế SoC/FPGA và RF chain chủ động | ADI có HDL/reference design cho tổ hợp này; cần thiết lập DMA, Linux/no-OS và Vivado [S9, S10] |
| ZedBoard + AD-FMCOMMS3-EBZ | Prototype nhỏ, đã có board | Cũng có reference design, nhưng cần đánh giá dư tài nguyên cho receiver trước khi chọn [S9] |

Không cần RFSoC chỉ vì đề tài có chữ 6G. Với mục tiêu ban đầu 2 MS/s, quyết định nâng cấp nên đến từ detector, memory bandwidth hoặc nhu cầu chạy độc lập đã đo được. Danh sách trên là lựa chọn kỹ thuật, chưa phải báo giá hoặc xác nhận tồn kho.

## 8. Kế hoạch kiểm chứng và tiêu chí nghiệm thu

| Mốc | Kiểm chứng | Tiêu chí đề xuất |
|---|---|---|
| 1 | Float TX/RX, không nhiễu | Khôi phục đúng bits; sai số transform ở mức numerical tolerance |
| 2 | RTL với fixed-point reference | Bit-exact trên vectors định trước, random, biên độ sát ngưỡng và backpressure |
| 3 | Loopback số trên board | Không lỗi trên ít nhất 10^6 bit ở điều kiện không nhiễu; không mất/thừa mẫu |
| 4 | Fixed-point so với float | Mục tiêu mất ≤ 0.5 dB tại BER 10^-3 trên cùng kênh và detector, nếu đường BER đạt mốc này |
| 5 | Tài nguyên/timing | Báo cáo post-fit LE/RAM/multiplier, Fmax/timing, cycles/frame và clipping |
| 6 | RF theo frame | Đo BER/EVM, timing/CFO, mức gain và rate thực; đối chiếu raw I/Q với reference |
| 7 | Streaming, nếu thuộc phạm vi | 2 MS/s mỗi chiều yêu cầu, chạy 60 s không UHD overrun/underrun, không mất sequence/frame |
| 8 | So sánh OFDM | Cùng Fs, công suất, payload rate hoặc báo đầy đủ overhead; nêu rõ receiver của từng hệ |

Ma trận kênh: AWGN → đa đường tĩnh → đa đường có Doppler. Bắt đầu 4 đường, delay tối đa 4 µs (< CP 8 µs), Doppler từng đường trong khoảng ±1.5 kHz. Cần cả integer và fractional Doppler; sweep SNR 0–25 dB. Tại mỗi điểm BER, mục tiêu thu ≥200 lỗi hoặc chạm giới hạn 10^7 bit; báo số bit và không diễn giải zero-error thành BER bằng 0 tuyệt đối.

Các giá trị kênh trên là test profile tự thiết kế, không tự gọi là profile chuẩn 3GPP. Doppler nên được tạo bằng channel emulator số; người đi bộ trong phòng không tạo được kịch bản high-mobility tương đương. Với ví dụ fc = 3.5 GHz, v = 500 km/h, Doppler cực đại theo v·fc/c khoảng 1.62 kHz.

## 9. Lộ trình ước lượng

Ước lượng cho một người đã có nền tảng RTL/DSP; phụ thuộc toolchain và thời gian debug.

1. Tuần 1–2: xác định USRP, golden model, CP/scaling/ordering, OFDM baseline.
2. Tuần 3–4: FFT/IFFT, RAM transpose, TX và fixed-point verification.
3. Tuần 5–6: RX transforms, loopback DE2-115, báo cáo tài nguyên.
4. Tuần 7–8: RF qua PC/USRP, đồng bộ và channel estimation trên PC.
5. Tuần 9–12 trở đi: detector FPGA và/hoặc Ethernet streaming; chỉ cam kết phạm vi sau profile tài nguyên và cycles.

Mốc báo cáo tối thiểu có giá trị: TX/RX transforms trên FPGA + RF test theo frame + detector tham chiếu + đánh giá fixed-point. Mốc hoàn chỉnh thời gian thực: phải có detector, đồng bộ, giao tiếp liên tục và số đo không mất mẫu.

## 10. Thông tin cần xác định trước khi khóa v1.0

- USRP model, revision, RF daughterboard, số thiết bị và phụ kiện clock có sẵn.
- Thời hạn, ngân sách và yêu cầu bắt buộc: FPGA TX, FPGA TX/RX, hay toàn bộ receiver độc lập.
- Toolchain đang dùng, IP FFT/MAC có sẵn và license; chưa chốt phiên bản Quartus/Vivado/ISE trước khi xác minh support.
- Có cần Δf chuẩn 15 kHz, tần số RF cụ thể, FEC hoặc mục tiêu thông lượng lớn hơn không.

Khi máy tính đã cài UHD và kết nối thiết bị, chạy `uhd_find_devices` rồi `uhd_usrp_probe` để lấy model/motherboard/daughterboard. Chưa có lệnh nào được thực thi lên USRP trong bản đánh giá này.

## Nguồn kỹ thuật

- [S1 — ITU IMT-2030](https://www.itu.int/en/ITU-R/study-groups/rsg5/rwp5d/pages/imt.aspx)
- [S2 — MathWorks: OTFS Modulation, biến đổi và CP](https://www.mathworks.com/help/comm/ug/otfs-modulation.html)
- [S3 — Terasic DE2-115](https://www.terasic.com.tw/cgi-bin/page/archive.pl?CategoryNo=139&Language=English&No=502&PartNo=1)
- [S4 — Altera Cyclone IV E](https://www.altera.com/products/fpga/cyclone-iv/e)
- [S5 — UHD: USRP2 và N2x0](https://files.ettus.com/manual/page_usrp2.html)
- [S6 — UHD: FPGA build và FPGA theo từng model](https://files.ettus.com/manual/gen3_fpga_build.html)
- [S7 — Ettus X310](https://www.ettus.com/all-products/x310-kit/)
- [S8 — Ettus: RFNoC development](https://kb.ettus.com/index.php?action=pdfbook&format=single&title=Getting_Started_with_RFNoC_Development)
- [S9 — ADI: FMCOMMS2/3/4 HDL và supported carriers](https://analogdevicesinc.github.io/hdl/projects/fmcomms2/index.html)
- [S10 — ADI: AD-FMCOMMS3-EBZ](https://www.analog.com/en/resources/evaluation-hardware-and-software/evaluation-boards-kits/eval-ad-fmcomms3-ebz.html)
