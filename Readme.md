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
