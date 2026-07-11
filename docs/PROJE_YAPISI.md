# Proje Dizin Yapısı

Deneyler **Kaggle notebook** üzerinde çalışır; bu repo notebook + yardımcı kod + config
versiyonlaması için kullanılır. Aşağıdaki dosyalar şu an **boş iskelet** — içerik projeyle
birlikte doldurulacak.

```
3d_bio/
├── README.md                     # Giriş + çalışma modeli
├── YOL_HARITASI.md               # Strateji, metrik, haftalık plan
├── requirements.txt              # Kaggle notebook'ta pip install referansı
│
├── docs/
│   └── PROJE_YAPISI.md           # (bu dosya)
│
├── notebooks/                    # ⭐ ANA deney yüzeyi (Kaggle'da çalışır)
│   ├── 01_eda.ipynb              # Veri keşfi: zarr/geff yükle, napari ile incele
│   ├── 02_baseline_ultrack.ipynb # YOL A: Ultrack ile ilk gönderim
│   ├── 03_detection.ipynb        # YOL B: 3B U-Net detection eğitimi/çıkarımı
│   ├── 04_linking.ipynb          # Bağlama + bölünme; graf kurma
│   ├── 05_evaluation.ipynb       # Yerel metrik (Edge/Division Jaccard), hata analizi
│   └── 06_submission.ipynb       # graf → .geff → CSV, submit
│
├── src/                          # Notebook'lara import edilen yardımcı modüller
│   │                             #   (Kaggle utility dataset olarak paketlenir)
│   ├── data/
│   │   ├── dataset.py            # open_dataset, list_samples (OME-Zarr + GEFF)
│   │   └── preprocess.py         # normalizasyon, patch, anizotropi
│   ├── detection/
│   │   ├── detector.py           # merkez tespiti (ultrack_auto | unet3d)
│   │   └── unet3d.py             # 3B U-Net (MONAI) + local-max suppression
│   ├── linking/
│   │   └── linker.py             # graf bağlama (ultrack ILP | hungarian) + bölünme
│   ├── eval/
│   │   ├── metrics.py            # Edge + Division Jaccard (7 µm)
│   │   └── submission.py         # .geff → CSV dönüşümü
│   └── utils/
│       ├── io.py                 # yol/ortam yardımcıları (kaggle vs. local)
│       └── viz.py                # napari 3B görselleştirme (TP/FP/FN)
│
├── configs/                      # deney parametreleri (yaml)
│   ├── baseline_ultrack.yaml     # YOL A
│   └── unet3d.yaml               # YOL B
│
├── kaggle/                       # Kaggle'a özel meta
│   ├── kernel_metadata/          # `kaggle kernels push` için kernel-metadata.json
│   └── utility_dataset/          # src/'yi dataset olarak yüklemek için (kaggle datasets)
│
└── outputs/                      # Kaggle'dan indirilen artifact kopyaları
    ├── predictions/              # .geff / ara tahminler
    ├── submissions/              # arşivlenen submission CSV'leri (skorlarla)
    └── figures/                  # rapor/görsel çıktılar
```

## Neden bu düzen?

- **notebooks/ merkezde:** Tüm çalıştırma Kaggle'da olduğu için deneyler numaralı
  defterlerde ilerler (01→06 pipeline sırasını yansıtır).
- **src/ ayrı:** Notebook hücrelerinde kod tekrarını önlemek için yardımcı mantık modüllere
  taşınır; Kaggle'a **utility dataset** olarak eklenip `import src...` ile kullanılır.
  Böylece kod hem versiyonlanır hem de defterler temiz kalır.
- **Yerel veri klasörü yok:** Yarışma verisi Kaggle'da `/kaggle/input/...` altında hazır
  bağlı; yerelde indirmiyoruz.
- **outputs/ sadece arşiv:** Ağır ağırlıklar Kaggle dataset'lerinde tutulur; burada yalnızca
  submission CSV'leri ve raporlanacak görseller versiyonlanır.

## `.gitignore` önerisi

```
outputs/predictions/*
!outputs/predictions/.gitkeep
*.zarr
*.geff
*.pth
.ipynb_checkpoints/
```
