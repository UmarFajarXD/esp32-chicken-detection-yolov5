# Pseudocode untuk Esp32-S3-Object-Detection-CS.ino

Dokumen ini berisi representasi logika (pseudocode) dari program deteksi objek YOLOv5n pada ESP32-S3.

## 1. Definisi Struktur Data & Konstanta
```text
KONSTANTA I_HEIGHT = 250             // Tinggi citra asli dari server
KONSTANTA I_WIDTH = 250              // Lebar citra asli dari server
KONSTANTA DST_WIDTH = 224            // Lebar masukan model (YOLO input)
KONSTANTA DST_HEIGHT = 224           // Tinggi masukan model (YOLO input)
KONSTANTA NUM_BOXES = 3087           // Jumlah prediksi bounding box YOLO
KONSTANTA NUM_CLASSES = 85           // 80 kelas objek + 5 parameter bounding box

KONSTANTA CONFIDENCE_THRESHOLD = 0.5 // Batas minimal keyakinan deteksi
KONSTANTA IOU_THRESHOLD = 0.45       // Batas tumpang tindih untuk NMS (Non-Maximum Suppression)

KONSTANTA original_width = 480       // Resolusi lebar target layar TFT
KONSTANTA original_height = 320      // Resolusi tinggi target layar TFT

KONSTANTA WIFI_SSID = "ESP32-Access-Point"
KONSTANTA WIFI_PASS = "13572468"

STRUKTUR Detection:
    x: FLOAT           // Titik koordinat X kiri-atas box
    y: FLOAT           // Titik koordinat Y kiri-atas box
    w: FLOAT           // Lebar box
    h: FLOAT           // Tinggi box
    confidence: FLOAT  // Skor keyakinan objek
    class_scores: FLOAT// Skor akhir probabilitas kelas
    class_id: INTEGER  // Indeks kelas terdeteksi
AKHIR STRUKTUR

GLOBAL picture_1: ARRAY [I_HEIGHT][I_WIDTH] OF INTEGER (16-bit RGB565)
```

---

## 2. Fungsi Pembantu (Helper Functions)

### A. Perhitungan Tingkat Overlap Kotak (`compute_iou`)
```text
FUNGSI compute_iou(a: Detection, b: Detection) -> FLOAT:
    x_left = MAX(a.x, b.x)
    y_top = MAX(a.y, b.y)
    x_right = MIN(a.x + a.w, b.x + b.w)
    y_bottom = MIN(a.y + a.h, b.y + b.h)

    JIKA x_right < x_left ATAU y_bottom < y_top MAKA:
        KEMBALIKAN 0.0
    AKHIR JIKA

    intersection_area = (x_right - x_left) * (y_bottom - y_top)
    area_a = a.w * a.h
    area_b = b.w * b.h
    union_area = area_a + area_b - intersection_area

    KEMBALIKAN intersection_area / union_area
AKHIR FUNGSI
```

### B. Eliminasi Bounding Box Duplikat (`non_maximum_suppression`)
```text
FUNGSI non_maximum_suppression(VAR detections: ARRAY OF Detection, VAR det_count: INTEGER, iou_threshold: FLOAT):
    UNTUK i DARI 0 HINGGA det_count - 1 LAKUKAN:
        JIKA detections[i].confidence <= 0 MAKA:
            LANJUTKAN
        AKHIR JIKA
        UNTUK j DARI i + 1 HINGGA det_count - 1 LAKUKAN:
            JIKA detections[j].confidence <= 0 MAKA:
                LANJUTKAN
            AKHIR JIKA
            JIKA compute_iou(detections[i], detections[j]) > iou_threshold MAKA:
                detections[j].confidence = 0 // Batalkan deteksi dengan keyakinan lebih rendah
            AKHIR JIKA
        AKHIR UNTUK
    AKHIR UNTUK

    // Memadatkan array deteksi dari elemen yang dibatalkan
    new_count = 0
    UNTUK i DARI 0 HINGGA det_count - 1 LAKUKAN:
        JIKA detections[i].confidence > 0 MAKA:
            detections[new_count] = detections[i]
            new_count = new_count + 1
        AKHIR JIKA
    AKHIR UNTUK
    det_count = new_count
AKHIR FUNGSI
```

### C. Normalisasi Pixel Gambar (`normalizeImage`)
```text
FUNGSI normalizeImage(input: ARRAY [DST_HEIGHT][DST_WIDTH] OF INTEGER, VAR output: ARRAY [1][3][DST_HEIGHT][DST_WIDTH] OF FLOAT):
    UNTUK i DARI 0 HINGGA DST_WIDTH - 1 LAKUKAN:
        UNTUK j DARI 0 HINGGA DST_HEIGHT - 1 LAKUKAN:
            pixel = input[i][j]
            // Ekstrak bit RGB565 (16-bit)
            r = (pixel >> 11) DAN 0x1F
            g = (pixel >> 5) DAN 0x3F
            b = pixel DAN 0x1F

            // Konversi menjadi RGB888 (8-bit per channel)
            r8 = (r KIRI_SHIFT 3) ATAU (r KANAN_SHIFT 2)
            g8 = (g KIRI_SHIFT 2) ATAU (g KANAN_SHIFT 4)
            b8 = (b KIRI_SHIFT 3) ATAU (b KANAN_SHIFT 2)

            // Simpan ke bentuk float ternormalisasi [0.0, 1.0]
            output[0][0][i][j] = r8 / 255.0
            output[0][1][i][j] = g8 / 255.0
            output[0][2][i][j] = b8 / 255.0
        AKHIR UNTUK
    AKHIR UNTUK
AKHIR FUNGSI
```

