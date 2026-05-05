# Tutorial 8: gRPC in Rust - Reflection

---

### 1. Perbedaan Utama antara Unary, Server Streaming, dan Bi-directional Streaming RPC
*   **Unary RPC:** Klien mengirimkan satu permintaan tunggal ke server dan menerima satu respons balik, mirip dengan pemanggilan fungsi konvensional atau *request-response* pada REST API. Metode ini cocok untuk operasi yang bersifat transaksional tunggal, seperti *Create, Read, Update, Delete* (CRUD) atau autentikasi pengguna.
*   **Server Streaming RPC:** Klien mengirimkan satu permintaan, tetapi server membalas dengan serangkaian respons berupa *stream* data. Sangat ideal untuk kasus di mana ukuran data respons terlalu besar untuk dikirim sekaligus (seperti riwayat transaksi yang masif) atau saat mengirimkan data pembaruan *real-time* satu arah (misalnya, *live score* olahraga atau pergerakan harga saham).
*   **Bi-directional Streaming RPC:** Klien dan server berbagi koneksi dua arah secara asinkron; keduanya dapat mengirimkan dan membaca pesan kapan saja secara independen. Pendekatan ini sangat cocok untuk interaksi yang bersifat kontinu dan dinamis, seperti aplikasi *chatting real-time*, gim *multiplayer*, atau sinkronisasi data dua arah.

### 2. Pertimbangan Keamanan dalam Implementasi Layanan gRPC di Rust
*   **Enkripsi Data:** Karena gRPC menggunakan HTTP/2, lapisan *Transport Layer Security* (TLS) bersifat wajib di lingkungan produksi untuk mencegah *eavesdropping* (penyadapan) data. Pemanfaatan mTLS (*Mutual TLS*) juga disarankan untuk memastikan identitas klien maupun server tervalidasi.
*   **Autentikasi (Authentication):** Dapat diimplementasikan menggunakan Token JWT (*JSON Web Tokens*) melalui *metadata* (setara dengan *headers* pada HTTP). Di dalam Rust (menggunakan library `tonic`), hal ini ditangani melalui konsep *Interceptors* untuk memvalidasi kredensial pada setiap *request*.
*   **Otorisasi (Authorization):** Setelah identitas diverifikasi, sistem perlu menerapkan *Role-Based Access Control* (RBAC) pada *layer* layanan untuk memastikan pengguna hanya dapat mengakses RPC *method* yang sesuai dengan izin (hak akses) mereka.

### 3. Tantangan dalam Menangani Bi-directional Streaming di Rust gRPC
*   **Manajemen Status (State Management):** Mengelola status koneksi setiap klien dalam memori yang diakses secara bersamaan (konkurensi) memerlukan kehati-hatian untuk menghindari kebocoran memori (*memory leaks*) saat koneksi terputus.
*   **Konkurensi dan Deadlocks:** Dalam Rust, memastikan akses ke *shared state* (seperti *hash map* untuk menyimpan *channel* klien) sangat ketat. Pemilihan primitif sinkronisasi yang tidak tepat (seperti menggunakan `std::sync::Mutex` alih-alih `tokio::sync::Mutex` pada fungsi asinkron) dapat memicu *deadlock*.
*   **Penanganan Error:** Ketika salah satu pihak terputus secara tiba-tiba, *server* harus mampu mendeteksi *network partition* dan membersihkan sumber daya (*resources*) yang terpakai dengan bersih tanpa mengganggu aliran (*stream*) klien lainnya.

### 4. Kelebihan dan Kekurangan Penggunaan `tokio_stream::wrappers::ReceiverStream`
*   **Kelebihan:** Sangat memudahkan proses integrasi antara *channel* asinkron buatan Tokio (`mpsc`) dengan antarmuka (*trait*) `Stream` yang diwajibkan oleh kerangka kerja gRPC (`tonic`). Hal ini memungkinkan pembuatan *stream* dari operasi *background task* secara rapi dan modular.
*   **Kekurangan:** Penambahan lapisan *wrapper* ini dapat memberikan sedikit beban memori (*overhead*). Selain itu, kita harus berhati-hati dalam menentukan ukuran batas (*buffer size*) pada `mpsc`; jika antrean penuh, *backpressure* akan terjadi dan dapat memblokir proses *producer* di dalam *server*.

### 5. Struktur Kode Rust gRPC untuk Mendorong Penggunaan Ulang (Code Reuse) dan Modularitas
*   **Pemisahan Layer:** Pisahkan antara *Network/Transport Layer* (kode yang berurusan langsung dengan `tonic` dan protobuf) dengan *Business Logic Layer* (layanan domain inti). 
*   **Modularitas Protobuf:** Definisi `.proto` dapat diekstraksi ke dalam *crate* (*library*) tersendiri. Hal ini sangat berguna jika kita menerapkan arsitektur *microservices* menggunakan bahasa yang berbeda (misalnya interaksi antara layanan Rust dengan Spring Boot atau Django); mereka dapat berbagi *crate/repository* protobuf yang sama tanpa harus menduplikasi kontrak.
*   **Dependency Injection:** Terapkan penggunaan *Trait* di Rust untuk layanan repositori atau layanan eksternal, sehingga kode mudah diuji (*unit testing*) menggunakan sistem *mocking*.

