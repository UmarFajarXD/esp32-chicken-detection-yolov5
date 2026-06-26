Pseudocode Dataset

---

# **Catatan (Persiapan Software)**

> **Sebelum melakukan proses pelatihan dataset pada Visual Studio Code, pastikan menggunakan versi perangkat lunak berikut agar kompatibel dengan seluruh proses deployment hingga ESP32-S3:**

| Software           | Versi         |
| ------------------ | ------------- |
| Python             | **3.11.9**    |
| PyTorch            | **2.6.0**     |
| ONNX               | **1.17.0**    |
| Visual Studio Code | Versi terbaru |
| Git                | Versi terbaru |

### Install Python 3.11.9

Unduh dan instal Python 3.11.9, kemudian aktifkan opsi **Add Python to PATH** saat proses instalasi.

Verifikasi instalasi:

```
python --version
```

Output

```
Python 3.11.9
```

---

### Install PyTorch 2.6.0

```
pip install torch==2.6.0 torchvision torchaudio
```

Verifikasi

```
python -c "import torch; print(torch.__version__)"
```

Output

```
2.6.0
```

---

### Install ONNX 1.17.0

```
pip install onnx==1.17.0
```

Verifikasi

```
python -c "import onnx; print(onnx.__version__)"
```

Output

```
1.17.0
```

---

### Install seluruh library YOLOv5

```
pip install -r requirements.txt
```

---

# Persiapan Dataset

```
Mulai

Membuat folder dataset

Membuat folder train

Membuat folder train/images

Membuat folder train/labels

Membuat folder val

Membuat folder val/images

Membuat folder val/labels

Menyalin seluruh gambar training

Menyalin seluruh file label training

Menyalin seluruh gambar validasi

Menyalin seluruh file label validasi

Membuat file dataset.yaml

Menentukan lokasi dataset

Menentukan lokasi folder train

Menentukan lokasi folder validation

Menentukan jumlah kelas

Menentukan nama kelas

Menyimpan dataset.yaml

Selesai
```

---

# Membuka Visual Studio Code

```
Mulai

Menjalankan Visual Studio Code

Membuka folder proyek YOLOv5

Membuka Terminal

Memastikan Python 3.11.9 telah terpasang

Menjalankan perintah

python --version

Memastikan PyTorch 2.6.0 telah terpasang

Menjalankan perintah

python -c "import torch; print(torch.__version__)"

Memastikan ONNX 1.17.0 telah terpasang

Menjalankan perintah

python -c "import onnx; print(onnx.__version__)"

Memastikan library YOLOv5 telah terinstal

Menjalankan perintah

pip install -r requirements.txt

Selesai
```

---

# Training Dataset

```
Mulai

Membuka Terminal Visual Studio Code

Menjalankan perintah

python train.py --img 224 --batch 16 --epochs 100 --data dataset.yaml --weights yolov5n.pt

Membaca dataset

Membaca label

Melakukan augmentasi data

Melakukan proses training

Menghitung loss

Memperbarui bobot model

Mengulangi proses hingga seluruh epoch selesai

Menyimpan model terbaik

best.pt

Selesai
```

---

# Pengujian Model

```
Mulai

Membuka Terminal Visual Studio Code

Menjalankan perintah

python detect.py --weights runs/train/exp/weights/best.pt --img 224 --conf 0.25 --source path/to/test/image.jpg

Memuat model best.pt

Melakukan preprocessing gambar

Melakukan inferensi

Menghasilkan bounding box

Menampilkan confidence

Menyimpan hasil deteksi

Selesai
```

---

# Export Model ke ONNX

```
Mulai

Membuka Terminal Visual Studio Code

Menjalankan perintah

python export.py --weights runs/train/exp/weights/best.pt --include onnx --imgsz 224

Memuat model best.pt

Mengubah model menjadi format ONNX

Menyimpan file

best.onnx

Selesai
```

---

# Konversi Menggunakan Synapedge

```
Mulai

Membuka PowerShell

Menjalankan perintah

python "C:\Users\LENOVO\Downloads\TA - backup\synapedge\synapedge\synapedge.py" "runs/train/exp/weights/best.onnx" -o "C:\Users\LENOVO\Downloads\TA - backup\yolov5-ori\yolov5n.c"

Membaca file best.onnx

Melakukan parsing layer

Menghasilkan file C

Menyimpan

yolov5n.c

Menghasilkan file header

yolov5n.h

Selesai
```

---

# Mengubah File C menjadi C++

```
Mulai

Membuka folder hasil Synapedge

Mencari file

yolov5n.c

Mengubah nama file menjadi

yolov5n.cpp

Memindahkan file ke proyek Arduino

Selesai
```

---

# Optimasi Manual

```text
Mulai

Membuka file yolov5n.cpp

Menambahkan

#pragma GCC optimize ("O3")

Mencari fungsi

node_model_11_Resize

Mengubah

H_out = 14

Mengubah

W_out = 14

Mencari fungsi

node_model_15_Resize

Mengubah

H_out = 28

Mengubah

W_out = 28

Menyimpan perubahan

Selesai
```

---

# Validasi Model

```
Mulai

Membuka Terminal Visual Studio Code

Menjalankan perintah

python val.py --weights runs/train/exp/weights/best.pt --data dataset.yaml --img 224 --save-txt

Melakukan validasi dataset

Menghitung Precision

Menghitung Recall

Menghitung F1-Score

Menghitung mAP

Membuat Precision Curve

Membuat Recall Curve

Membuat PR Curve

Membuat F1 Curve

Membuat Confusion Matrix

Menyimpan seluruh hasil evaluasi

Selesai
```

---

# Implementasi pada ESP32-S3

```
Mulai

Membuka Arduino IDE

Menyalin file

yolov5n.cpp

Menyalin file

yolov5n.h

Melakukan kompilasi program

Mengunggah program ke ESP32-S3

Menjalankan inferensi

Menampilkan hasil deteksi objek

Selesai
```
