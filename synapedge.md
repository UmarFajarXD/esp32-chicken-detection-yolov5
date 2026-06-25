pseudocode SynapEdge untuk proses pembagian bobot (weights) model menjadi beberapa file header (.h)

ALGORITMA PEMBAGIAN BOBOT MODEL SYNAPEDGE

Input:
    weights_data      ← seluruh bobot model yang telah dikuantisasi
    MAX_CHUNK_SIZE    ← ukuran maksimum setiap file
                         (default = 5 MB)

Proses:

1. Hitung ukuran total bobot model
      total_size ← ukuran(weights_data)

2. Tentukan jumlah file yang dibutuhkan
      num_chunks ← ceil(total_size / MAX_CHUNK_SIZE)

3. Untuk setiap bagian bobot
      FOR i ← 0 TO num_chunks - 1 DO

          start_index ← i × MAX_CHUNK_SIZE

          end_index ← MIN(
                            start_index + MAX_CHUNK_SIZE,
                            total_size
                         )

          chunk_data ← weights_data[start_index : end_index]

          IF ukuran(chunk_data) > 0 THEN

              buat file:
                  model_weights_i.h

              tulis:
                  const uint8_t weights_i[] = {
                      chunk_data
                  };

              ENDIF

          ENDFOR

4. Simpan metadata model
      total_weight_files ← num_chunks

Output:
      model_weights_0.h
      model_weights_1.h
      ...
      model_weights_(num_chunks-1).h

### Pseudocode Perhitungan Jumlah File

Input:
    TOTAL_MODEL_SIZE = 26 MB
    MAX_CHUNK_SIZE   = 5 MB

num_chunks ← ceil(26 / 5)

num_chunks ← ceil(5.2)

num_chunks ← 6

Output:
    weights_0.h
    weights_1.h
    weights_2.h
    weights_3.h
    weights_4.h
    weights_5.h

### Jika Diubah Menjadi 9 File

Input:
    TOTAL_MODEL_SIZE = 26 MB
    MAX_CHUNK_SIZE   = 3 MB

num_chunks ← ceil(26 / 3)

num_chunks ← ceil(8.67)

num_chunks ← 9

Output:
    weights_0.h
    weights_1.h
    weights_2.h
    weights_3.h
    weights_4.h
    weights_5.h
    weights_6.h
    weights_7.h
    weights_8.h

### Pseudocode dengan Perbaikan Bug Off-by-One

FOR i ← 0 TO num_chunks - 1 DO

    chunk_data ← ambil_data_bobot(i)

    IF ukuran(chunk_data) = 0 THEN
         SKIP
    ELSE
         simpan_file_weights(i)
    ENDIF

    ENDFOR
