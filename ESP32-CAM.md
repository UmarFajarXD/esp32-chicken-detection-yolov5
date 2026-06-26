Pseudocode Dari ESP32-CAM

---

# Algoritma 1. Inisialisasi Kamera

```
ALGORITMA Inisialisasi_Kamera

BEGIN

Pilih konfigurasi pin kamera AI Thinker

Atur resolusi kamera menjadi 1024 × 768 piksel

Atur jumlah buffer kamera menjadi 2

Atur kualitas JPEG menjadi 80

Inisialisasi kamera

IF kamera berhasil diinisialisasi THEN

    Tampilkan "CAMERA OK"

ELSE

    Tampilkan "CAMERA FAIL"

ENDIF

END
```

---

# Algoritma 2. Inisialisasi Access Point WiFi

```
ALGORITMA Inisialisasi_Access_Point

BEGIN

Atur mode WiFi menjadi Access Point

Masukkan SSID

Masukkan Password

Aktifkan Access Point

Ambil alamat IP Access Point

END
```

---

# Algoritma 3. Inisialisasi Web Server

```
ALGORITMA Inisialisasi_Web_Server

BEGIN

Hubungkan URL "/images.h"

dengan fungsi Capture_Gambar

Jalankan Web Server

END
```

---

# Algoritma 4. Mengubah Ukuran Gambar (Resize Image)

```
ALGORITMA Resize_Gambar

BEGIN

FOR setiap baris tujuan DO

    FOR setiap kolom tujuan DO

        Hitung koordinat X sumber

        Hitung koordinat Y sumber

        Salin nilai piksel sumber

        ke piksel tujuan

    ENDFOR

ENDFOR

END
```

---

# Algoritma 5. Mengambil Gambar dari Kamera

```
ALGORITMA Capture_Gambar

BEGIN

Ambil gambar dari kamera

IF gambar gagal diperoleh THEN

    Kirim respon

    "Capture Failed"

    Selesai

ENDIF

Ambil data gambar JPEG

Ambil ukuran gambar

Ambil lebar gambar

Ambil tinggi gambar

END
```

---

# Algoritma 6. Alokasi Buffer RGB565

```
ALGORITMA Alokasi_Buffer

BEGIN

Hitung jumlah piksel gambar

Alokasikan memori

untuk buffer RGB565

END
```

---

# Algoritma 7. Konversi JPEG ke RGB565

```
ALGORITMA Konversi_JPEG_RGB565

BEGIN

Konversi gambar JPEG

menjadi RGB565

IF konversi berhasil THEN

    Lanjutkan proses

ELSE

    Kirim respon

    "Image Convert Failed"

    Bebaskan memori

ENDIF

END
```

---

# Algoritma 8. Resize Hasil Konversi

```
ALGORITMA Resize_Hasil_Konversi

BEGIN

Hitung ukuran gambar tujuan

Alokasikan memori

untuk gambar hasil resize

Panggil fungsi Resize_Gambar

Bebaskan buffer RGB565

END
```

---

# Algoritma 9. Mengubah Data Gambar menjadi String

```
ALGORITMA Konversi_Gambar_Ke_String

BEGIN

Inisialisasi variabel String

FOR setiap piksel gambar DO

    Ambil nilai RGB565

    Tambahkan ke String

    Pisahkan dengan tanda koma

ENDFOR

END
```

---

# Algoritma 10. Mengirim Data ke Client

```
ALGORITMA Kirim_Data_HTTP

BEGIN

Hitung panjang String

Atur Content Length

Kirim data gambar

melalui HTTP Response

Bebaskan memori gambar

END
```

---

# Algoritma 11. Inisialisasi Sistem

```
ALGORITMA Setup

BEGIN

Mulai komunikasi Serial

Panggil Inisialisasi_Access_Point

Panggil Inisialisasi_Kamera

Panggil Inisialisasi_Web_Server

END
```

---

# Algoritma 12. Program Utama

```
ALGORITMA Program_Utama

BEGIN

Panggil Setup

WHILE TRUE DO

    Dengarkan permintaan

    dari HTTP Client

    IF terdapat permintaan

    "/images.h" THEN

        Panggil Capture_Gambar

        Panggil Alokasi_Buffer

        Panggil Konversi_JPEG_RGB565

        Panggil Resize_Hasil_Konversi

        Panggil Konversi_Gambar_Ke_String

        Panggil Kirim_Data_HTTP

    ENDIF

ENDWHILE

END
```

---