### 6. Langkah Tambahan untuk Logika Pemrosesan Pembayaran yang Lebih Kompleks pada `MyPaymentService`
*   **Penerapan Idempotency:** Menambahkan kunci idempotensi (*idempotency keys*) pada *request* untuk mencegah pembayaran berganda (*double charging*) jika terjadi *retry* akibat gangguan jaringan.
*   **Integrasi Basis Data (Database Integration):** Menyimpan status pembayaran (misal: *Pending, Success, Failed*) ke dalam basis data relasional seperti PostgreSQL dengan penerapan kaidah normalisasi dan prinsip ACID transaksi untuk menjaga konsistensi data.
*   **Pola Saga / Distribusi Transaksi:** Jika pembayaran memicu proses di *microservice* lain (misalnya pembaruan inventaris), diperlukan mekanisme kompensasi transaksi apabila terjadi kegagalan sistemik.

### 7. Dampak gRPC Terhadap Arsitektur Sistem Terdistribusi dan Interoperabilitas
gRPC mengubah paradigma dari arsitektur berorientasi dokumen menjadi pemanggilan prosedur yang ketat. Penggunaan Protocol Buffers (*protobuf*) memastikan kontrak API (*schema*) sangat jelas dan terikat kuat. Dari segi interoperabilitas, gRPC memudahkan integrasi *polyglot* (multi-bahasa pemrograman); klien dalam Java, Python, atau Go dapat di-*generate* secara otomatis dari *file* `.proto` yang sama, meminimalkan ketidaksesuaian tipe data antar *microservice*. Namun, interaksi langsung gRPC dengan peramban (*browser* web/REST) masih membutuhkan komponen tambahan seperti *gRPC-Gateway*.

### 8. Kelebihan dan Kekurangan HTTP/2 (Mendasari gRPC) vs HTTP/1.1 (REST API)
*   **Kelebihan HTTP/2:** Mendukung *Multiplexing* (memungkinkan banyak *request/response* secara bersamaan dalam satu koneksi TCP), kompresi *header* secara bawaan, serta pengiriman data berbentuk biner yang membuatnya jauh lebih efisien dan ringan secara ukuran *payload* dibanding HTTP/1.1 yang berbasi teks.
*   **Kekurangan HTTP/2:** *Debugging* lebih sulit karena wujudnya yang biner (tidak bisa dibaca secara kasatmata melalui alat inspeksi web standar tanpa *decoder*). Selain itu, HTTP/2 rentan terhadap masalah *TCP Head-of-Line Blocking*, di mana satu paket jaringan yang hilang dapat menunda seluruh *stream* pada koneksi TCP yang sama.

### 9. Perbandingan Model Request-Response REST API dengan Bidirectional Streaming gRPC
*   **REST API:** Bersifat inisiatif klien (*client-driven*). Jika klien membutuhkan pembaruan *real-time*, mereka harus melakukan *polling* (bertanya berulang kali) yang boros *bandwidth*, atau menggunakan WebSockets yang harus dikonfigurasi terpisah. Responsivitas terbatas pada seberapa sering klien bertanya.
*   **gRPC Bidirectional:** Setelah koneksi dibuat, *stream* akan tetap terbuka. Klien dan server saling memegang *channel* untuk mengirimkan data segera setelah data tersebut tersedia. Hasilnya adalah latensi yang sangat rendah, *overhead* yang minim, dan pengiriman pesan *real-time* secara alami di level protokol.

### 10. Implikasi Pendekatan Berbasis Skema (Protocol Buffers) vs JSON Tanpa Skema
*   **Kinerja dan Ukuran:** Payload Protocol Buffers diserialisasi ke dalam bentuk biner. Proses penyandian (*encoding*) dan penguraiannya (*decoding*) jauh lebih cepat serta memakan *bandwidth* jaringan yang jauh lebih kecil dibandingkan *parsing* JSON berbasis teks.
*   **Validasi dan Evolusi:** Protobuf memaksakan disiplin yang ketat; setiap entitas memiliki tipe data bawaan (*strongly-typed*). Ini meminimalisasi *error* tak terduga (*runtime error*) di produksi akibat perbedaan ekspektasi kolom JSON. Sebaliknya, JSON menawarkan fleksibilitas yang lebih tinggi saat tahap eksplorasi awal karena pengembang dapat menambahkan atau mengubah kunci secara bebas tanpa perlu melakukan kompilasi ulang antarmuka skema.