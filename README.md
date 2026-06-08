# Pemetaan Mangrove Menggunakan Random Forest

**Oleh:** Defani Arman Alfitriansyah

Seluruh tahapan analisis dalam notebook ini dijalankan di **Google Colaboratory** menggunakan library `geemap` yang terhubung dengan **Google Earth Engine (GEE)**. Notebook dapat diakses di: https://colab.research.google.com/drive/1QbNTV-tW1OE2vAPIRL58LTYWdCHZ9AuW?usp=sharing

---

## Daftar Isi

1. [Gambaran Umum](#gambaran-umum)
2. [Platform & Package](#platform-dan-package)
3. [Alur Kerja](#alur-kerja)
4. [Indeks Rumus](#indeks-rumus)
   - [NDVI](#1-ndvi)
   - [NDWI](#2-ndwi)
   - [NDMI](#3-ndmi)
   - [NDBI](#4-ndbi)
   - [IRECI](#5-ireci)
5. [Cloud Masking — Cloud Score+](#cloud-masking)
6. [Algoritma Random Forest](#algoritma-random-forest)
7. [Band Sentinel-2 yang Digunakan](#band-sentinel-2)
8. [Tahapan Analisis](#tahapan-analisis)
9. [Evaluasi Akurasi](#evaluasi-akurasi)
10. [Daftar Referensi](#daftar-referensi)

---

## Gambaran Umum

Notebook ini mengimplementasikan klasifikasi tutupan lahan mangrove menggunakan algoritma **Random Forest** berbasis data **Sentinel-2 SR Harmonized** melalui platform **Google Earth Engine (GEE)**. Output berupa peta klasifikasi tiga kelas: Badan Air, Mangrove, dan Non-Mangrove.

---

## Platform dan Package

Seluruh proses komputasi dijalankan di **Google Colaboratory** menggunakan koneksi ke Google Earth Engine melalui library `geemap`.

### Instalasi

```python
!pip install earthengine-api geemap cartopy matplotlib numpy scikit-learn
```

### Package yang Digunakan

| Package | Versi | Fungsi |
|---------|-------|--------|
| `earthengine-api` | latest | Akses Google Earth Engine Python API |
| `geemap` | latest | Visualisasi peta interaktif berbasis GEE di Colab |
| `geemap.cartoee` | (bagian geemap) | Ekspor peta GEE ke matplotlib dengan proyeksi kartografi |
| `cartopy` | latest | Proyeksi peta dan gridline (LONGITUDE_FORMATTER, LATITUDE_FORMATTER) |
| `matplotlib` | latest | Visualisasi dan plotting hasil klasifikasi |
| `numpy` | latest | Operasi array dan matriks |
| `pandas` | latest | Manipulasi data tabular (variable importance) |
| `seaborn` | latest | Heatmap confusion matrix dan visualisasi statistik |
| `scikit-learn` | latest | Metrik evaluasi tambahan |

### Autentikasi GEE

```python
import ee
ee.Authenticate()
ee.Initialize(project='ee-defaniarman')
```

---

## Alur Kerja

```
Sentinel-2 SR Harmonized (2024–2025)
            |
    Cloud Masking (Cloud Score+, cs_cdf >= 0.60)
            |
    Komposit Median + Clip Batas Kajian
            |
    Hitung Indeks: NDVI, NDWI, NDMI, NDBI, IRECI
            |
    Ekstraksi Nilai ke Titik Sampel (sampleRegions)
            |
    Training Random Forest (500 pohon, split 70:30)
            |
    Klasifikasi Citra --> 3 Kelas
            |
    Evaluasi: Confusion Matrix, OA, Kappa
```

---

## Indeks Rumus

### 1. NDVI
**Normalized Difference Vegetation Index**

Mengukur kerapatan dan kehijauan vegetasi.

$$
\text{NDVI} = \frac{B8 - B4}{B8 + B4}
$$

| Simbol | Band Sentinel-2 | Panjang Gelombang |
|--------|-----------------|-------------------|
| B8 | NIR (Near-Infrared) | ~842 nm |
| B4 | Red | ~665 nm |

**Interpretasi:** Nilai mendekati +1 = vegetasi lebat. Nilai mendekati 0 = lahan terbuka. Nilai negatif = air.

```python
ndvi = s2.select('B8').subtract(s2.select('B4')) \
         .divide(s2.select('B8').add(s2.select('B4'))) \
         .rename('NDVI')
```

> Sitasi: Rouse et al. (1973); Tucker (1979); Jamaluddin et al. (2022)

---

### 2. NDWI
**Normalized Difference Water Index**

Mengidentifikasi badan air permukaan dan kelembapan kanopi.

$$
\text{NDWI} = \frac{B3 - B8}{B3 + B8}
$$

| Simbol | Band Sentinel-2 | Panjang Gelombang |
|--------|-----------------|-------------------|
| B3 | Green | ~560 nm |
| B8 | NIR (Near-Infrared) | ~842 nm |

**Interpretasi:** Nilai positif = badan air. Nilai negatif = vegetasi/lahan kering.

```python
ndwi = s2.select('B3').subtract(s2.select('B8')) \
         .divide(s2.select('B3').add(s2.select('B8'))) \
         .rename('NDWI')
```

> Sitasi: McFeeters (1996)

---

### 3. NDMI
**Normalized Difference Moisture Index**

Mengukur kandungan air pada vegetasi (kelembapan kanopi).

$$
\text{NDMI} = \frac{B8 - B11}{B8 + B11}
$$

| Simbol | Band Sentinel-2 | Panjang Gelombang |
|--------|-----------------|-------------------|
| B8 | NIR (Near-Infrared) | ~842 nm |
| B11 | SWIR-1 (Short-Wave Infrared) | ~1610 nm |

**Interpretasi:** Nilai tinggi = vegetasi dengan kelembapan tinggi (cocok untuk mangrove). Nilai rendah = vegetasi kering atau lahan terbuka.

```python
ndmi = s2.select('B8').subtract(s2.select('B11')) \
         .divide(s2.select('B8').add(s2.select('B11'))) \
         .rename('NDMI')
```

> Sitasi: Shi et al. (2016); Jamaluddin et al. (2022)

---

### 4. NDBI
**Normalized Difference Built-up Index**

Mengidentifikasi kawasan terbangun/permukiman.

$$
\text{NDBI} = \frac{B11 - B8}{B11 + B8}
$$

| Simbol | Band Sentinel-2 | Panjang Gelombang |
|--------|-----------------|-------------------|
| B11 | SWIR-1 (Short-Wave Infrared) | ~1610 nm |
| B8 | NIR (Near-Infrared) | ~842 nm |

Catatan: NDBI adalah kebalikan dari NDMI. Nilai positif mengindikasikan kawasan terbangun; nilai negatif mengindikasikan vegetasi.

```python
ndbi = s2.select('B11').subtract(s2.select('B8')) \
         .divide(s2.select('B11').add(s2.select('B8'))) \
         .rename('NDBI')
```

> Sitasi: Zha et al. (2003)

---

### 5. IRECI
**Inverted Red-Edge Chlorophyll Index**

Mengukur kandungan klorofil menggunakan band Red-Edge — sangat sensitif terhadap vegetasi mangrove.

$$
\text{IRECI} = \frac{B8 - B4}{B5 / B6}
$$

Dapat ditulis juga sebagai:

$$
\text{IRECI} = \frac{(B8 - B4) \times B6}{B5}
$$

| Simbol | Band Sentinel-2 | Panjang Gelombang |
|--------|-----------------|-------------------|
| B8 | NIR (Near-Infrared) | ~842 nm |
| B4 | Red | ~665 nm |
| B5 | Red-Edge 1 | ~705 nm |
| B6 | Red-Edge 2 | ~740 nm |

**Interpretasi:** Nilai tinggi = kandungan klorofil tinggi, vegetasi sehat dan lebat. Sangat berguna untuk membedakan mangrove dari vegetasi lain.

```python
ireci = s2.select('B8').subtract(s2.select('B4')) \
          .divide(s2.select('B5').divide(s2.select('B6'))) \
          .rename('IRECI')
```

> Sitasi: Clevers et al. (2000); Suardana et al. (2023)

---

## Cloud Masking

Proses cloud masking menggunakan **Cloud Score+ (CS+)**, produk quality assessment (QA) berbasis deep learning yang dikembangkan oleh Google. CS+ menghasilkan skor QA per piksel pada rentang kontinu [0, 1], di mana nilai mendekati 1 = piksel bersih dan nilai mendekati 0 = piksel terkontaminasi awan atau bayangan awan.

### Model QA

Setiap piksel dimodelkan sebagai kombinasi linear antara reflektansi asli permukaan dan kontaminasi atmosfer:

$$
p_m = p_t \cdot q + c \cdot (1 - q)
$$

$$
QA(p_m) = q
$$

| Simbol | Keterangan |
|--------|------------|
| $p_m$ | Nilai piksel yang diukur sensor |
| $p_t$ | Reflektansi asli permukaan |
| $c$ | Komponen kontaminasi atmosfer |
| $q$ | Skor QA piksel, $q \in [0, 1]$ |

### Atmospheric Similarity (ASIM)

$$
ASIM_i(x_i, y_i) := \min_{p \in \{x_i, y_i\}} QA(p)
$$

Jika $r$ adalah citra referensi bersih ($QA(r) = 1$), maka $ASIM(x, r) = QA(x)$.

### Log Transform Input

$$
x' = \frac{\log(x + 1)}{10}
$$

### Penerapan dalam Notebook

```python
csPlus = ee.ImageCollection('GOOGLE/CLOUD_SCORE_PLUS/V1/S2_HARMONIZED')

def maskS2CSPlus(image):
    qa = image.select('cs_cdf')
    return image.updateMask(qa.gte(0.60))

s2 = (s2_base
    .linkCollection(csPlus, ['cs_cdf'])
    .map(maskS2CSPlus)
    .median()
    .clip(batas_kajian)
    .divide(10000))
```

Threshold `cs_cdf >= 0.60` digunakan untuk menyeimbangkan omission dan commission error.

> Sitasi: Pasquarella et al. (2023)

---

## Algoritma Random Forest

Random Forest (RF) adalah algoritma ensemble berbasis pohon keputusan yang menggunakan **majority voting** dari sejumlah pohon untuk menentukan kelas akhir setiap piksel.

<img width="721" height="437" alt="Screenshot 2026-06-09 012602" src="https://github.com/user-attachments/assets/cec19ddd-14f9-463e-b383-f8bfc73ffa12" />


Dataset dibagi ke sejumlah pohon keputusan (Tree 1, Tree 2, ..., Tree 500). Setiap pohon menghasilkan prediksi kelas secara independen. Kelas akhir ditentukan melalui **Majority Voting** dari seluruh pohon.

### Implementasi dalam GEE

```python
classifier = ee.Classifier.smileRandomForest(500).train(
    features=training_data,
    classProperty='lc',
    inputProperties=all_bands
)
classified_image = image_with_indices.select(all_bands).classify(classifier)
```

| Parameter | Nilai |
|-----------|-------|
| Jumlah pohon | 500 |
| Properti kelas | `lc` (land cover) |
| Fitur prediktor | 15 (10 band + 5 indeks) |
| Split data | 70% train / 30% test |

> Sitasi: Breiman (2001); Jamaluddin et al. (2022)

---

## Band Sentinel-2

| Band | Nama | Panjang Gelombang | Resolusi |
|------|------|-------------------|----------|
| B2 | Blue | ~490 nm | 10 m |
| B3 | Green | ~560 nm | 10 m |
| B4 | Red | ~665 nm | 10 m |
| B5 | Red-Edge 1 | ~705 nm | 20 m |
| B6 | Red-Edge 2 | ~740 nm | 20 m |
| B7 | Red-Edge 3 | ~783 nm | 20 m |
| B8 | NIR | ~842 nm | 10 m |
| B8A | Narrow NIR | ~865 nm | 20 m |
| B11 | SWIR-1 | ~1610 nm | 20 m |
| B12 | SWIR-2 | ~2190 nm | 20 m |
| NDVI | Vegetation Index | — | — |
| NDWI | Water Index | — | — |
| NDMI | Moisture Index | — | — |
| NDBI | Built-up Index | — | — |
| IRECI | Red-Edge Chlorophyll Index | — | — |

**Total prediktor:** 15 fitur (10 band spektral + 5 indeks turunan)

> Sitasi: ESA (2012); ESA (2015); Suardana et al. (2023)

---

## Tahapan Analisis

### 1. Preprocessing Citra

- Sumber data: `COPERNICUS/S2_SR_HARMONIZED`
- Periode: 1 Januari 2024 – 10 Desember 2025
- Filter tutupan awan: `CLOUDY_PIXEL_PERCENTAGE < 20`
- Cloud masking: Cloud Score+ (`cs_cdf >= 0.60`)
- Komposit: Median seluruh scene yang lolos filter
- Normalisasi reflektansi: dibagi 10.000
<img width="868" height="837" alt="download (43)" src="https://github.com/user-attachments/assets/b0ed1456-09f8-4b3c-9fd7-0bb004eca11a" />

### 2. Pelatihan Model

- Algoritma: `ee.Classifier.smileRandomForest(500)`
- Jumlah pohon: 500
- Properti kelas: `lc`
- Split: 70% train / 30% test (`randomColumn`)

### 3. Kelas Klasifikasi

| Kode | Kelas | Warna |
|------|-------|-------|
| 0 | Badan Air | Biru muda |
| 1 | Mangrove | Hijau |
| 2 | Non-Mangrove | Kuning muda |
<img width="868" height="837" alt="download (44)" src="https://github.com/user-attachments/assets/852c7d25-ee35-45f3-9974-d6fde20bdde2" />

---

## Evaluasi Akurasi
<img width="551" height="451" alt="download (45)" src="https://github.com/user-attachments/assets/250dc752-33f4-4955-8f66-b79dd1152a89" />

### Overall Accuracy (OA)

$$
\text{OA} = \frac{\sum_{i} m_{ii}}{\sum_{i}\sum_{j} m_{ij}}
$$

### User's Accuracy (UA) dan Producer's Accuracy (PA)

$$
UA = \frac{TN}{TN + FP}
$$

$$
PA = \frac{TP}{TP + FN}
$$

> Sitasi: Jamaluddin et al. (2022)

### Kappa Coefficient

$$
\kappa = \frac{P_o - P_e}{1 - P_e}
$$

| Nilai Kappa | Interpretasi |
|-------------|--------------|
| > 0.80 | Sangat baik |
| 0.60 – 0.80 | Baik |
| 0.40 – 0.60 | Sedang |
| < 0.40 | Lemah |

### Hasil

| Metrik | Nilai | Interpretasi |
|--------|-------|--------------|
| Overall Accuracy | **0.9455** | 94.55% piksel diklasifikasikan dengan benar |
| Kappa Coefficient | **0.8511** | Sangat baik (> 0.80) |

---

## Daftar Referensi

**Algoritma & Model**

Breiman, L. (2001). Random Forests. *Machine Learning*, 45, 5–32. https://doi.org/10.1023/A:1010933404324

**Cloud Masking**

Pasquarella, V. J., Brown, C. F., Czerwinski, W., & Rucklidge, W. J. (2023). Comprehensive quality assessment of optical satellite imagery using weakly supervised video learning. *CVPR Workshop 2023*, pp. 2125–2135.

**Indeks Vegetasi**

Clevers, J. G. P. W., Jong, S. M. De, Epema, G. F., Addink, E. A., & Box, P. O. (2000). MERIS and the Red-Edge Index. *2nd EARSeL Workshop*, Enschede. *(IRECI)*

McFeeters, S. K. (1996). The use of the Normalized Difference Water Index (NDWI) in the delineation of open water features. *International Journal of Remote Sensing*, 17(7), 1425–1432. *(NDWI)*

Rouse, J. W., Haas, R. H., Schell, J. A., & Deering, D. W. (1973). Monitoring Vegetation Systems in the Great Plains with ERTS. *Remote Sensing Center*. *(NDVI)*

Shi, T., Liu, J., Hu, Z., Liu, H., Wang, J., & Wu, G. (2016). New spectral metrics for mangrove forest identification. *Remote Sensing Letters*, 7(9), 885–894. *(NDMI)*

Tucker, C. J. (1979). Red and photographic infrared linear combinations for monitoring vegetation. *Remote Sensing of Environment*, 8, 127–150. *(NDVI, DVI)*

Zha, Y., Gao, J., & Ni, S. (2003). Use of normalized difference built-up index in automatically mapping urban areas from TM imagery. *International Journal of Remote Sensing*, 24(3), 583–594. *(NDBI)*

**Pemetaan Mangrove dengan Random Forest**

Jamaluddin, I., Chen, Y.-N., Ridha, S. M., Mahyatar, P., & Ayudyanti, A. G. (2022). Two Decades Mangroves Loss Monitoring Using Random Forest and Landsat Data in East Luwu, Indonesia (2000–2020). *Geomatics*, 2, 282–296. https://doi.org/10.3390/geomatics2030016

Suardana, A. A. Md. A. P., Anggraini, N., Nandika, M. R., Aziz, K., As-syakur, A. R., Ulfa, A., Wijaya, A. D., Prasetio, W., Winarso, G., & Dewanti, R. (2023). Estimation and Mapping Above-Ground Mangrove Carbon Stock Using Sentinel-2 Data Derived Vegetation Indices in Benoa Bay of Bali Province, Indonesia. *Forest and Society*, 7(1), 116–134. https://doi.org/10.24259/fs.v7i1.22062

**Data Satelit**

ESA. (2012). *Sentinel-2: ESA's Optical High-Resolution Mission for GMES Operational Services*.

ESA. (2015). *Sentinel-2 User Handbook*. ESA Standard Document.

Gorelick, N., Hancher, M., Dixon, M., Ilyushchenko, S., Thau, D., & Moore, R. (2017). Google Earth Engine: Planetary-scale geospatial analysis for everyone. *Remote Sensing of Environment*, 202, 18–27.

**Software & Library**

Jupyter / Google Colaboratory: https://colab.research.google.com

Wu, Q. (2020). geemap: A Python package for interactive mapping with Google Earth Engine. *Journal of Open Source Software*, 5(51), 2305. https://doi.org/10.21105/joss.02305

Wu, Q., & Landgrebe, T. (2023). *geemap* (Python package). https://geemap.org

Pedregosa, F., Varoquaux, G., Gramfort, A., Michel, V., Thirion, B., Grisel, O., ... Duchesnay, É. (2011). Scikit-learn: Machine Learning in Python. *Journal of Machine Learning Research*, 12, 2825–2830.

Hunter, J. D. (2007). Matplotlib: A 2D graphics environment. *Computing in Science & Engineering*, 9(3), 90–95.

Harris, C. R., et al. (2020). Array programming with NumPy. *Nature*, 585, 357–362.

Waskom, M. (2021). seaborn: statistical data visualization. *Journal of Open Source Software*, 6(60), 3021. https://doi.org/10.21105/joss.03021
