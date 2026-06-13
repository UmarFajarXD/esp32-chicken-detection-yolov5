Pseudocode Program ESP32-CAM (Esp32-Camera-CS)

1. Deklarasi Konstanta & Variabel Global
text

DEKLARASI KONSTANTA:
    DST_HEIGHT = 250                       // Tinggi gambar tujuan (250 piksel)
    DST_WIDTH = 250                        // Lebar gambar tujuan (250 piksel)
    WIFI_SSID = "ESP32-Access-Point"       // SSID Access Point ESP32
    WIFI_PASS = "13572468"                 // Password Access Point ESP32
    URL = "/images.h"                      // Endpoint HTTP untuk pengambilan gambar
DEKLARASI VARIABEL GLOBAL:
    SRC_WIDTH: Integer                     // Lebar gambar asal (dinamis)
    SRC_HEIGHT: Integer                    // Tinggi gambar asal (dinamis)
    RES = Resolusi Kamera (1024 x 768)     // Resolusi kamera yang ditargetkan
    server = WebServer pada Port 80       // Inisialisasi web server
2. Prosedur Inisialisasi & Loop Utama
A. Prosedur setup()
Prosedur utama yang dijalankan sekali saat ESP32 pertama kali dinyalakan.

text

PROSEDUR setup()
    Inisialisasi Serial Komunikasi dengan Baud Rate 115200
    PANGGIL initWifi()
    PANGGIL initCamera()
    PANGGIL initServer()
END PROSEDUR
B. Prosedur initWifi()
Mengatur ESP32 agar bertindak sebagai Soft Access Point (AP) sehingga perangkat lain dapat terhubung langsung ke ESP32.

text

PROSEDUR initWifi()
    Tampilkan pesan "Setting AP..." ke Serial Monitor
    Aktifkan Soft Access Point dengan WIFI_SSID dan WIFI_PASS
    Dapatkan IP Address dari Soft AP
    Tampilkan IP Address ke Serial Monitor
END PROSEDUR
C. Prosedur initCamera()
Mengatur konfigurasi hardware kamera ESP32-CAM (Ai-Thinker).

text

PROSEDUR initCamera()
    BUAT objek konfigurasi kamera (cfg)
    Atur Pinout kamera ke AiThinker
    Atur Resolusi kamera ke RES (1024 x 768)
    Atur Buffer Count = 2 (double buffering)
    Atur Kualitas JPEG = 80
    
    Inisialisasi kamera dengan konfigurasi cfg (ok = Camera.begin(cfg))
    JIKA ok bernilai TRUE MAKA
        Tampilkan "CAMERA OK" ke Serial Monitor
    ELSE
        Tampilkan "CAMERA FAIL" ke Serial Monitor
    END JIKA
END PROSEDUR
D. Prosedur initServer()
Menghubungkan endpoint URL dengan fungsi handler dan menjalankan server.

text

PROSEDUR initServer()
    Daftarkan Route: JIKA ada HTTP Request GET ke URL ("/images.h") MAKA jalankan captureJpg()
    Mulai Web Server (server.begin())
END PROSEDUR
E. Prosedur loop()
Fungsi loop utama Arduino yang berjalan terus-menerus untuk melayani client.

text

PROSEDUR loop()
    Jalankan server.handleClient() untuk menangani HTTP request yang masuk
END PROSEDUR
3. Pemrosesan Citra & Handler HTTP
A. Prosedur captureJpg()
Fungsi utama penanganan HTTP Request /images.h yang mengambil gambar, mengubah format, meresize, dan mengirimkan data piksel ke klien.

text

