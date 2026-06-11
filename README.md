# Dokumentasi Pseudocode Program & Dataset

Repositori ini berisi dokumentasi alur logika (Pseudocode) untuk tahapan persiapan dataset/pelatihan model dan kode program utama tugas akhir deteksi objek.

---

## 1. Pseudocode Persiapan Dataset & Pelatihan Model (Google Colab)

Algoritma ini mendeskripsikan langkah-langkah pengumpulan data, anotasi, pelatihan model YOLOv5, hingga konversi model menjadi kode C++ agar dapat berjalan di ESP32-S3.

```text
ALGORITMA Persiapan_Dataset_Dan_Pelatihan_YOLOv5

DEKLARASI:
    Dataset_Mentah : Kumpulan gambar objek (ayam) hasil pemotretan
    Dataset_Anotasi : Gambar beserta koordinat bounding box kelas ayam (Format YOLO: class_id, cx, cy, w, h)
    Data_Train, Data_Val : Pembagian data latih (80%) dan data validasi (20%)
    Model_Pretrained = "yolov5n.pt"  // Bobot awal YOLOv5 Nano
    Model_Best : Hasil pelatihan terbaik (best.pt)
    Model_ONNX : Hasil ekspor model ke format ONNX
    Cpp_Files : File yolo5n.cpp, yolo5n.h, dan yolo5n_weights_*.h

PROSEDUR UTAMA()
    // 1. Persiapan & Preprocessing Dataset
    KUMPULKAN Dataset_Mentah dari lapangan / kamera
    LAKUKAN Anotasi koordinat objek menggunakan tools (misal: Roboflow / LabelImg)
    SIMPAN hasil anotasi ke dalam format anotasi YOLO (.txt)
    BAGI dataset menjadi Data_Train (80%) dan Data_Val (20%)
    UNGGAH dataset ke penyimpanan Google Drive
    
    // 2. Konfigurasi Lingkungan Google Colab
    AKTIFKAN runtime Google Colab berbasis GPU
    CLONE repositori resmi ultralytics/yolov5
    INSTALL dependency pendukung (PyTorch, OpenCV, dll.)
    BUAT file konfigurasi "custom_dataset.yaml" (berisi path Data_Train, Data_Val, jumlah kelas, dan nama kelas "ayam")
    
    // 3. Pelatihan Model (Training)
    JALANKAN perintah training YOLOv5:
        - Input ukuran gambar: 224 x 224 piksel
        - Batch size: 16
        - Epochs (Putaran): 50 (atau sesuai kebutuhan)
        - Bobot dasar: Model_Pretrained
    
    SIMPAN hasil pelatihan terbaik ke file Model_Best (best.pt)
    
    // 4. Ekspor dan Konversi ke C++ (C-Code Generation)
    JALANKAN script ekspor YOLOv5 untuk mengubah Model_Best (PyTorch) menjadi Model_ONNX
    JALANKAN program C-code generator untuk mengekstrak arsitektur jaringan dan nilai bobot (Weights) dari Model_ONNX
    
    HASILKAN file output Cpp_Files (yolo5n.cpp, yolo5n.h, yolo5n_weights_*.h)
    DOWNLOAD Cpp_Files untuk dimasukkan ke proyek firmware Arduino
ENDPROSEDUR
```

---

## 2. Pseudocode Program Utama Deteksi Objek (ESP32-S3 Firmware)

Algoritma ini menjelaskan alur kerja program pada mikrokontroler ESP32-S3 untuk menerima gambar, melakukan preprocessing, memprosesnya dengan C++ model YOLOv5n, dan mendeteksi objek.

```text
ALGORITMA Deteksi_Objek_ESP32_YOLOv5

DEKLARASI:
    picture_1 : ARRAY [250][250] OF INTEGER          // Menyimpan gambar mentah RGB565
    resized_image : ARRAY [224][224] OF INTEGER      // Menyimpan gambar skala 224x224
    normalized_image : ARRAY [3][224][224] OF FLOAT  // Matriks warna RGB skala [0.0 - 1.0]
    output_yolo : ARRAY [3087][85] OF FLOAT          // Matriks hasil output forward pass model
    detections : LIST OF Detection                    // Menyimpan daftar bounding box valid
    CONFIDENCE_THRESHOLD = 0.20                       // Batas keyakinan deteksi (20%)
    IOU_THRESHOLD = 0.45                              // Batas tumpang tindih box

PROSEDUR SETUP()
    INISIALISASI Serial Monitor, Layar TFT, dan Memori PSRAM
    INISIALISASI Jaringan WiFi
    
    WHILE WiFi tidak terhubung DO
        Tunggu 500 milidetik
    ENDWHILE
    
    IF WiFi terhubung THEN
        Kirim HTTP GET ke server lokal "http://192.168.4.1/images.h"
        
        IF Response HTTP Sukses THEN
            data_string = Ambil teks gambar dari HTTP response
            picture_1 = Konversi data_string menjadi ARRAY [250][250] (RGB565)
            
            // Preprocessing Gambar untuk Model YOLOv5n
            resized_image = Resize picture_1 dari ukuran 250x250 ke 224x224
            normalized_image = Konversi RGB565 ke RGB888 dan bagi nilai piksel dengan 255.0
            
            // Jalankan Inferensi Model YOLO (C++ Array)
            output_yolo = forward_pass(normalized_image)
            
            // Dekode Hasil Bounding Box
            FOR i = 0 TO 3086 DO
                confidence = output_yolo[i][4]
                
                IF confidence >= CONFIDENCE_THRESHOLD THEN
                    max_class_score = Nilai Terbesar dari probabilitas kelas output_yolo[i][5...84]
                    class_id = Indeks Kelas Terbesar
                    combined_score = max_class_score * confidence
                    
                    // Filter hanya mendeteksi kelas Ayam (Class ID 14)
                    IF class_id == 14 AND combined_score >= CONFIDENCE_THRESHOLD THEN
                        det.x = output_yolo[i][0] - (output_yolo[i][2] / 2.0)
                        det.y = output_yolo[i][1] - (output_yolo[i][3] / 2.0)
                        det.w = output_yolo[i][2]
                        det.h = output_yolo[i][3]
                        det.confidence = combined_score
                        det.class_id = class_id
                        
                        Tambahkan det ke LIST detections
                    ENDIF
                ENDIF
            ENDFOR
            
            // Gabungkan kotak pembatas yang mendeteksi objek yang sama
            detections = Non_Maximum_Suppression(detections, IOU_THRESHOLD)
            
            // Tampilkan Hasil Gambar & Bounding Box di Layar TFT
            Layar_TFT = Tampilkan picture_1 (skala 480x320)
            FOR EACH det IN detections DO
                Gambar kotak hijau di layar TFT sesuai koordinat det
                Tulis teks label "Ayam (nilai score)" di atas kotak tersebut
            ENDFOR
        ENDIF
    ENDIF
ENDPROSEDUR

PROSEDUR LOOP()
    // Prosedur kosong (Inference dijalankan sekali saat alat dinyalakan)
ENDPROSEDUR
```
