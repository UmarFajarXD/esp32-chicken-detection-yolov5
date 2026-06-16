# Workflow & Pseudocode untuk Dataset Custom "Ayam" (Roboflow & Synapedge)

Dokumen ini menjelaskan alur kerja (workflow) lengkap untuk melatih dataset custom ayam Anda dari Roboflow di Google Colab, mengekspornya ke format C++ menggunakan tool **Synapedge**, serta konfigurasi program pada ESP32-S3.

---

## BAGIAN 1: Workflow Konfigurasi & Training di Google Colab
Berikut adalah pseudocode/langkah-langkah eksekusi yang dijalankan pada Google Colab berdasarkan link notebook yang Anda berikan:

```text
PROSEDUR training_dan_ekspor_c():
    // 1. Setup Lingkungan YOLOv5
    CLONE_REPOSITORY("https://github.com/ultralytics/yolov5")
    PINDAH_DIREKTORI("yolov5")
    INSTALL_DEPENDENCIES("requirements.txt")
    INSTALL_LIBRARY("onnx==1.17.0")

    // 2. Download Dataset dari Roboflow (Ayam)
    // Format dataset yang diunduh harus berupa "YOLOv5 PyTorch"
    DOWNLOAD_DATASET_FROM_ROBOFLOW(API_KEY="YOUR_ROBOFLOW_API_KEY", FORMAT="yolov5")
    // Ini menghasilkan folder dataset (train, val) dan file 'data.yaml'

    // 3. Training Model YOLOv5n
    // Menggunakan resolusi citra 224x224 piksel agar ringan bagi ESP32-S3
    JALANKAN_TRAINING(
        model = "yolov5n.pt",
        data = "data.yaml",
        epochs = 100,
        img_size = 224,
        batch_size = 16
    )
    // Hasil training disimpan sebagai file bobot terbaik: 'runs/train/exp/weights/best.pt'

    // 4. Ekspor Model ke Format ONNX
    JALANKAN_EKSPOR(
        weights = "runs/train/exp/weights/best.pt",
        img_size = 224,
        batch_size = 1,
        format = "onnx"
    )
    // Hasil ekspor disimpan sebagai file model: 'runs/train/exp/weights/best.onnx'

    // 5. Kompilasi ONNX ke Kode C++ via Synapedge
    CLONE_REPOSITORY("https://github.com/asad-shafi/synapedge.git")
    JALANKAN_SYNAPEDGE_COMPILER(
        input = "runs/train/exp/weights/best.onnx",
        output = "yolov5n.c"
    )
    // Ini menghasilkan file:
    // - yolov5n.c (Struktur layer model) -> Nanti diubah namanya menjadi 'yolov5n.cpp'
    // - yolov5n.h (Header definisi tensor)
    // - yolov5n_weights_0.h s/d yolov5n_weights_8.h (Pecahan bobot model)
AKHIR PROSEDUR
```

---

## BAGIAN 2: Konfigurasi Kode Arduino (ESP32-S3) untuk Dataset Ayam
Ketika Anda berpindah dari dataset COCO (80 kelas) ke dataset custom Ayam (1 kelas), **dimensi tensor keluaran model berubah**.
* **COCO Dataset:** 85 kolom (`[x, y, w, h, box_confidence, class_0_score, ..., class_79_score]`)
* **Dataset Ayam:** 6 kolom (`[x, y, w, h, box_confidence, ayam_score]`)

Oleh karena itu, Anda harus mengubah konstanta `NUM_CLASSES` dari **`85` menjadi `6`**.

### Pseudocode Program Arduino (Modifikasi untuk Ayam)

```text
// KONSTANTA & STRUKTUR DATA
KONSTANTA I_HEIGHT = 250
KONSTANTA I_WIDTH = 250
KONSTANTA DST_WIDTH = 224
KONSTANTA DST_HEIGHT = 224
KONSTANTA NUM_BOXES = 3087
KONSTANTA NUM_CLASSES = 6            // Berubah dari 85 menjadi 6 (5 parameter box + 1 kelas "ayam")

KONSTANTA CONFIDENCE_THRESHOLD = 0.5
KONSTANTA IOU_THRESHOLD = 0.45

KONSTANTA original_width = 480
KONSTANTA original_height = 320

STRUKTUR Detection:
    x: FLOAT
    y: FLOAT
    w: FLOAT
    h: FLOAT
    confidence: FLOAT
    class_scores: FLOAT
    class_id: INTEGER
AKHIR STRUKTUR
```

### Fungsi Utama Dekode Output (`parse_yolo_output` untuk Ayam)
Fungsi ini disesuaikan untuk membaca probabilitas kelas ayam pada indeks ke-5 dari output model.