### D. Skala Citra (`resizeImage` & `resizeImage_up`)
```text
// Mengubah citra ke ukuran input model (224x224)
FUNGSI resizeImage(src: POINTER, VAR dst: POINTER):
    UNTUK y DARI 0 HINGGA DST_HEIGHT - 1 LAKUKAN:
        UNTUK x DARI 0 HINGGA DST_WIDTH - 1 LAKUKAN:
            srcX = (x * I_WIDTH) / DST_WIDTH
            srcY = (y * I_HEIGHT) / DST_HEIGHT
            dst[y * DST_WIDTH + x] = src[srcY * I_WIDTH + srcX]
        AKHIR UNTUK
    AKHIR UNTUK
AKHIR FUNGSI

// Mengubah citra ke resolusi layar TFT (480x320)
FUNGSI resizeImage_up(src: POINTER, VAR dst: POINTER):
    UNTUK y DARI 0 HINGGA original_height - 1 LAKUKAN:
        UNTUK x DARI 0 HINGGA original_width - 1 LAKUKAN:
            srcX = (x * I_WIDTH) / original_width
            srcY = (y * I_HEIGHT) / original_height
            dst[y * original_width + x] = src[srcY * I_WIDTH + srcX]
        AKHIR UNTUK
    AKHIR UNTUK
AKHIR FUNGSI
```

### E. Penggambaran Citra ke Layar (`drawImage`)
```text
FUNGSI drawImage(x_offset: INTEGER, y_offset: INTEGER, imageArray: POINTER, width: INTEGER, height: INTEGER):
    UNTUK y DARI 0 HINGGA height - 1 LAKUKAN:
        UNTUK x DARI 0 HINGGA width - 1 LAKUKAN:
            index = y * width + x
            TAMPILKAN_PIXEL_TFT(x + x_offset, y + y_offset, imageArray[index])
        AKHIR UNTUK
    AKHIR UNTUK
AKHIR FUNGSI
```

### F. Parsing & Filter Hasil Deteksi YOLOv5 (`parse_yolo_output`)
```text
FUNGSI parse_yolo_output(output: ARRAY [NUM_BOXES][NUM_CLASSES] OF FLOAT, VAR detections: ARRAY OF Detection, VAR det_count: INTEGER):
    det_count = 0
    scale_x = original_width / DST_WIDTH
    scale_y = original_height / DST_HEIGHT

    UNTUK i DARI 0 HINGGA NUM_BOXES - 1 LAKUKAN:
        confidence = output[i][4]
        JIKA confidence < CONFIDENCE_THRESHOLD MAKA:
            LANJUTKAN
        AKHIR JIKA

        // Cari skor probabilitas kelas tertinggi
        max_class_score = -INFINITY
        class_id = -1
        UNTUK j DARI 5 HINGGA NUM_CLASSES - 1 LAKUKAN:
            JIKA output[i][j] > max_class_score MAKA:
                max_class_score = output[i][j]
                class_id = j - 5
            AKHIR JIKA
        AKHIR UNTUK

        // Filter kelas objek: bird (14), dog (18), giraffe (23), teddy bear (77)
        JIKA class_id != 14 DAN class_id != 18 DAN class_id != 23 DAN class_id != 77 MAKA:
            LANJUTKAN
        AKHIR JIKA

        combined_score = max_class_score * confidence
        JIKA combined_score < CONFIDENCE_THRESHOLD MAKA:
            LANJUTKAN
        AKHIR JIKA

        // Ambil center_x, center_y, width, height dari output model
        cx = output[i][0]
        cy = output[i][1]
        w = output[i][2]
        h = output[i][3]

        // Konversi koordinat model ke koordinat layar LCD
        x_min = INT((cx - w / 2.0) * scale_x)
        y_min = INT((cy - h / 2.0) * scale_y)
        x_max = INT((cx + w / 2.0) * scale_x)
        y_max = INT((cy + h / 2.0) * scale_y)

        // Batasi koordinat agar berada dalam rentang layar
        x_min = CLAMP(x_min, 0, original_width - 1)
        y_min = CLAMP(y_min, 0, original_height - 1)
        x_max = CLAMP(x_max, 0, original_width - 1)
        y_max = CLAMP(y_max, 0, original_height - 1)

        // Simpan hasil deteksi
        detections[det_count].x = x_min
        detections[det_count].y = y_min
        detections[det_count].w = x_max - x_min
        detections[det_count].h = y_max - y_min
        detections[det_count].confidence = combined_score
        detections[det_count].class_id = class_id
        det_count = det_count + 1
    AKHIR UNTUK

    // Lakukan eliminasi tumpang tindih menggunakan NMS
    non_maximum_suppression(detections, det_count, IOU_THRESHOLD)
AKHIR FUNGSI
```

