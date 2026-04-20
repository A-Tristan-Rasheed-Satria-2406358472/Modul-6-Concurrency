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
