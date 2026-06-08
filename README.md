# Pemetaan Mangrove Menggunakan Random Forest

**Oleh:** Defani Arman Alfitriansyah

Notebook ini membangun pipeline klasifikasi tutupan lahan mangrove secara otomatis menggunakan **Google Earth Engine (GEE)** dan algoritma **Random Forest**. Data yang digunakan adalah citra **Sentinel-2 SR Harmonized** dengan periode 2024–2025, dilengkapi cloud masking berbasis Cloud Score+. Model dilatih menggunakan 15 fitur prediktor — 10 band spektral dan 5 indeks turunan — untuk membedakan tiga kelas tutupan: Badan Air, Mangrove, dan Non-Mangrove. Area kajian mencakup kawasan pesisir di sekitar koordinat 7.12–7.23°S, 113.05–113.18°E.

---

## Daftar Isi

1. [Alur Kerja](#alur-kerja)
2. [Preprocessing & Komposit](#preprocessing-dan-komposit)
3. [False Color Composite](#false-color-composite)
4. [Indeks Spektral](#indeks-spektral)
5. [Profil Spektral (Spectral Signature)](#profil-spektral)
6. [Korelasi Band & Indeks terhadap Kelas](#korelasi-band-dan-indeks)
7. [Hasil Klasifikasi Random Forest](#hasil-klasifikasi)
8. [Overlay Mangrove di Atas NIR](#overlay-mangrove)
9. [Evaluasi Akurasi](#evaluasi-akurasi)

---

## Alur Kerja

```
Sentinel-2 SR Harmonized (2024–2025)
            |
    Cloud Masking (cs_cdf >= 0.60)
            |
    Komposit Median + Clip Area Kajian
            |
    Hitung Indeks: NDVI, NDWI, NDMI, NDBI, IRECI
            |
    Ekstraksi Nilai ke Titik Sampel
            |
    Training Random Forest (500 pohon, split 70:30)
            |
    Klasifikasi Citra --> 3 Kelas
            |
    Evaluasi: Confusion Matrix, OA, Kappa
```

---

## Preprocessing dan Komposit

Data Sentinel-2 difilter berdasarkan batas kajian, periode waktu, dan persentase tutupan awan di bawah 20%. Cloud masking dilakukan menggunakan Cloud Score+ dengan threshold `cs_cdf >= 0.60`. Seluruh scene yang lolos filter digabung menggunakan operasi **median** untuk menghasilkan satu citra komposit bebas awan, lalu dinormalisasi ke rentang reflektansi 0–1 (dibagi 10.000).

---

## False Color Composite

Komposit warna semu menggunakan kombinasi band **B8 (NIR) – B11 (SWIR-1) – B4 (Red)**. Kombinasi ini efektif untuk membedakan vegetasi (tampak hijau terang), badan air (biru-hitam gelap), dan lahan terbangun atau lahan terbuka (kuning-cokelat).

![False Color Composite B8-11-4](download__43_.png)

Pada citra di atas, area hijau terang di sepanjang aliran sungai dan pesisir menandakan keberadaan vegetasi mangrove yang memiliki reflektansi NIR tinggi. Area ungu-hitam gelap merupakan tambak dan badan air, sedangkan area kuning adalah lahan pertanian dan permukiman.

---

## Indeks Spektral

Lima indeks turunan dihitung dari kombinasi band Sentinel-2 dan digunakan sebagai fitur tambahan dalam model klasifikasi.

| Indeks | Rumus | Fungsi |
|--------|-------|--------|
| NDVI | (B8 - B4) / (B8 + B4) | Kerapatan vegetasi |
| NDWI | (B3 - B8) / (B3 + B8) | Identifikasi badan air |
| NDMI | (B8 - B11) / (B8 + B11) | Kelembapan kanopi |
| NDBI | (B11 - B8) / (B11 + B8) | Kawasan terbangun |
| IRECI | (B8 - B4) / (B5 / B6) | Kandungan klorofil (Red-Edge) |

---

## Profil Spektral

Spectral signature menampilkan rata-rata nilai reflektansi tiap kelas pada seluruh band Sentinel-2. Pola ini digunakan untuk memverifikasi separabilitas antar kelas sebelum klasifikasi.

![Profil Spektral (Spectral Signature)](download__48_.png)

Beberapa pola penting yang terlihat:

- **Mangrove** (hijau tua) menunjukkan nilai NIR (B6–B8A) yang sangat tinggi, merupakan ciri khas vegetasi lebat dengan biomassa tinggi. Terdapat penurunan tajam di B11 (SWIR) akibat penyerapan air pada daun.
- **Non-Mangrove** (kuning) memiliki pola mirip mangrove di NIR namun dengan nilai SWIR yang lebih tinggi, mencerminkan vegetasi yang lebih kering atau lahan campuran.
- **Badan Air** (biru muda) konsisten rendah di semua band, dengan sedikit peningkatan di band biru-hijau (B2–B3), pola khas untuk air jernih atau keruh.

Separabilitas yang jelas antara ketiga kelas di band NIR dan SWIR menjadi dasar kinerja model yang baik.

---

## Korelasi Band dan Indeks

Heatmap ini menampilkan nilai korelasi Pearson antara setiap variabel prediktor dengan masing-masing kelas tutupan lahan. Semakin mendekati +1 (hijau tua) atau -1 (merah tua), semakin kuat hubungan linear variabel tersebut dengan kelas.

![Korelasi Band & Indeks terhadap Tiap Kelas](download__46_.png)

Temuan utama dari heatmap korelasi:

- **NDWI** memiliki korelasi tertinggi dengan kelas Badan Air (+0.96), menjadikannya prediktor terkuat untuk mendeteksi air.
- **NDVI** berkorelasi kuat dengan Mangrove (+0.78) dan berkorelasi negatif kuat dengan Badan Air (-0.96).
- **Band NIR** (B6, B7, B8, B8A) semuanya berkorelasi positif kuat dengan kelas Mangrove (0.68–0.70).
- **Band SWIR** (B11, B12) berkorelasi kuat dengan kelas Non-Mangrove (0.68 dan 0.71), mencerminkan karakteristik lahan kering atau terbangun.
- **IRECI** menunjukkan korelasi 0.74 dengan Mangrove, mengonfirmasi kegunaannya dalam mendeteksi klorofil vegetasi pesisir.

---

## Hasil Klasifikasi

Klasifikasi Random Forest menghasilkan peta tutupan lahan dengan tiga kelas. Model dilatih menggunakan 500 pohon keputusan dengan 15 fitur prediktor.

![Random Forest Classification](download__44_.png)

Peta klasifikasi menunjukkan distribusi spasial yang masuk akal secara ekologis: mangrove (hijau tua) terkonsentrasi di sepanjang tepian sungai dan garis pantai, badan air (biru muda) mendominasi area tambak dan sungai utama, serta non-mangrove (kuning muda) tersebar di lahan pertanian dan permukiman.

---

## Overlay Mangrove

Hasil ekstraksi kelas mangrove dari Random Forest divisualisasikan sebagai overlay warna hijau di atas citra band NIR grayscale, sehingga distribusi spasial mangrove lebih mudah diinterpretasikan dalam konteks lanskap aslinya.

![Overlay Hasil Klasifikasi Mangrove di Atas NIR](download__47_.png)

Overlay ini memperlihatkan bahwa mangrove tumbuh secara linear mengikuti aliran sungai dan membentuk sabuk hijau di tepi pantai. Pola ini konsisten dengan karakteristik ekologi mangrove yang bergantung pada pasokan air payau dari percampuran air sungai dan laut.

---

## Evaluasi Akurasi

Model dievaluasi menggunakan data uji 30% yang dipisahkan secara acak menggunakan `randomColumn`. Evaluasi dilakukan dengan **Confusion Matrix** untuk melihat distribusi prediksi per kelas.

![Confusion Matrix](download__45_.png)

Hasil confusion matrix:

| | Prediksi: Badan Air (0) | Prediksi: Mangrove (1) | Prediksi: Non-Mangrove (2) |
|---|---|---|---|
| **Aktual: Badan Air (0)** | 13 | 0 | 0 |
| **Aktual: Mangrove (1)** | 0 | 83 | 0 |
| **Aktual: Non-Mangrove (2)** | 0 | 6 | 8 |

Dari matriks di atas:

- Kelas **Badan Air** dan **Mangrove** diklasifikasikan dengan sempurna (0 kesalahan).
- Kelas **Non-Mangrove** mengalami 6 piksel yang salah diklasifikasikan sebagai Mangrove, kemungkinan akibat kemiripan spektral vegetasi non-mangrove kering dengan mangrove di beberapa band.

### Metrik Akurasi

| Metrik | Formula | Hasil |
|--------|---------|-------|
| Overall Accuracy | Jumlah benar / Total sampel | ~94.6% (110/116) |
| Kappa Coefficient | (Po - Pe) / (1 - Pe) | Sangat baik (> 0.80) |

Nilai Overall Accuracy di atas 94% dan Kappa yang tinggi mengindikasikan bahwa model Random Forest berhasil memisahkan ketiga kelas tutupan lahan dengan sangat baik, terutama untuk kelas mangrove yang menjadi fokus utama penelitian.

---

> Analisis dilakukan menggunakan Google Earth Engine Python API, geemap, matplotlib, seaborn, dan scikit-learn.
> Citra: Sentinel-2 SR Harmonized (COPERNICUS/S2_SR_HARMONIZED), periode 2024–2025.