---

## 3. Alur Utama Jalannya Program

### Prosedur Setup (`setup`)
```text
PROSEDUR setup():
    MULAI_SERIAL(115200)
    INISIALISASI_LAYAR_TFT()
    SET_ROTASI_TFT(3)
    FILL_LAYAR_TFT(WARNA_PUTIH)

    // Deteksi memori tambahan PSRAM
    JIKA INISIALISASI_PSRAM() BERHASIL MAKA:
        CETAK_SERIAL("PSRAM terdeteksi dan diinisialisasi.")
    LAIN HAL:
        CETAK_SERIAL("PSRAM gagal diinisialisasi!")
    AKHIR JIKA

    // Hubungkan WiFi
    MULAI_WIFI(WIFI_SSID, WIFI_PASS)
    SELAMA WIFI_BELUM_TERHUBUNG LAKUKAN:
        TUNGGU(500 ms)
    AKHIR SELAMA

    JIKA WIFI_TERHUBUNG MAKA:
        HTTP_CLIENT http
        http.MULAI("http://192.168.4.1/images.h")
        httpCode = http.KIRIM_PERMINTAAN_GET()
        CETAK_SERIAL("HTTP Response Code: ", httpCode)

        JIKA httpCode > 0 MAKA:
            StringGambar = http.DAPATKAN_RESPON_STRING() // Ambil file string CSV piksel gambar
            
            // Menguraikan CSV string ke array picture_1 [250][250]
            i = 0, j = 0, k = 0
            SELAMA k < PANJANG(StringGambar) LAKUKAN:
                commaIndex = INDEX_OF(',', k)
                JIKA commaIndex == -1 MAKA:
                    picture_1[i][j] = TO_INT(SUBSTRING(k))
                    BERHENTI_SELAMA
                LAIN HAL:
                    picture_1[i][j] = TO_INT(SUBSTRING(k, commaIndex))
                AKHIR JIKA

                k = commaIndex + 1
                j = j + 1
                JIKA j > (I_WIDTH - 1) MAKA:
                    i = i + 1
                    j = 0
                AKHIR JIKA
            AKHIR SELAMA

            /* ALOKASI MEMORI DI PSRAM */
            SrcImage = ALOKASI_PSRAM(original_width * original_height * 2)
            resizedImage = ALOKASI_PSRAM(DST_WIDTH * DST_HEIGHT * 2)
            normalized = ALOKASI_PSRAM(sizeof(float) * DST_WIDTH * DST_HEIGHT * 3)
            output = ALOKASI_PSRAM(sizeof(float) * NUM_BOXES * NUM_CLASSES)
            detections = ALOKASI_PSRAM(sizeof(Detection) * 600)

            /* 1. Tampilkan Gambar di Layar TFT */
            resizeImage_up(picture_1, SrcImage)
            drawImage(0, 0, SrcImage, original_width, original_height)
            BEBASKAN_PSRAM(SrcImage)

            /* 2. Persiapan Masukan Model */
            resizeImage(picture_1, resizedImage)
            normalizeImage(resizedImage, normalized)

            /* 3. Menjalankan Forward Pass YOLOv5 */
            WAKTU_MULAI = AMBIL_MIKRODETIK()
            forward_pass(normalized, output) // Inferensi Neural Network
            WAKTU_SELESAI = AMBIL_MIKRODETIK()

            BEBASKAN_PSRAM(normalized)
            BEBASKAN_PSRAM(resizedImage)

            /* 4. Parsing Hasil Deteksi & NMS */
            det_count = 0
            parse_yolo_output(output, detections, det_count)

            DURASI = (WAKTU_SELESAI - WAKTU_MULAI) / 1000000.0
            CETAK_SERIAL("Waktu Inferensi: ", DURASI, " detik")

            /* 5. Menggambar Kotak Merah Deteksi */
            UNTUK i DARI 0 HINGGA det_count - 1 LAKUKAN:
                CETAK_INFO_DETEKSI_DI_SERIAL(detections[i])
                // Gambar garis tebal 2 piksel
                GAMBAR_KOTAK_DI_LAYAR(detections[i].x, detections[i].y, detections[i].w, detections[i].h, MERAH)
                GAMBAR_KOTAK_DI_LAYAR(detections[i].x - 1, detections[i].y - 1, detections[i].w + 2, detections[i].h + 2, MERAH)
            AKHIR UNTUK

            BEBASKAN_PSRAM(detections)
            BEBASKAN_PSRAM(output)
        AKHIR JIKA
        http.SELESAI()
    AKHIR JIKA
AKHIR PROSEDUR
```

### Prosedur Loop Utama (`loop`)
```text
PROSEDUR loop():
    // Kosong, karena program didesain untuk berjalan sekali pada setup() saat startup.
AKHIR PROSEDUR
```
