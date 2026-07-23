# EDA Bulguları — Birleşik (Betül + Erdem)

İki bağımsız EDA'nın (`bet_exp/eda` → PR #4, `erd_exp/eda_detailed`) birleştirilmiş sonucu,
artı veri dosyalarından **doğrudan doğrulanan** gerçekler. Çelişkiler çözüldü.

> **Dosya adı notu:** Erdem'in dalında `docs/EDA_BULGULARI.md`, Betül'ün dalında
> `docs/eda_bulgulari.md` vardı. macOS case-insensitive olduğu için ikisi çakışır.
> Bundan sonra **tek isim: `EDA_BULGULARI.md`**.

---

## 0. ⚠️ Ortam kısıtları — her şeyi belirliyor

Bu bir **code competition**: seçilen notebook, **gizli test seti veriyle değiştirilerek**
private olarak yeniden çalıştırılıyor ve skor o çalıştırmanın çıktısından hesaplanıyor.

| Kısıt | Sonuç |
|---|---|
| **Internet KAPALI zorunlu** | `pip install` **imkânsız** — utility dataset şart |
| **`zarr` imajda YOK** | Veri okunamaz! → `cell-tracking-libs` dataset'i **zorunlu** |
| **Test verisi runtime'da değişiyor** | Test dataset isimlerini **asla hardcode etme**, `test/*.zarr` glob'la |
| **GPU yok** (`torch 2.10.0+cpu`) | 3B U-Net eğitimi gerçekçi değil |

> ⚠️ **Envanteri internet KAPALIYKEN al.** Internet açıkken alınan liste yanıltıcı
> (orada `zarr` görünüyor ama submit ortamında yok). Aşağıdaki liste submit modundan.

**İmajda VAR:** `blosc2`, `zstandard`, `dask`, `xarray`, `tensorstore`, `skimage 0.25.2`,
`scipy 1.16.3`, `networkx 3.6.1`, `polars`, `pandas`, **`numpy 2.0.2`**, `numba 0.60`,
`pulp`, `cvxpy`, `sqlalchemy`, `torch 2.10.0+cpu`.

**İmajda YOK:** **`zarr`**, **`numcodecs`**, `ultrack`, `tracksdata`, `geff`, `monai`,
`cupy`, `higra`, `edt`.

### 🔑 Zorunlu kurulum: `cell-tracking-libs` utility dataset

