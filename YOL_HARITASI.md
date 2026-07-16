# Biohub – Cell Tracking During Development
## Kaggle Yarışması: Kapsamlı Yol Haritası

> **Yarışma:** [Biohub - Cell Tracking During Development](https://www.kaggle.com/competitions/biohub-cell-tracking-during-development)
> **Düzenleyen:** Chan Zuckerberg Biohub – Royer Lab
> **Başlangıç:** 29 Haziran 2026
> **Konu:** Zebra balığı (zebrafish) embriyosunda hücrelerin 3B uzayda ve zamanda tespiti ve takibi

---

## 1. Yarışmanın Amacı Nedir? (Özet)

Zebra balığı embriyosunun gelişiminin ilk saatleri boyunca **ışık-tabakası mikroskopu (light-sheet microscopy)** ile çekilmiş 3B zaman-serisi (film) görüntüleri veriliyor. Bir embriyoda binlerce hücre var ve bunlar zamanla:

- **hareket ediyor** (göç / migration),
- **bölünüyor** (mitoz / cell division),
- yeni dokular oluşturuyor.

**Göreviniz:** Her hücrenin merkezini bulmak (detection) ve aynı hücreyi ardışık zaman kareleri boyunca birbirine bağlayarak **soy ağacını (lineage / track)** çıkarmak. Yani "t anındaki bu hücre, t+1 anındaki hangi hücredir?" sorusunu ve "bu hücre ne zaman ikiye bölündü?" sorusunu cevaplamak.

Bu, biyolojide **kök hücre**, **bağışıklık yanıtı**, **doku rejenerasyonu** ve **kanser** araştırmalarının temelini oluşturan bir problem. Yarışma, Royer Lab'in 2025'te *Nature Methods*'ta yayımlanan **Ultrack** algoritmasının üzerine kuruluyor.

### Problem tipi
Bu klasik bir sınıflandırma/regresyon değil. İki alt problemi birleştiren bir **"tracking-by-detection"** problemi:
1. **Detection (Tespit):** 3B hacimde hücre çekirdeklerinin merkez koordinatlarını `(t, z, y, x)` bul.
2. **Linking (Bağlama):** Ardışık karelerdeki tespitleri bir **graf (node + edge)** olacak şekilde birbirine bağla; bölünmeleri (1 ebeveyn → 2 çocuk) doğru yakala.

---

## 1.5. ⚠️ ORTAM KISITLARI — Planı Belirleyen Gerçekler

> Bunlar veriden ve Kaggle submit ekranından **doğrulandı** (bkz. [docs/EDA_BULGULARI.md](docs/EDA_BULGULARI.md)).
> Aşağıdaki 4. bölümün orijinal planını **geçersiz kılar**.

Bu bir **code competition**: seçtiğiniz notebook, **gizli test seti veriyle değiştirilerek**
private olarak yeniden çalıştırılır; skor o çalıştırmanın `submission.csv` çıktısından hesaplanır.

| Kısıt | Sonuç |
|---|---|
| **Internet KAPALI zorunlu** | `pip install` **imkânsız** — utility dataset şart |
| **`zarr` imajda YOK** | Veri okunamaz → `cell-tracking-libs` dataset'i **zorunlu** |
| **Test verisi runtime'da değişir** | Test isimlerini **hardcode etme**, `test/*.zarr` glob'la |
| **GPU yok** (`torch 2.10.0+cpu`) | 3B U-Net eğitimi gerçekçi değil |

- **VAR:** `skimage`, `scipy`, `networkx`, `polars`, `pandas`, `numpy 2.0.2`, `numba`,
  `pulp`, `cvxpy`, `torch(+cpu)`, `dask`, `xarray`, `blosc2`, `zstandard`
- **YOK:** **`zarr`**, **`numcodecs`**, `ultrack`, `tracksdata`, `geff`, `monai`, `cupy`, `higra`, `edt`

**Zorunlu ilk hücre** (envanteri internet KAPALIYKEN al — açıkken liste yanıltıcı):
```python
import sys
sys.path.append('/kaggle/input/datasets/erdeemt/cell-tracking-libs/pylibs')  # append! insert(0) DEGIL
import zarr
```
Dataset `zarr 3.2.1` + `numcodecs` sağlar; içindeki `numpy 2.5.1` ortamın `2.0.2`'sini
gölgelememeli, o yüzden **`append`**. Dataset submit edilen notebook'a **ekli** olmalı.

**`test/` içindeki dosyalar placeholder'dır** — `train/`deki aynı isimlilerle bayt-bayt aynı
(md5 doğrulandı). Sızıntı değil; gerçek test private re-run'da yerlerine geçer.

---

## 2. Veri Yapısı (Çok Önemli — İlk Gün Anlaşılmalı)

```
/kaggle/input/competitions/biohub-cell-tracking-during-development/
├── sample_submission.csv
├── train/    → eşleşmiş .zarr (görüntü) + .geff (ground-truth track grafı)  [199 örnek]
└── test/     → sadece .zarr (görüntü) — placeholder; runtime'da gizli setle değişir
```

### 2.1 Görüntü formatı: OME-Zarr
- **Boyut düzeni:** `(T, Z, Y, X)` — Zaman, Derinlik, Yükseklik, Genişlik.
- **Fiziksel ölçek (anisotropik!):**
  - Z = **1.625 µm/piksel**
  - Y = **0.40625 µm/piksel**
  - X = **0.40625 µm/piksel**
- ⚠️ **Dikkat:** Z ekseni, X/Y'den ~4 kat daha kaba. Mesafe hesaplarken pikselleri değil **mikrometreyi (µm)** kullanmalısınız. Metriğin eşleştirme toleransı **7 µm**. Bunu unutmak en sık yapılan hatadır.
- Zarr formatı sayesinde tüm terabaytlık veriyi RAM'e yüklemeden **parça parça (chunk)** okuyabilirsiniz.

### 2.2 Ground-truth formatı: GEFF (graph) — tracksdata
- **Node (düğüm):** Bir hücre merkezi = `(t, z, y, x)` float koordinat.
- **Edge (kenar):** Aynı hücreyi `t` → `t+1` boyunca bağlayan ok.
- **Bölünme (division):** Bir ebeveyn node, `t+1`'de **iki** çocuk node'a bağlanır.
- ⚠️ **KRİTİK:** Ground-truth **seyrek (sparse)**. Yani her karede hücrelerin sadece bir alt kümesi etiketli. Bu, eğitim ve değerlendirme stratejinizi tümüyle etkiler (aşağıda "seyrek etiket" bölümüne bakın).

### 2.3 Gönderim (submission) formatı
1. Test verisi için `.geff` tahmin dosyaları üret (her veri seti için bir dosya).
2. `geffs_to_csv.py` ile GEFF → CSV'ye çevir.
3. CSV'yi Kaggle'a yükle.
4. (Yerelde) CSV → GEFF geri çevirip ground-truth ile skorla.

---

## 3. Değerlendirme Metriği (Skorun Nasıl Hesaplandığı)

Kaynak: [royerlab/kaggle-cell-tracking-competition/metrics.md](https://github.com/royerlab/kaggle-cell-tracking-competition/blob/main/metrics.md)

### 3.1 Edge Jaccard (ana metrik – ağırlık %90)
1. Tahmin edilen node'lar, ground-truth node'lara **optimal ikili eşleştirme (bipartite matching)** ile atanır — maksimum merkez mesafesi **7 µm**.
2. Tahmin kenarları sınıflanır:
   - **TP:** Her iki uç da GT node'a eşleşir ve aralarında GT kenarı var.
   - **FN:** GT kenarı var ama eşleşen tahmin kenarı yok.
   - **FP:** Tahmin kenarı GT bağlantısıyla çelişiyor.
3. `Edge Jaccard = TP / (TP + FP + FN)`

**Fazla tahmin cezası (adjusted):**
```
adjusted_jaccard = max(0, jaccard · (1 − 0.1 · (T_pred − T_true) / T_true))
```
> Yani gereğinden **çok fazla node** üretirseniz ceza yersiniz. "Her voxel'i hücre say" gibi kaba stratejiler işe yaramaz.

### 3.2 Division Jaccard (bölünme metriği – ağırlık %10)
- Bölünme zamanına **±1 kare tolerans** verilir.
- Bir GT bölünmesi TP sayılır: bölünme öncesi eşleşen node + her iki kız hücre soyunda eşleşen node + tek bir bağlı bileşen + iki çıkışlı (bölünen) tahmin node'u.

### 3.3 Nihai skor
```
Final Score = Adjusted Edge Jaccard + 0.1 · Division Jaccard
```
Tüm videolar üzerinde **micro-average** alınır.

> **Stratejik çıkarım:** Puanınızın %90'ı doğru **bağlama (linking)** ediyor. Önce sağlam bir detection + linking hattı kurun; bölünme optimizasyonunu en sona bırakın (sadece %10).

---

## 4. Teknik Yaklaşımlar

> ### 🔴 GÜNCELLEME — YOL A ve YOL B'nin ikisi de KAPALI
> `ultrack` ve `monai` imajda yok, internet kapalı olduğu için kurulamıyorlar; ayrıca GPU yok.
> **Geçerli plan: aşağıdaki YOL C.** YOL A/B tarihsel referans olarak bırakıldı.

### YOL C — Klasik, sıfır ek bağımlılık ⭐ GEÇERLİ PLAN

İmajda hazır olan paketlerle **tam bir pipeline** kurulabiliyor:

| Aşama | Araç | Parametre (EDA'dan) |
|---|---|---|
| Normalizasyon | zarr attrs `image_statistics.quantiles` | hesap gerekmez |
| Detection | `skimage` + `scipy.ndimage.maximum_filter` | anizotropik local-max, bastırma ~5 µm = `(3,12,12)` voxel |
| Hedef yoğunluk | — | **`T_pred ≈ 200/kare`** (T_true ~213) |
| Linking | `scipy.optimize.linear_sum_assignment` | gate **8 µm**, gap-closing kapalı |
| Graf → CSV | `networkx` + `pandas` | `submission.csv` |

EDA bunun işe yarayacağını söylüyor: hareket 2–3 µm medyan, ~%95'i 7 µm altında → Hungarian
linking edge'lerin çoğunu doğru bağlar ve skorun **%90'ı edge**. Bölünme (%10) sona bırakılır.

**İleride:** `pulp`/`cvxpy` var → istenirse Ultrack'in ILP mantığı elle kurulabilir.
Ultrack'i utility dataset'e paketlemek (`higra`, `edt` wheel'leri) mümkün ama baseline
skoru görülmeden yatırım yapılmamalı.

---

### YOL A — Ultrack tabanlı ❌ (kapalı: `ultrack` kurulamıyor)
Royer Lab'in kendi aracı **Ultrack**, tam da bu veri için tasarlandı. En hızlı yüksek skoru bununla alırsınız.

**Boru hattı (pipeline):**
1. **Ön işleme:** Quantile normalizasyon (veri yükleyicide mevcut), gürültü azaltma.
2. **Foreground + kontur tespiti:** Klasik görüntü işleme veya hazır bir çekirdek segmentasyonu.
3. **Aday segment üretimi:** Ultrack birden çok segmentasyon hipotezi üretir (crowded doku için ideal).
4. **Linking + çözüm:** Ultrack, takibi bir **optimizasyon (integer linear programming)** problemi olarak çözer.
5. Çıktıyı GEFF grafına çevir → CSV → gönder.

**Kütüphaneler:** [`ultrack`](https://github.com/royerlab/ultrack), `napari` (3B görselleştirme + hata ayıklama), `tracksdata`.

### YOL B — Derin öğrenme ❌ (kapalı: `monai` yok, GPU yok, `torch` CPU-only)
Organizatörlerin önerdiği mimari:

1. **Detection:** **Temporal attention'lı 3B U-Net** → voxel bazında özellik + tespit olasılık haritası. **Local-maximum suppression** ile merkez koordinatları çıkarılır.
2. **Linking:** Tespit merkezlerinde havuzlanan özellikler → **cross-attention transformer** tüm `(t, t+1)` node çiftlerini skorlar.
3. **Eğitim:** ⚠️ Yalnızca **ground-truth kenarları** backprop'ta kullanılır; arka plan tespitleri ve etiketsiz hücreler yok sayılır (seyrek etiket problemi).

**Kütüphaneler:** PyTorch, `tracksdata`, `zarr`, `polars`, `scipy`, `napari`, `monai` (3B tıbbi görüntü için hazır U-Net'ler).

### Önerilen strateji
> ~~Önce YOL A ile leaderboard'a bir skor koyun~~ → **YOL C ile leaderboard'a bir skor koyun.**
> Ultrack/DL yolları ortam kısıtlarıyla kapalı. Skor geldikten sonra iyileştirme sırası:
> (1) detection eşiği/`T_pred` kalibrasyonu, (2) linking maliyet fonksiyonu,
> (3) bölünme, (4) gerekirse Ultrack'i utility dataset'e paketleme.

---

## 5. Haftalık Yol Haritası

> **Çalışma ortamı:** Tüm deneyler **Kaggle notebook**'ta koşar (yerelde çalıştırmıyoruz). Veri `/kaggle/input/...` altında bağlı gelir; bu repo notebook + `src/` + config versiyonlaması içindir. `src/` modüllerini bir **Kaggle utility dataset** olarak yükleyip notebook'a ekleyin.

### Hafta 1 — Kurulum & Veriyi Anlama
- [ ] Kaggle yarışmasına takım olarak katıl, kuralları oku (özellikle harici veri / GPU kotası).
- [ ] `notebooks/01_eda.ipynb`'i Kaggle'a yükle; `open_dataset()` ile bir train örneğini yükle.
- [ ] Görüntüyü ve ground-truth track'leri incele (Kaggle'da napari başsız; slice görsellerini kaydedip indir veya notebook içi 2B projeksiyonla bak). Verinin nasıl göründüğünü gözünle gör.
- [ ] `(T, Z, Y, X)` boyutlarını, ölçekleri, seyrek etiket yoğunluğunu not al.
- [ ] Metrik kodunu (`metrics.md` + eval) notebook'ta çalıştır; **ground-truth'u kendisine verip 1.0 aldığını doğrula** (sanity check).

### Hafta 2 — İlk Gönderim (Baseline) — **YOL C**
- [ ] `02_baseline_classical.ipynb`: detection (local-max) + linking (Hungarian 8 µm) → `submission.csv`.
- [ ] **Internet KAPALI** olduğunu doğrula; test isimlerinin glob'landığını doğrula.
- [ ] `T_pred ≈ 200/kare` olacak şekilde eşiği kalibre et (fazla-tahmin cezası).
- [ ] **İlk CSV'yi gönder** — sıralamada bir sayı görmek moral ve referans verir.
- [ ] Çalışma süresini ölç; gizli test daha büyük olabilir (bütçe payı bırak).
- [ ] ~~Ultrack'i test veri setine uygula~~ (kapalı).

### Hafta 3–4 — Detection Kalitesini Yükselt
- [ ] Detection'ı ölç: kaç GT node'u 7 µm içinde yakalayabiliyorsun? (recall).
- [ ] 3B U-Net (MONAI) ile çekirdek merkez heatmap regresyonu dene.
- [ ] Anizotropiyi doğru işle: konvolüsyon kernel'lerinde veya voxel yeniden örneklemede Z ölçeğini dikkate al.
- [ ] Fazla-tahmin cezasını izle: `T_pred ≈ T_true` olacak şekilde eşik ayarla.

### Hafta 5–6 — Linking & Bölünmeler
- [ ] Basit linking (nearest-neighbor / Hungarian) → gelişmiş (Ultrack ILP veya transformer).
- [ ] Hareket modeli ekle (hücreler kareler arası küçük, Brownian benzeri hareket eder).
- [ ] Bölünme tespitini ekle (±1 kare tolerans; iki-çıkışlı node mantığı). Son %10 için.
- [ ] Cross-validation kur; leaderboard'a değil **kendi yerel skoruna** güven (overfit'i önle).

### Hafta 7+ — İyileştirme & Ensemble
- [ ] Hiperparametre taraması (mesafe eşiği, NMS yarıçapı, linking maliyeti).
- [ ] Hibrit: DL detection + Ultrack linking.
- [ ] Ensemble / test-time augmentation.
- [ ] Gönderim hattını otomatikleştir, hataları napari'de görselleştirerek (TP/FP/FN renkli) ayıkla.

---

## 6. Kullanılacak Araçlar — Özet Tablo

| Amaç | Araç | Not |
|------|------|-----|
| Veri okuma | **`zarr` 3.2.1** | OME-Zarr v3; chunk = 1 zaman karesi |
| GT okuma | **düz `zarr`** | `geff`/`tracksdata` YOK — gerek de yok |
| Detection | **`skimage` + `scipy.ndimage`** | anizotropik local-max |
| Linking | **`scipy.optimize.linear_sum_assignment`** | Hungarian, 8 µm gate |
| Graf işlemleri | **`networkx`** | node/edge/bölünme |
| Çıktı | **`pandas`** | `submission.csv` |
| ~~Takip (klasik)~~ | ~~Ultrack~~ | ❌ kurulamıyor |
| ~~Detection (DL)~~ | ~~PyTorch + MONAI~~ | ❌ monai yok, GPU yok |
| ~~Görselleştirme~~ | ~~napari~~ | ❌ yok; matplotlib 2B projeksiyon kullan |
| Kod paylaşımı | Kaggle **utility dataset** | `src/`'yi dataset olarak yükle (internet gerekmez) |
| Hesaplama | **Kaggle notebook (CPU)** | Internet KAPALI zorunlu; süre limitine dikkat |

---

## 7. Sık Yapılan Hatalar / Dikkat Noktaları

1. **Anizotropi (Z ≠ X,Y):** Mesafeleri µm cinsinden hesapla, piksel değil. 7 µm toleransı buna göre.
2. **Seyrek ground-truth:** Etiketsiz bir hücreyi "yanlış" sayma; eğitimde sadece etiketli kenarlarla backprop yap. Metrik de sadece etiketli node'lara bakar.
3. **Fazla tahmin cezası:** Node sayısını GT'ye yakın tut; "her şeyi tespit et" stratejisi ceza yer.
4. **Bellek:** 3B + zaman = devasa tensörler. Chunk/patch tabanlı işle, tüm hacmi RAM'e yükleme.
5. **Skor ağırlığı:** %90 edge, %10 division. Önce bağlamayı sağlamlaştır.
6. **Yerel metrik doğrulaması:** Kaggle'a gönderi kotası sınırlı. Yerel eval'i birebir kur, sanity check yap (GT→GT = 1.0).
7. **Leaderboard'a overfit etme:** Public LB küçük olabilir; kendi CV skorlarına güven.

---

## 8. Öncelikli Kaynaklar

- **Metrik dokümanı:** [metrics.md](https://github.com/royerlab/kaggle-cell-tracking-competition/blob/main/metrics.md)
- **Resmi yarışma reposu:** [royerlab/kaggle-cell-tracking-competition](https://github.com/royerlab/kaggle-cell-tracking-competition)
- **Ultrack (yazılım + doküman):** [github.com/royerlab/ultrack](https://github.com/royerlab/ultrack) · [napari plugin rehberi](https://royerlab.github.io/ultrack/napari.html)
- **Ultrack makalesi (Nature Methods 2025):** [nature.com/articles/s41592-025-02778-0](https://www.nature.com/articles/s41592-025-02778-0)
- **Zebrahub (veri kaynağı):** [biohub.org/blog/zebrahub](https://biohub.org/blog/zebrahub-tracks-zebrafish-development/) · [Cell makalesi](https://www.cell.com/cell/fulltext/S0092-8674(24)01147-4)
- **Baseline notebook'lar:**
  - [Nearest Neighbor (getting started)](https://www.kaggle.com/code/inversion/cell-tracking-getting-started-w-nearest-neighbor)
  - [Classical Baseline](https://www.kaggle.com/code/xiaoleilian/biohub-cell-tracking-classical-baseline)
- **Forum duyurusu:** [image.sc forum](https://forum.image.sc/t/biohub-cell-tracking-during-development-kaggle-competition/121671)
- **Metrik teorisi (opsiyonel):** [CHOTA metric (arXiv)](https://arxiv.org/pdf/2408.11571) — cell tracking metriklerini derinlemesine anlamak için.

---

## 9. Takım Görev Dağılımı Önerisi (2 kişi)

| Kişi | Odak |
|------|------|
| **1. Kişi** | Detection hattı: 3B U-Net / segmentasyon, veri yükleme, normalizasyon, anizotropi |
| **2. Kişi** | Linking + metrik: Ultrack/transformer, bölünme mantığı, yerel değerlendirme, gönderim otomasyonu |
| **Ortak** | napari ile hata ayıklama, CV stratejisi, ensemble kararları |

> İki kişi de **1. hafta** boyunca birlikte veriyi napari'de incelesin ve metriği yerelde çalıştırsın — ortak zemin şart.

---

### TL;DR
> Zebra balığı embriyosunda 3B+zaman mikroskop videolarında hücreleri **tespit et** ve **soy ağacını** çıkar. Skor = %90 doğru bağlama + %10 bölünme (7 µm eşleşme toleransı, anizotropik voxel).
>
> **Bu bir code competition: internet YOK, GPU YOK, test verisi runtime'da değişir.** Ultrack ve MONAI kurulamıyor → **YOL C**: `skimage` local-max detection + `scipy` Hungarian linking (8 µm). En büyük tuzaklar: anizotropi (µm kullan), test isimlerini hardcode etmek, ve fazla-tahmin cezası (**`T_pred ≈ 200/kare`** hedefle).
