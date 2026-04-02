# Kiến trúc Fields (Trường Hữu Hạn) trong Plonky3

Trong toán học mật mã và hệ thống chứng minh không tri thức (ZK-proofs), **trường hữu hạn (Finite Fields)** là phần nền tảng cốt lõi định nghĩa hầu hết các biểu diễn và phép toán. Plonky3 thiết kế không gian cho `Fields` một cách cẩn thận để dung hòa giữa tính trừu tượng (để hỗ trợ nhiều loại trường khác nhau) và tính tối ưu hiệu năng phần cứng.

Tài liệu này tập trung vào thiết kế và các module của không gian "Fields" trong Plonky3.

## Nền tảng Core: Crate `p3-field`

Crate `field` (nằm trong thư mục `/field`) cung cấp một khuôn khổ (framework) và hệ thống Traits hoàn chỉnh cho tất cả các triển khai trường hữu hạn sau này.

### Các Traits (Giao diện) Chính

1. **`PrimeCharacteristicRing`:** 
   Là hạt nhân của các phép toán đại số. Nó định nghĩa các tính chất của một vành giao hoán với đặc số nguyên tố. Trait này tích hợp sẵn các phép cộng, trừ, nhân và các giá trị cơ bản như `ZERO`, `ONE`, `TWO`, `NEG_ONE`. Ngoài ra, nó định nghĩa các hàm tính toán cơ bản nhưng được dùng rất nhiều và có thể được tối ưu riêng như `double()`, `halve()`, `square()`, `cube()`, hay các biến thể exponentiation (luỹ thừa).

2. **`Algebra<F>` và `BasedVectorSpace<F>`:**
   Cho phép định nghĩa một không gian vector hoặc cấu trúc đại số trên một trường cơ sở `F`. Điều này đặc biệt hữu dụng khi xây dựng các **Trường mở rộng (Extension Fields)**. Nhờ `Algebra`, ta có thể tính toán tích vô hướng hỗn hợp (`mixed_dot_product`) và tổ hợp tuyến tính hiệu quả.

3. **`PrimeField` và `PrimeField32` / `PrimeField64`:**
   Các trường hữu hạn bậc gốc (cấp `p`). `PrimeField32` và `PrimeField64` phân loại các trường có thể biểu diễn vừa vặn trong số biến tương ứng (32-bit hoặc 64-bit), giúp trình biên dịch áp dụng các chỉ thị thanh ghi hiệu quả hơn.

4. **`TwoAdicField`:**
   Đặc hữu cho họ FFTs (Fast Fourier Transform). Trait này định nghĩa một trường có chứa phần tử căn nguyên thuỷ bậc `2^n`. Nó quản lý quá trình sinh (Generator) và các luân chuyển của tập n-th roots of unity, cực kỳ quan trọng cho các sơ đồ commit dạng FRI.

5. **`PackedField`:**
   Đây là trait thể hiện khả năng SIMD (Single Instruction, Multiple Data). Nó cho phép gộp nhiều phần tử trường vào chung một thanh ghi bộ nhớ tĩnh lớn (như AVX2 = 256 bits, AVX-512 = 512 bits, Neon) và thực thi phép toán song song định kiểu.

### Các Modules Tiện Ích Trong `p3-field`
- **`batch_inverse.rs`**: Áp dụng nghịch đảo hàng loạt qua thuật toán Montgomery, thay vì tốn chi phí nghịch đảo từng phần tử.
- **`exponentiation.rs`**: Cung cấp hàm bình phương và nhân chuẩn.
- **`coset.rs`**: Quản lý tập thay thế tuyến tính (cosets).

---

## Các Trường Cụ Thể (Concrete Fields)

Thay vì tích hợp chung, Plonky3 phân mảnh các loại trường thành các crate khác nhau. Mỗi crate thường được thiết kế cực kì tối ưu (Sử dụng cả mã Assembly hoặc Intrinsics của Rust) theo đặc tả phần cứng:

### 1. Họ các trường nhỏ 31-Bit (Small Prime Fields)
Được sinh ra để tối ưu hóa đặc điểm của CPU hiện đại (đó là làm việc với 32-bit/64-bit word):
- **`baby-bear`**: Trường nguyên tố với p = 15 * 2^27 + 1. Thân thiện với ZKVM vì số nguyên tố này thích hợp với radix-2 FFT và tính toán hiệu năng cực cao trên số học 32-bit.
- **`koala-bear`**: Tương tự Baby-bear (p = 2^31 - 2^24 + 1), dùng cho mục đích khai thác tập căn hai có tính năng song song khác biệt.
- **`mersenne-31`**: Trường cấu trúc dựa trên số Mersenne nhỏ thứ 8 p = 2^31 - 1. Số học trên số Mersenne cực nhanh bằng việc sử dụng dị biến bitwise (dịch bit) thay cho phép modulo chia thông thường.
- **`monty-31`**: Có vẻ tối ưu cho kiến trúc modulo Montgomery 31-bit.

### 2. Họ trường lớn (Large Fields)
- **`goldilocks`**: Đặc trưng của Polygon Zero, p = 2^64 - 2^32 + 1. Số này được lấy tên "Goldilocks" bởi nó vừa đủ lớn cho mặt số học, nhưng cũng lọt vừa đúng vào thanh ghi 64-bit để cho tốc độ xử lý vô địch trong nhóm trường kích thước cỡ trung.
- **`bn254`**: Không thuộc loại thân thiện tối cao với STARK/FFT, nhưng lại phổ biến nhất với SNARKs và hệ sinh thái Smart Contract của Ethereum. Việc có sẵn bn254 giúp Plonky3 dễ dàng kết hợp và xác minh chứng minh STARK trực tiếp bên trong một mạch SNARK (như Halo2/Groth16).

---

## Testing Framework (`field-testing`)

Là một crate độc lập dùng để hỗ trợ kiểm thử thống nhất:
- Cung cấp các random generators (Proptest) và các hàm kiểm tra xem một cấu trúc tự định nghĩa có thỏa mãn chuẩn `PrimeField` hay `TwoAdicField` đã quy định trong module `field` hay không.
- Thử nghiệm các hệ phương trình đồng nhất để kiểm định lỗi tràn số (overflow buffer), điều cực kỳ dễ gặp khi thiết lập lại assembly operations thông qua tập lệnh AVX2 hay AVX-512.

## Tóm gọn lại

Thiết kế cơ sở cho Fields của Plonky3 thiên hoàn toàn về tính **chất lượng linh kiện (composability)** và tính **kín kẽ của hệ thống phần cứng (hardware sympathy)**.
Người dùng có thể bắt đầu mô tả AIR (Algebraic Intermediate Representation) với một `PrimeField` chung chung. Sau đó, họ tùy chọn compile hệ thống với `baby-bear` hay `goldilocks` mà không cần viết lại Logic Proof, chỉ dùng các Argument chuyển đổi.