PROSEDUR captureJpg()
    Catat waktu mulai: startTime = micros()
    
    // 1. Mengambil Gambar dari Kamera
    Ambil frame gambar dari kamera (frame = esp32cam::capture())
    
    JIKA frame kosong (nullptr) MAKA
        Tampilkan "CAPTURE FAILED!" ke Serial Monitor
        Kirim Respon HTTP 503 (Service Unavailable) ke Klien
        KELUAR dari Prosedur
    END JIKA
    
    Tampilkan informasi dimensi dan ukuran file JPG ke Serial Monitor
    
    Dapatkan pointer data JPEG (imageArray = frame->data())
    Dapatkan ukuran data JPEG (imageSize = frame->size())
    
    // 2. Alokasi Buffer untuk RGB565
    Hitung panjang buffer RGB565 (rgb565_len = frame->getWidth() * frame->getHeight())
    Alokasikan memori di PROGMEM untuk buffer RGB565 (rgb565_buf) sebesar rgb565_len * 2 bytes
    
    // 3. Konversi JPEG ke RGB565
    Konversi JPG ke RGB565 menggunakan fungsi jpg2rgb565 (success = jpg2rgb565(...))
    
    JIKA konversi sukses MAKA
        Tampilkan "rgb565 passed" ke Serial Monitor
        Atur SRC_WIDTH = frame->getWidth()
        Atur SRC_HEIGHT = frame->getHeight()
        
        // 4. Alokasi Buffer Gambar Hasil Resize (250x250)
        Hitung panjang buffer resize (I_LEN = DST_WIDTH * DST_HEIGHT) // 250 * 250 = 62500
        Alokasikan memori di PROGMEM untuk buffer hasil (dstImg) sebesar I_LEN * 2 bytes
        
        // 5. Lakukan Resize Gambar menggunakan Nearest-Neighbor
        PANGGIL resizeImage(rgb565_buf, dstImg)
        Tampilkan "resize passed" ke Serial Monitor
        
        Bebaskan memori rgb565_buf (free)
        
        // 6. Konversi Buffer Piksel ke Format String Comma-Separated
        Inisialisasi string kosong (hContent = "")
        UNTUK y dari 0 hingga DST_HEIGHT - 1 LAKUKAN
            UNTUK x dari 0 hingga DST_WIDTH - 1 LAKUKAN
                Hitung index satu dimensi (z = y * DST_WIDTH + x)
                JIKA z < (I_LEN - 1) MAKA
                    Tambahkan nilai piksel dstImg[z] diikuti tanda koma (",") ke hContent
                ELSE
                    Tambahkan nilai piksel terakhir dstImg[z] ke hContent (tanpa koma)
                END JIKA
            END UNTUK
        END UNTUK
        
        // 7. Kirim Data Ke Klien
        Atur HTTP Content Length sesuai panjang hContent
        Kirim Respon HTTP 200 dengan Tipe Konten "text/h" berisi hContent
        Tampilkan "Capture Send" ke Serial Monitor
        
        Bebaskan memori dstImg (free)
        
    ELSE // JIKA konversi JPEG ke RGB565 gagal
        Tampilkan "CONVERT TO RG565 FAILED!" ke Serial Monitor
        Kirim Respon HTTP 503 ke Klien
        Bebaskan memori rgb565_buf (free)
    END JIKA
    
    // 8. Hitung Durasi Eksekusi
    Catat waktu selesai: endTime = micros()
    Hitung durasi_s = (endTime - startTime) / 1000000.0
    Tampilkan durasi_s dalam detik ke Serial Monitor
END PROSEDUR
B. Prosedur resizeImage(src, dst)
Menerapkan algoritma penataan ulang piksel tetangga terdekat (Nearest-Neighbor Interpolation) untuk mengubah ukuran gambar asli menjadi $250 \times 250$.

text

PROSEDUR resizeImage(ARRAY OF uint16_t src, ARRAY OF uint16_t dst)
    UNTUK y dari 0 hingga DST_HEIGHT - 1 LAKUKAN
        UNTUK x dari 0 hingga DST_WIDTH - 1 LAKUKAN
            // Skala koordinat x dan y ke dimensi asli
            srcX = (x * SRC_WIDTH) / DST_WIDTH
            srcY = (y * SRC_HEIGHT) / DST_HEIGHT
            
            // Salin nilai piksel dari koordinat asal ke koordinat tujuan
            dst[y * DST_WIDTH + x] = src[srcY * SRC_WIDTH + srcX]
        END UNTUK
    END UNTUK
END PROSEDUR