Erdem'in hazırladığı dataset (`erdeemt/cell-tracking-libs`) `pylibs/` altında **açılmış
paketler** içerir (pip ile kurulmaz, `sys.path`'e eklenir): `zarr 3.2.1`, `numcodecs 0.16.5`,
`donfig`, `packaging`, `pyyaml`, `typing_extensions`, `google_crc32c`, **`numpy 2.5.1`**.

```python
import sys
sys.path.append('/kaggle/input/datasets/erdeemt/cell-tracking-libs/pylibs')
import zarr   # 3.2.1
```

> ❗ **`append` kullan, `insert(0)` KULLANMA.** pylibs'te `numpy 2.5.1` var; başa eklersen
> ortamın `numpy 2.0.2`'sini gölgeler ve ona karşı derlenmiş `scipy`/`skimage`/`torch`
> riske girer. `append` ile numpy ortamdan, zarr/numcodecs pylibs'ten gelir.

✅ **Doğrulandı** (internet kapalı): `numpy` → 2.0.2 (ortam), `zarr` → pylibs;
görüntü `(64,256,256) uint16` ve GEFF okundu (numcodecs blosc/zstd numpy 2.0.2 altında
çalışıyor); `scipy`/`skimage` sağlam.

> **Dataset, submit edilen notebook'a ekli olmalı** — private re-run'da da gerekli.

> **Yol haritasının iki ana yolu da kapalı.** Ayrıntı ve revize plan: [YOL_HARITASI.md](../YOL_HARITASI.md).
> `tracksdata`/`geff`'in yokluğu önemsiz — GEFF'i düz `zarr` ile okuyoruz.

### ❗ "Test = train" bir sızıntı DEĞİL
`test/` altındaki 4 dosya, `train/`deki aynı isimli dosyalarla **bayt-bayt aynı** (md5 doğrulandı).
Bu bir sızıntı değil, **placeholder**: gerçek gizli test private re-run'da yerlerine geçiyor.
Train GT'yi submit etmek gizli test üzerinde ~0 verirdi.

---

## 1. Veri yapısı — dosyadan doğrulandı

```
/kaggle/input/competitions/biohub-cell-tracking-during-development/
├── sample_submission.csv
├── train/  199 × (.zarr + .geff)
└── test/   4 × .zarr  (placeholder; runtime'da değişir)
```

- **Görüntü:** her örnek `(T,Z,Y,X) = (100, 64, 256, 256)`, `uint16`, **zarr v3**, OME-NGFF 0.5.
- **Chunk = `[1, 64, 256, 256]`** → bir zaman karesi = bir chunk (~8.4 MB). Kare kare oku, tüm hacmi RAM'e alma.
- **Ölçek (zarr attrs'ten, otoritatif):** `scale (T,Z,Y,X) = [1.0, 1.625, 0.40625, 0.40625]`, birim **micrometer**.
  - Z ekseni X/Y'den **4× kaba**. Hacim ≈ 104³ µm.
  - ✅ Yol haritasındaki değerler doğruymuş. Z dizi boyutu **64** (Erdem'in dokümanındaki
    "~62–63 dilim" derinlikten türetilmiş bir yuvarlama artefaktıydı).

### Bedava normalizasyon sabitleri
`.zarr` attrs içinde **`image_statistics.quantiles`** hazır geliyor — hesaplamaya gerek yok:

| q | 0.0 | 0.001 | 0.01 | 0.1 | 0.9 | 0.99 | 0.999 | 1.0 |
|---|-----|-------|------|-----|-----|------|-------|-----|
| değer | 15 | 26.2 | 38 | 75 | 497 | 1478 | 2145 | 4319 |

Arka plan ~75, çekirdekler 1500+.

---

## 2. Ground-truth (GEFF) — şema kesinleşti

`geff_version 1.1`, `directed: true`, zarr v3. Yapı:

```
<sample>.geff/
├── nodes/ids            (N,)   uint64   — keyfi id'ler
├── nodes/props/{t,z,y,x}/values   (N,)  int64
├── edges/ids            (E,2)  uint64   — (source, target)
└── edges/props/         BOŞ
```

- **Koordinatlar VOXEL ve int64** (float değil). Üç bağımsız kanıt: GEFF axes aralıkları
  görüntü boyutlarıyla birebir (`z∈[1,63]`, `y∈[74,230]`, `x∈[73,253]`); `sample_submission`
  örneği `z=32,y=128,x=128` = hacim merkezi; axes metadata `scale`'i ayrıca taşıyor.
  → **Mesafeler için `voxel × (1.625, 0.40625, 0.40625)`.**
- **Track/lineage id property'si YOK** → soylar graf bağlantısından türetilir.

---

## 3. GT seyrek — ama sandığımız şekilde değil

- Toplam **133 318 node**, **128 883 edge** (199 örnek) → ortalama **~6.6 etiketli node/kare**.
- Görüntüde ise kare başına **yüzlerce** çekirdek var. GT = embriyonun tamamı değil,
  video başına **~15–25 hücre soyunun** baştan sona etiketlenmesi.
- **Zamansal boşluk YOK:** bir soyun içinde kesinti yok (`uzunluk ≈ süre`).
  → **gap-closing gerekmez.** (Betül'ün ilk "boşluk var" okuması, farklı soyların farklı
  zamanlarda başlamasından doğan bir aggregate artefaktıydı.)

### İki domain: `44b6` vs `6bba` → CV buna göre bölünmeli
| Grup | n | Medyan node | node/kare | Bölünme |
|------|---|-------------|-----------|---------|
| 44b6 | 71 | 214 | **2.1** | 26 |
| 6bba | 128 | 826 | **8.4** | 125 |

Farklı embriyolar, etiket yoğunluğu **~4× farklı**. **CV'yi prefix'e göre böl** (leakage),
iki domaini de temsil et.

---

## 4. `T_true` ve fazla-tahmin cezası — en kritik nokta

Resmi [metrics.md](https://github.com/royerlab/kaggle-cell-tracking-competition/blob/main/metrics.md):

> `adjusted_jaccard = max(0, jaccard · (1 − 0.1 · (T_pred − T_true) / T_true))`
> `T_pred` = tahmin edilen **toplam node sayısı**; `T_true` = **sağlanan kaba tahmin**,
> **ground-truth'un etiketlemediği hücreler dahil** toplam gerçek node sayısı.

**Sonuç:** `T_true` seyrek GT node sayısı (~600/örnek) **değil**, gerçek hücre sayısı
(~21 000/örnek ≈ **213/kare**). Yani **tüm hücreleri tespit etmeliyiz** — "sadece etiketlileri
tahmin et" hem imkânsız hem gereksiz. Ceza yalnızca *gerçek hücre sayısını aşarsan* devreye girer.

> ⚠️ **`T_true` bize VERİLMİYOR** — veri dizininde yok, skorlayıcı sunucu tarafında kullanıyor.
> Erdem'in blob sayacı tahmini (**~213 çekirdek/kare**, örnekler arası ~50–730) elimizdeki
> tek proxy. **`T_pred ≈ 200/kare` hedefle.**

---

## 5. Detection — sinyal güçlü, zorluk instance ayrımı

- GT merkezlerinde yoğunluk medyan ~1750; rastgele voxel ~200 → **global kontrast ~8×**.
- **AMA yerel komşu kontrastı sadece ~1.5× SNR** — çekirdekler paketli/temas halinde.
  → **Asıl zorluk foreground değil, birbirine değen çekirdekleri ayırmak.**
- Çekirdek ~**10 µm çap** (yarı-maks yarıçap ~5 µm), elipsoid. Her Z derinliğinde GT var.
- **Z profili:** yüzeyde (z=0) zayıf, orta derinlikte (z≈45–90 µm) tepe, uçta düşüş.
- **Zaman profili:** ortalama yoğunluk t≈30'da tepe, sona doğru düşüş (photobleaching olabilir).

---

## 6. Linking — kolay taraf

- Kare-arası yer değiştirme: medyan **~2–3 µm**, 95p **~5.5 µm**, 99p **~8 µm**, **~%95 < 7 µm**.
  İnce kuyruk 15–60 µm (nadir).
  → **`max_distance ≈ 8 µm`** (95–99p'yi kapsar).
- **Eşleşme belirsizliği yok:** kare-içi GT-GT en yakın komşu medyan **~25 µm**; <7 µm sadece **%1–2**.
  7 µm toleransı güvenli.
- Yön kalıcılığı **kısmi** (hızlılar düz gider, yavaşlar jitter). Hareket modeli **marjinal fayda**
  → **NN/Hungarian yeterli.**
- **Track bitişleri alan kenarında yoğun** → hücreler görüş alanından çıkıyor
  → **appearance/disappearance maliyeti gerekli**.

---

## 7. Bölünmeler — nadir, sona bırak

- Toplam **151 bölünme / 133 318 node = %0.11**. Örneklerin **112/199'unda sıfır**, max 5.
- Geometri: ebeveyn-kız **~6 µm**, kız-kız ~11 µm.
  → ebeveyn-kız < linking yarıçapı, yani **aynı ~8 µm radius bölünme kenarlarını da yakalar**.
- Skorun %10'u + ±1 kare toleransı + nadirlik → **erken optimizasyon getirisi düşük**.

---

## 8. Gönderim formatı (`sample_submission.csv`)

Kolonlar: `id, dataset, row_type, node_id, t, z, y, x, source_id, target_id`

- **node** satırı: `row_type=node`, `node_id` (dataset içinde **1'den** başlar, ardışık),
  `t,z,y,x` (**voxel**), `source_id=target_id=-1`.
- **edge** satırı: `row_type=edge`, `node_id=-1`, `t=z=y=x=-1`, `source_id→target_id`
  (t'deki node_id → **t+1**'deki node_id).
- **Bölünme:** ebeveyn, iki ayrı edge satırında `source_id` olarak yer alır.
- `id` = global satır indeksi (0,1,2,…). `dataset` = test örnek adı.
- Dosya adı **`submission.csv`** olmalı.

✅ Doğrulandı: train GT'den bu şemayla CSV üretildi, sample ile kolonlar birebir eşleşti
(4320 satır; node_id'ler ardışık, tüm edge'ler t→t+1).

---

## Baseline kararları (klasik pipeline — tek uygulanabilir yol)

1. **Ölçek:** her dataset'in zarr attrs'inden oku (hardcode etme).
2. **Normalizasyon:** attrs'teki `image_statistics.quantiles` kullan.
3. **Detection:** Otsu/eşik + anizotropik `maximum_filter` ile local-max.
   Çekirdek ~10 µm → bastırma yarıçapı ~5 µm = voxel'de `(3, 12, 12)`.
4. **`T_pred` hedefi ≈ 200/kare** — eşiği buna kalibre et (fazla-tahmin cezası).
5. **Linking:** Hungarian (`scipy.optimize.linear_sum_assignment`), gate **8 µm**,
   mesafe µm cinsinden; gap-closing **kapalı**; eşleşmeyenler appear/disappear.
6. **Bölünme:** baseline'da **atla** (%10).
7. **CV:** prefix-bazlı (`44b6`/`6bba`).
8. **Öncelik:** edge %90 → önce linking'i sağlamlaştır.

## Açık sorular
- [ ] Accelerator GPU yapılınca `torch` CUDA build'i geliyor mu? (gelirse DL yolu açılır)
- [ ] Gizli test setinin boyutu bilinmiyor → pipeline'ın çalışma süresi bütçelenmeli.
- [ ] `T_true` tahminimiz (~213/kare) ne kadar isabetli? Skor geri bildirimiyle kalibre et.
