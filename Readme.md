# Commit 1 Reflection Notes

Di bagian ini ngerjain web server sederhana pakai Rust. Awalnya program cuma nerima koneksi doang sekarang udah bisa baca request yang masuk dari client.

Bagian paling penting di sini itu function handle_connection. Jadi setiap ada client yang nyambung ke server, koneksi itu langsung diproses lewat function ini. Di dalamnya aku pakai BufReader buat ngebaca data dari TcpStream, karena jauh lebih enak kalau request dibaca per baris dibanding baca raw byte langsung.

Alur yang dipakai di bagian pembacaan request itu kurang lebih:

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

      println!("Request: {:#?}", http_request);
}
```

### Kalau dijelasin :

1. .lines() dipakai buat mecah request jadi baris-baris teks
2. .map(|result| result.unwrap()) ngambil isi tiap baris dari Result
3. .take_while(|line| !line.is_empty()) bikin proses baca berhenti pas ketemu baris kosong, yang jadi penanda akhir header HTTP
4. .collect() ngumpulin semua baris itu ke dalam vector supaya gampang dilihat atau diproses lagi

Untuk sekarang servernya memang baru sampai tahap baca request dan nampilin isinya ke terminal

### Insight dari bagian ini:

1. HTTP request ternyata bentuk dasarnya text biasa aja
2. Struktur header itu konsisten dan dipisahkan oleh baris kosong
3. Cara kerja server itu terus nunggu koneksi baru di loop, jadi selama program jalan dia siap nerima request kapan pun.

### Refleksi pribadi:

Menurutku ini fondasi yang penting. Di sini mulai mayan kebayang gimana alur komunikasi client-server sebenarnya terjadi dari bawah bukan cuma pakai framework jadi.

# Commit 2 Reflection Notes

Di commit ini servernya baru mulai beneran ngasih response yang bisa dirender oleh browser. Sebelumnya server hanya baca HTTP request dan menampilkannya di terminal, sekarang `handle_connection` sudah buat HTTP response sederhana yang berisi halaman HTML dari file `hello.html`.

Kalo di run `cargo run` di terminal dan buka URL `http://127.0.0.1:7878` akan muncul ini. Dimana browser sudah muncullin halaman html yang dibuat

![Commit 2 screen capture](/assets/images/commit2.png)

### Perubahan yang paling terasa ada di bagian ini:

```rust
let status_line = "HTTP/1.1 200 OK";
let contents = fs::read_to_string("hello.html").unwrap();
let length = contents.len();

let response =
    format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

stream.write_all(response.as_bytes()).unwrap();
```

### Kalau dijelasin :

1. `status_line` dipakai untuk ngasih tahu browser bahwa request berhasil diproses dengan status `200 OK`
2. `fs::read_to_string("hello.html").unwrap()` membaca isi file HTML sebagai string, jadi konten halaman tidak ditulis langsung di dalam kode Rust
3. `contents.len()` menghitung panjang body response yang nanti dikirim lewat header `Content-Length`
4. `format!` menyusun response HTTP lengkap, mulai dari status line, header, baris kosong pemisah header dan body, lalu isi HTML
5. `stream.write_all(response.as_bytes()).unwrap()` mengirim response tersebut ke browser lewat koneksi TCP

### Insight dari bagian ini:

1. Browser butuh response HTTP yang formatnya benar supaya bisa menampilkan halaman
2. Header `Content-Length` penting karena bantu browser mengetahui ukuran body yang diterima.
3. Baris kosong `\r\n\r\n` punya peran penting sebagai pemisah antara header dan body dalam HTTP response
4. File HTML bisa dipisahkan dari kode server, sehingga struktur program jadi lebih rapi dan kontennya lebih mudah diubah

### Refleksi pribadi:

Menurutku bagian ini bikin konsep web server jadi lebih kebayang. Untuk nampilin halaman sederhana di browser, server perlu baca request, menyusun response HTTP dengan format yang sesuai, lalu ngirim HTML sebagai body response.

# Commit 3 Reflection Notes

Di commit ini servernya mulai bisa bedain request yang valid dan yang tidak. Sebelumnya, apa pun path yang dibuka di browser, server tetap selalu ngirim `hello.html`. Sekarang server ngecek request line dulu, jadi kalau user buka `/` server akan ngasih halaman utama, sedangkan kalau buka path lain seperti `/bad`, server akan ngasih halaman error `404.html`

Kalo di run `cargo run` terus buka URL `http://127.0.0.1:7878/bad`, browser akan nampilin halaman not found seperti ini

![Commit 3 screen capture](/assets/images/commit3.png)

### Perubahan ada di bagian ini:

```rust
let request_line = buf_reader.lines().next().unwrap().unwrap();

let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
    ("HTTP/1.1 200 OK", "hello.html")
} else {
    ("HTTP/1.1 404 NOT FOUND", "404.html")
};

let contents = fs::read_to_string(filename).unwrap();
let length = contents.len();

let response =
    format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

stream.write_all(response.as_bytes()).unwrap();
```

