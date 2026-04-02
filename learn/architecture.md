# Kiến trúc Dự án Plonky3

Dự án **Plonky3** là một bộ công cụ (toolkit) cung cấp các thành phần cơ bản (primitives) mã nguồn mở dùng để phát triển các hệ thống IOPs đa thức (Polynomial Interactive Oracle Proofs - PIOPs). Nó chủ yếu được sử dụng để xây dựng hệ thống chứng minh ZK (Zero-Knowledge) zkVMs dựa trên STARK, và về nguyên tắc cũng có thể được sử dụng cho các mạch PLONK hoặc cấu trúc PIOP khác. Không giống như các framework tĩnh, Plonky3 có thiết kế mô-đun hoá (modular) cao với nhiều crates chuyên biệt kết hợp lại với nhau.

Dưới đây là một bản phác thảo về kiến trúc và các thành phần chính của dự án dựa trên cấu trúc các thư mục mã nguồn và Rust crates có trong workspace.

## Tổng quan Kiến trúc

Plonky3 được chia thành nhiều không gian con chuyên biệt, gọi là các `crates`. Mỗi module giải quyết một phần độc lập của toàn bộ quy trình xây dựng Proof Systems, cho phép người dùng lắp ráp hệ thống chứng minh tuỳ chỉnh theo nhu cầu (ví dụ: thay đổi loại trường, hàm băm, hoặc thuật toán đa thức).

Các nhóm thành phần chính bao gồm:

### 1. Fields (Trường Hữu Hạn)
Hỗ trợ đa dạng các trường (Fields) và tiện ích mở rộng tập lệnh để tối ưu hiệu năng:
- **`baby-bear`**, **`koala-bear`**: Các trường tổng quát 31-bit.
- **`mersenne-31`**, **`monty-31`**: Hỗ trợ tập trung vào trường Mersenne số nguyên tố nhỏ.
- **`goldilocks`**: Cung cấp cấu trúc hoạt động trong trường Goldilocks (trường được tối ưu tốt trên CPU 64-bit).
- **`bn254`**: Dành cho đường cong Elliptic Curve bn254, phổ biến trong việc xác minh trên Ethereum.
- **`field`**, **`field-testing`**: Các định nghĩa interface lõi của trường (traits) và bộ công cụ phục vụ việc test chức năng của fields.

Mỗi trường đều có hỗ trợ mở rộng phần cứng thông qua các tập lệnh AVX2, AVX-512 và NEON.

### 2. Hashing & Symmetric Cryptography (Hàm băm & Mật mã đối xứng)
Thay vì hardcode vào một hàm băm nhất định, Plonky3 cung cấp nhiều tính năng Mật mã, cho phép developer "plug and play":
- **`poseidon1`**, **`poseidon2`**: Các thuật toán mã hóa Poseidon thường thân thiện với hệ thống ZK.
- **`keccak`**, **`sha256`**, **`blake3`**: Các thuật toán phổ biến truyền thống với tốc độ cao thực thi trên native CPU (`blake3` cũng đang được điều chỉnh cho các loại leaf nhỏ).
- **`rescue`**, **`monolith`**: Các dạng hàm băm thân thiện chuẩn ZK (ZK-friendly) khác.
- **`symmetric`**: Định nghĩa và cấu trúc cơ bản chung để quản lý các hàm băm đồng nhất.

*(Đối với một số hàm băm như Poseidon2, Blake3 hay Keccak, dự án còn cung cấp các hàm chuyên biệt tạo sẵn AIR (Algebraic Intermediate Representation) như `keccak-air`, `poseidon2-air`, `blake3-air`).*

### 3. Commitment Schemes & Trees (Lược đồ cam kết & Cây Merkle)
Xây dựng cơ sở lý thuyết để cam kết đa thức hoặc vector dữ liệu:
- **`merkle-tree`**: Cấu trúc Merkle tree tổng quát có thể tùy chỉnh hàm băm.
- **`fri`**: Fast Reed-Solomon Interactive Oracle Proof of Proximity (FRI) – giao thức cốt lõi đằng sau hệ thống chuẩn STARK để chứng minh khoảng cách đa thức.
- **`commit`**: Quản lý đa thức và mô hình scheme.
- **`whir`**: Quản lý liên quan đến Weighted Hash protocols và các giao thức chứng minh tương tác nhanh tương tự (VD: WHIR PCS).

