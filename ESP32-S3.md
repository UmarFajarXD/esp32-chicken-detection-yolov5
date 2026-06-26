# Pseudocode untuk Esp32-S3

pseudocode dari program deteksi objek YOLOv5n pada ESP32-S3.
---

# Algoritma 1. Inisialisasi Sistem ESP32-S3

```
ALGORITMA Inisialisasi_Sistem

BEGIN

Mulai komunikasi Serial dengan baudrate 115200

Inisialisasi TFT LCD

Atur orientasi layar

Atur ukuran font

Bersihkan layar

Tampilkan judul sistem

Periksa ketersediaan PSRAM

IF PSRAM tersedia THEN

    Alokasikan memori gambar

    Alokasikan memori tensor input

ELSE

    Tampilkan pesan
    "PSRAM Initialization Failed"

    Hentikan program

ENDIF

END
```

---

# Algoritma 2. Koneksi WiFi

```
ALGORITMA Koneksi_WiFi

BEGIN

Masukkan SSID

Masukkan Password

Hubungkan ESP32 ke WiFi

WHILE status WiFi belum Connected DO

    Tunggu 500 ms

    Tampilkan status koneksi

ENDWHILE

Tampilkan alamat IP ESP32

END
```

---

# Algoritma 3. Mengambil Gambar dari Server Kamera

```
ALGORITMA Ambil_Gambar

BEGIN

Kirim HTTP GET ke Server Kamera

IF respon berhasil THEN

    Terima seluruh data gambar

    Simpan gambar ke buffer

ELSE

    Tampilkan pesan
    "Download Image Failed"

ENDIF

END
```

---

# Algoritma 4. Resize Gambar

```
ALGORITMA Resize_Gambar

BEGIN

Baca gambar asli

Resize gambar

Ukuran awal
224 × 224 piksel

Menjadi

480 × 320 piksel

Simpan hasil resize

END
```

---

# Algoritma 5. Menampilkan Gambar pada TFT LCD

```
ALGORITMA Tampilkan_Gambar

BEGIN

FOR setiap piksel gambar DO

    Ambil nilai warna

    Tampilkan piksel pada TFT LCD

ENDFOR

END
```

---

# Algoritma 6. Konversi RGB565 menjadi RGB888

```
ALGORITMA Konversi_RGB565_RGB888

BEGIN

FOR setiap piksel DO

    Ambil nilai RGB565

    Pisahkan komponen

        Red

        Green

        Blue

    Ubah menjadi RGB888

    Simpan hasil

ENDFOR

END
```

---

# Algoritma 7. Normalisasi Input

```text
ALGORITMA Normalisasi_Gambar

BEGIN

FOR setiap piksel DO

    Red = Red / 255.0

    Green = Green / 255.0

    Blue = Blue / 255.0

Simpan ke tensor input

ENDFOR

END
```

---

# Algoritma 8. Inferensi YOLOv5

```text
ALGORITMA Inferensi_YOLO

BEGIN

Masukkan tensor input

Jalankan fungsi

Forward Pass

Simpan output model

END
```

---

# Algoritma 9. Parsing Output YOLO

```text
ALGORITMA Parsing_Output

BEGIN

jumlah_deteksi = 0

FOR setiap prediksi DO

    Ambil

        x_center

        y_center

        width

        height

        confidence

    IF confidence < threshold THEN

        Lewati prediksi

    ELSE

        Cari probabilitas kelas terbesar

        Hitung

        score = confidence × class probability

        IF score > threshold THEN

            Simpan

                Bounding Box

                Nama Kelas

                Confidence

            jumlah_deteksi++

        ENDIF

    ENDIF

ENDFOR

END
```

---

# Algoritma 10. Non-Maximum Suppression (NMS)

```text
ALGORITMA Non_Maximum_Suppression

BEGIN

Bandingkan seluruh Bounding Box

Hitung nilai IoU

IF IoU lebih besar dari threshold THEN

    Hapus Bounding Box dengan confidence lebih kecil

ENDIF

Ulangi hingga seluruh Bounding Box selesai dibandingkan

END
```

---

# Algoritma 11. Menggambar Bounding Box

```text
ALGORITMA Gambar_Bounding_Box

BEGIN

FOR setiap objek hasil deteksi DO

    Gambar kotak merah

    Tampilkan nama kelas

    Tampilkan nilai confidence

ENDFOR

END
```

---

# Algoritma 12. Menghitung Jumlah Objek

```
ALGORITMA Hitung_Jumlah_Objek

BEGIN

Reset seluruh counter

FOR setiap objek DO

    IF kelas = Person THEN

        person++

    ELSE IF kelas = Bottle THEN

        bottle++

    ELSE IF kelas = Cup THEN

        cup++

    ELSE IF kelas = Chair THEN

        chair++

    ELSE

        other++

    ENDIF

ENDFOR

END
```

---

# Algoritma 13. Menampilkan Informasi

```
ALGORITMA Tampilkan_Informasi

BEGIN

Hitung waktu inferensi

Tampilkan

    Inference Time

    Total Detection

    Person

    Bottle

    Chair

    Cup

    Other

END
```

---

# Algoritma 14. Program Utama (Loop)

```
ALGORITMA Program_Utama

BEGIN

Panggil Inisialisasi_Sistem

Panggil Koneksi_WiFi

WHILE TRUE DO

    Panggil Ambil_Gambar

    Panggil Resize_Gambar

    Panggil Tampilkan_Gambar

    Panggil Konversi_RGB565_RGB888

    Panggil Normalisasi_Gambar

    Panggil Inferensi_YOLO

    Panggil Parsing_Output

    Panggil Non_Maximum_Suppression

    Panggil Hitung_Jumlah_Objek

    Panggil Gambar_Bounding_Box

    Panggil Tampilkan_Informasi

    Tunggu 5 detik

ENDWHILE

END
```