### Kalau dijelasin :

1. `request_line` dipakai buat ambil baris pertama dari HTTP request, contohnya `GET / HTTP/1.1`
2. Kalau `request_line` sama dengan `GET / HTTP/1.1`, berarti user minta halaman utama dan server balikin `hello.html`
3. Kalau request-nya bukan `/`, server balikin status `404 NOT FOUND` dan isi halaman dari `404.html`
4. `status_line` dan `filename` dipisah pakai tuple supaya server bisa nentuin status response dan file HTML yang akan dikirim
5. Setelah file yang benar ditentukan, alur response-nya tetap sama seperti commit sebelumnya, yaitu baca file, hitung `Content-Length`, lalu kirim response ke browser

### Kenapa perlu refactoring:

Refactoring dibutuhkan supaya kode tidak terlalu banyak pengulangan. Kalau response untuk `200 OK` dan `404 NOT FOUND` ditulis terpisah semua, nanti bagian membaca file, menghitung length, membuat response, dan `write_all` bisa jadi dobel. Dengan refactoring, yang dibedakan cukup `status_line` dan `filename`, sedangkan proses membuat dan mengirim response tetap satu alur yang sama.

Menurutku cara ini juga bikin kode lebih gampang dikembangin. Misalnya nanti mau nambah path lain seperti `/about` atau `/contact`, tinggal tambahin kondisi untuk milih file dan status response yang sesuai tanpa harus nulis ulang semua proses kirim response.

### Insight dari bagian ini:

1. HTTP request line bisa dipakai untuk menentukan halaman mana yang diminta oleh browser
2. Server tidak harus selalu ngirim response yang sama, tapi bisa milih response berdasarkan isi request
3. Status code seperti `200 OK` dan `404 NOT FOUND` penting karena ngasih tahu browser hasil dari request yang dikirim
4. Refactoring bikin kode lebih rapi karena bagian yang sama cukup ditulis sekali

### Refleksi pribadi:

Menurutku bagian ini mulai nunjukin bentuk web server yang lebih realistis. Walaupun masih sederhana, servernya udah punya logic untuk validasi request dan ngasih response yang beda tergantung path yang diminta

# Commit 4 Reflection Notes

Di commit ini servernya dicoba untuk simulasi masalah yang muncul kalau server masih berjalan di single thread. Caranya dengan nambah route `/sleep` yang sengaja dibuat delay selama 10 detik sebelum ngasih response. Dari sini kelihatan kalau request lain jadi ikut ketahan selama request `/sleep` belum selesai diproses.

### Perubahan ada di bagian ini:

```rust
let (status_line, filename) = match &request_line[..] {
    "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"),
    "GET /sleep HTTP/1.1" => {
        thread::sleep(Duration::from_secs(10));
        ("HTTP/1.1 200 OK", "hello.html")
    }
    _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
};
```

### Kalau dijelasin :

1. `match &request_line[..]` dipakai buat ngecek request line dan nentuin response berdasarkan path yang diminta
2. Kalau request-nya `GET / HTTP/1.1`, server langsung balikin `hello.html` seperti biasa
3. Kalau request-nya `GET /sleep HTTP/1.1`, server akan berhenti dulu selama 10 detik pakai `thread::sleep(Duration::from_secs(10))`
4. Setelah delay selesai, baru server balikin response `200 OK` dengan file `hello.html`
5. Kalau path-nya selain `/` dan `/sleep`, server tetap balikin `404.html`

### Apa masalah yang kelihatan:

Masalahnya muncul karena server masih single thread. Artinya server cuma bisa memproses satu request dalam satu waktu. Kalau ada satu request yang lama, misalnya `/sleep`, request berikutnya harus nunggu dulu sampai proses sebelumnya selesai.

Saat dicoba buka `http://127.0.0.1:7878/sleep` di satu tab, lalu buka `http://127.0.0.1:7878` di tab lain, halaman utama juga ikut loading. Padahal halaman utama sebenarnya tidak punya delay. Ini terjadi karena server belum bisa menjalankan beberapa request secara bersamaan.

### Insight dari bagian ini:

1. Single thread server punya keterbatasan kalau ada request yang prosesnya lama
2. Satu request yang lambat bisa bikin request lain ikut menunggu
3. `thread::sleep` di sini bukan fitur utama server, tapi dipakai untuk simulasi request yang lambat
4. Dari masalah ini mulai kelihatan kenapa web server butuh concurrency atau thread pool

### Refleksi pribadi:

Menurutku pada bagian ini penting karena masalahnya langsung terasa saat dicoba di browser. Awalnya server kelihatan sudah bisa handle beberapa route, tapi ternyata kalau satu request dibuat lama, semua request lain ikut kena efeknya.