```text
FUNGSI parse_yolo_output(output: ARRAY [NUM_BOXES][NUM_CLASSES] OF FLOAT, VAR detections: ARRAY OF Detection, VAR det_count: INTEGER):
    det_count = 0
    scale_x = original_width / DST_WIDTH
    scale_y = original_height / DST_HEIGHT

    UNTUK i DARI 0 HINGGA NUM_BOXES - 1 LAKUKAN:
        confidence = output[i][4] // Objectness confidence
        JIKA confidence < CONFIDENCE_THRESHOLD MAKA:
            LANJUTKAN
        AKHIR JIKA

        // Cari skor kelas ayam pada kolom indeks 5
        ayam_score = output[i][5]
        class_id = 0 // Kelas 0 merepresentasikan "ayam" pada dataset Anda

        combined_score = ayam_score * confidence
        JIKA combined_score < CONFIDENCE_THRESHOLD MAKA:
            LANJUTKAN
        AKHIR JIKA

        // Ambil parameter kotak pembatas model
        cx = output[i][0]
        cy = output[i][1]
        w = output[i][2]
        h = output[i][3]

        // Konversi koordinat model (224x224) ke koordinat layar LCD (480x320)
        x_min = INT((cx - w / 2.0) * scale_x)
        y_min = INT((cy - h / 2.0) * scale_y)
        x_max = INT((cx + w / 2.0) * scale_x)
        y_max = INT((cy + h / 2.0) * scale_y)

        // Batasi (clamp) agar tidak melewati tepi layar
        x_min = CLAMP(x_min, 0, original_width - 1)
        y_min = CLAMP(y_min, 0, original_height - 1)
        x_max = CLAMP(x_max, 0, original_width - 1)
        y_max = CLAMP(y_max, 0, original_height - 1)

        // Simpan hasil deteksi ke dalam array
        detections[det_count].x = x_min
        detections[det_count].y = y_min
        detections[det_count].w = x_max - x_min
        detections[det_count].h = y_max - y_min
        detections[det_count].confidence = combined_score
        detections[det_count].class_id = class_id
        det_count = det_count + 1
    AKHIR UNTUK

    // Hapus bounding box ganda/tumpang tindih
    non_maximum_suppression(detections, det_count, IOU_THRESHOLD)
AKHIR FUNGSI
```

---

## BAGIAN 3: Langkah Integrasi ke Arduino IDE
Setelah mendapatkan file `.c`, `.h` dan `.h` weights dari Colab 2, lakukan konfigurasi berikut di folder sketch Arduino Anda:

1. **Ganti Nama File C:**
   Ubah file `yolov5n.c` hasil ekspor menjadi `yolov5n.cpp` (agar dapat dicompile menggunakan g++ compiler pada Arduino IDE).

2. **Gunakan Memori PSRAM:**
   Karena ESP32-S3 memiliki SRAM internal terbatas, pastikan alokasi tensor ditaruh di PSRAM.
   * Pada file `yolov5n.h`, beri tanda komentar (`//`) pada instansiasi union statik:
     ```cpp
     // static union tensor_union_0 tu0;
     // static union tensor_union_1 tu1;
     // ...dst
     ```
   * Pada file `yolov5n.cpp`, tambahkan library `#include "esp32-hal-psram.h"` dan alokasikan pointer union secara dinamis menggunakan `ps_malloc` di dalam fungsi `forward_pass()`:
     ```cpp
     void forward_pass(const float images[1][3][224][224], float output0[1][3087][6]) {
         union tensor_union_0 *tu0 = (union tensor_union_0 *)ps_malloc(sizeof(union tensor_union_0));
         union tensor_union_1 *tu1 = (union tensor_union_1 *)ps_malloc(sizeof(union tensor_union_1));
         
         // ... proses kalkulasi layer menggunakan pointer (*tu0, *tu1) ...

         free(tu0);
         free(tu1);
     }
     ```

3. **Ganti Ukuran Alokasi di Setup:**
   Di file utama Arduino sketch (`.ino`), ganti alokasi tensor output agar disesuaikan dengan dimensi dataset ayam Anda:
   ```cpp
   float(*output)[NUM_BOXES][NUM_CLASSES] = (float(*)[NUM_BOXES][NUM_CLASSES])ps_malloc(sizeof(float) * (NUM_BOXES * NUM_CLASSES));
   ```
   Karena `NUM_CLASSES` sudah diset `6`, alokasi memori ini sekarang menjadi lebih hemat (hanya sebesar $3087 \times 6 \times 4$ byte = **$74.088$ byte**, dibanding saat COCO dataset yang memerlukan $3087 \times 85 \times 4$ byte = **$1.049.580$ byte**).
