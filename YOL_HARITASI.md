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

## 2. Veri Yapısı (Çok Önemli — İlk Gün Anlaşılmalı)

```
/kaggle/input/competitions/biohub-cell-tracking-during-development/
├── train/    → eşleşmiş .zarr (görüntü) + .geff (ground-truth track grafı)
└── test/     → sadece .zarr (görüntü) — tahminlerinizi bunun için üreteceksiniz
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

## 4. Önerilen Teknik Yaklaşımlar

İki ana yol var. İkisini de aşamalı denemenizi öneririz.

### YOL A — Klasik / Ultrack tabanlı (hızlı başlangıç, güçlü baseline) ⭐ ÖNCE BU
Royer Lab'in kendi aracı **Ultrack**, tam da bu veri için tasarlandı. En hızlı yüksek skoru bununla alırsınız.

**Boru hattı (pipeline):**
1. **Ön işleme:** Quantile normalizasyon (veri yükleyicide mevcut), gürültü azaltma.
2. **Foreground + kontur tespiti:** Klasik görüntü işleme veya hazır bir çekirdek segmentasyonu.
3. **Aday segment üretimi:** Ultrack birden çok segmentasyon hipotezi üretir (crowded doku için ideal).
4. **Linking + çözüm:** Ultrack, takibi bir **optimizasyon (integer linear programming)** problemi olarak çözer.
5. Çıktıyı GEFF grafına çevir → CSV → gönder.

**Kütüphaneler:** [`ultrack`](https://github.com/royerlab/ultrack), `napari` (3B görselleştirme + hata ayıklama), `tracksdata`.

### YOL B — Derin öğrenme (yarışma için organizatör baseline'ı)
Organizatörlerin önerdiği mimari:

1. **Detection:** **Temporal attention'lı 3B U-Net** → voxel bazında özellik + tespit olasılık haritası. **Local-maximum suppression** ile merkez koordinatları çıkarılır.
2. **Linking:** Tespit merkezlerinde havuzlanan özellikler → **cross-attention transformer** tüm `(t, t+1)` node çiftlerini skorlar.
3. **Eğitim:** ⚠️ Yalnızca **ground-truth kenarları** backprop'ta kullanılır; arka plan tespitleri ve etiketsiz hücreler yok sayılır (seyrek etiket problemi).

**Kütüphaneler:** PyTorch, `tracksdata`, `zarr`, `polars`, `scipy`, `napari`, `monai` (3B tıbbi görüntü için hazır U-Net'ler).

### Önerilen strateji
> **Önce YOL A ile leaderboard'a bir skor koyun** (1. hafta). Ardından YOL B ile detection kalitesini artırın veya YOL A'nın segmentasyon adımını bir sinir ağıyla besleyin (**hibrit**: DL detection + Ultrack linking — genellikle en iyi sonuç bu).

---

## 5. Haftalık Yol Haritası

> **Çalışma ortamı:** Tüm deneyler **Kaggle notebook**'ta koşar (yerelde çalıştırmıyoruz). Veri `/kaggle/input/...` altında bağlı gelir; bu repo notebook + `src/` + config versiyonlaması içindir. `src/` modüllerini bir **Kaggle utility dataset** olarak yükleyip notebook'a ekleyin.

### Hafta 1 — Kurulum & Veriyi Anlama
- [ ] Kaggle yarışmasına takım olarak katıl, kuralları oku (özellikle harici veri / GPU kotası).
- [ ] `notebooks/01_eda.ipynb`'i Kaggle'a yükle; `open_dataset()` ile bir train örneğini yükle.
- [ ] Görüntüyü ve ground-truth track'leri incele (Kaggle'da napari başsız; slice görsellerini kaydedip indir veya notebook içi 2B projeksiyonla bak). Verinin nasıl göründüğünü gözünle gör.
- [ ] `(T, Z, Y, X)` boyutlarını, ölçekleri, seyrek etiket yoğunluğunu not al.
- [ ] Metrik kodunu (`metrics.md` + eval) notebook'ta çalıştır; **ground-truth'u kendisine verip 1.0 aldığını doğrula** (sanity check).

### Hafta 2 — İlk Gönderim (Baseline)
- [ ] Getting-started notebook'unu çalıştır: [Nearest Neighbor baseline](https://www.kaggle.com/code/inversion/cell-tracking-getting-started-w-nearest-neighbor).
- [ ] [Classical Baseline](https://www.kaggle.com/code/xiaoleilian/biohub-cell-tracking-classical-baseline) notebook'unu incele/çalıştır.
- [ ] **İlk CSV'yi gönder** — sıralamada bir sayı görmek moral ve referans verir.
- [ ] Ultrack'i "automatic tracking from image" modunda test veri setine uygula.

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
| Veri okuma | `zarr`, `tracksdata`, `open_dataset()` | OME-Zarr chunk okuma |
| Takip (klasik) | **Ultrack** | Bu veri için tasarlandı — önce bunu dene |
| Detection (DL) | **PyTorch + MONAI** | 3B U-Net, temporal attention |
| Linking (DL) | Cross-attention transformer | Organizatör baseline'ı |
| Graf işlemleri | `tracksdata` (`InMemoryGraph`) | Node/edge/bölünme |
| Sayısal işlem | `scipy`, `numpy`, `polars` | LMS, Hungarian, I/O |
| Görselleştirme | **napari** | 3B hata ayıklama (Kaggle'da başsız çalışır; görseli indirip incele) |
| Kod paylaşımı | Kaggle **utility dataset** | `src/`'yi dataset olarak yükle, notebook'a import et |
| Hesaplama | **Kaggle GPU notebook** | Tüm deneyler Kaggle'da; haftalık GPU kotasına dikkat |

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
> Zebra balığı embriyosunda 3B+zaman mikroskop videolarında hücreleri **tespit et** ve **soy ağacını** çıkar. Skor = %90 doğru bağlama + %10 bölünme (7 µm eşleşme toleransı, anizotropik voxel). **Önce Ultrack ile baseline kur, sonra derin öğrenme detection'ı ile geliştir.** En büyük tuzaklar: anizotropi (µm kullan), seyrek etiket, ve fazla-tahmin cezası.