### 4. Arithmetic & Polynomials (Toán học Đa thức & Biến đổi)
Các module thực hiện các tính toán đa thức với tốc độ cao:
- **`dft`** (Discrete Fourier Transform): Tính các biến đổi Fourier trên các trường 31-bit (Bao gồm Radix-2 DIT FFT và bộ Radix-2 Bowers FFT). Đóng vai trò cực kì quan trọng tối ưu hoá việc nội suy / đánh giá đa thức.
- **`interpolation`**: Kĩ thuật nội suy Barycentric và các mô hình nội suy tương tự.
- **`matrix`**: Định nghĩa và biến đổi thao tác trên cấu trúc ma trận nhằm hỗ trợ các tính toán bảng hệ thống.
- **`circle`**: Liên quan đến vòng tròn (Mersenne circle group FFT) để hỗ trợ Circle STARKs.
- **`mds`**: Thao tác sinh và nhân đối với các ma trận MDS (Maximum Distance Separable) dùng nhiều cho thuật toán băm kiểu (Poseidon, Rescue).

### 5. Proof Systems / PIOP Interfaces (Hệ thống Chứng minh ZK)
Các crates ghép nối với lý thuyết toán học lại thành một giao thức chuẩn (như STARKs):
- **`air`**: (Algebraic Intermediate Representation) Giao diện gốc để mô tả các mạch ràng buộc.
- **`uni-stark`**: Hệ thống cho Univariate STARKs (Chuẩn hoá theo thiết kế đa thức một biến).
- **`batch-stark`**: Quản lý batch logic khi cần tối ưu hiệu năng đối với các hàm batch verify/proofs.
- **`lookup`**: Cung cấp interface thực thi các kiến trúc tra bảng (Lookup arguments), làm giảm đáng kể kích thước của mạch/bảng chi phí (như Halo2/LogUp).
- **`challenger`**: Giao thức Fiat-Shamir (biến quá trình tương tác thành phi tương tác thông qua hàm băm), tạo ngẫu nhiên bảo mật trên proof.

### 6. Tools, Tests & Utilities
- **`util`**, **`multilinear-util`**: Chứa helper function liên quan đến multilinear, mảng và các utils tiện ích phục vụ mã nguồn.
- **`maybe-rayon`**: Crates hỗ trợ cơ chế chạy song song (Paralellism) giúp người dùng thoải mái đóng/mở công cụ tối ưu bằng Rayon, phục vụ việc generate Proofs hiệu năng trên đa luồng tùy môi trường.
- **`examples`**: Cung cấp các ví dụ để tham khảo cách thiết lập lệnh từ đầu. VD: Dùng `prove_prime_field_31` qua lệnh cargo.

## Pipeline quá trình chứng minh điển hình (Typical Proof Pipeline)
Trong kiến trúc của Plonky3, một quá trình Prove -> Verify STARK cơ bản thường trải qua các module:
1. Xác định ma trận **Trace** (`matrix`, `air`) bằng đại số.
2. Thêm quá trình băm fiat-shamir vào `challenger` để tạo ra challenge từ người chứng minh.
3. Nội suy / Đánh giá đa thức bằng xử lý **FFT/DFT** (`dft`, `interpolation`).
4. Gắn kết cấu trúc dữ liệu lên Cây Merkle thông qua PCS như `merkle-tree`, `fri`, và tuỳ chỉnh hàm băm mong muốn như `poseidon2`, `keccak`.
5. Sinh ra cấu trúc STARK tương ứng (bằng `uni-stark`).

Tóm lại, **Plonky3** cung cấp đầy đủ các khối linh kiện cấu thành để thiết kế những Proof System phi tập trung, từ những module tính toán cấp thấp như phép nhân phân đoạn đến cấp độ High-level nhất như các máy ảo Zero-Knowledge Virtual Machines (zkVM).
