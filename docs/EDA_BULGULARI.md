# EDA Bulguları — Biohub Cell Tracking

`notebooks/01_eda.ipynb` çıktılarının analizi. Görseller: [`outputs/figures/`](../outputs/figures/).
Derin inceleme örneği: `44b6_0113de3b` (seyrek, 2 soylu bir örnek); dataset geneli ise
tüm train örnekleri üzerinden.

---

## 1. Dataset geneli ([02_dataset_distributions](../outputs/figures/02_dataset_distributions.png))

| Özellik | Bulgu | Sonuç |
|---|---|---|
| **T (zaman)** | **Tüm örneklerde tam 100 kare** | Uniform; sabit zaman ekseni varsayabiliriz |
| **Y, X** | Hepsi **256 × 256 piksel** | Küçük, sabit alan → tüm hacim RAM'e sığar |
| **Z** | ~62–63 dilim (derinlik 0–100 µm / 1.625) | Anizotropik, ince Z |
| **node / örnek** | ~0–2000 (çoğu 200–1500) | GT yoğunluğu örnekten örneğe çok değişir |
| **soy (track) / örnek** | Çoğu **1–10**, kuyruk ~80'e | GT = az sayıda **tam izlenmiş hücre soyu** |
| **bölünme / örnek** | Çoğu **0–2** (birçok örnekte 0), max ~5 | Bölünmeler **nadir** |

> **Ana çıkarım:** "Seyrek ground-truth", embriyonun tamamının değil, video başına
> **birkaç hücre soyunun** baştan sona etiketlenmesi demek. Metrik yalnızca bu etiketli
> node'lara 7 µm içinde eşleşen tahminlere bakar.

---

## 2. Koordinat birimi: **VOXEL (piksel)** ✅ ([05_gt_spatial](../outputs/figures/05_gt_spatial.png))

- GT koordinat aralığı: x ∈ [72, 255], y ∈ [85, 230], z ∈ [0, 63] → **görüntü boyutlarıyla birebir** (X,Y=256, Z≈63).
- µm olsaydı x_max ≈ 104 (256·0.406) olurdu. Değil → **`COORDS_UM = False`**.
- **Linking'de mesafeler `voxel × (1.625, 0.406, 0.406)` ile µm'ye çevrilecek.** Metriğin 7 µm toleransı bu birimde.

---

## 3. Hareket — linking parametresi ([06_motion](../outputs/figures/06_motion.png))

- Kareler-arası |yer değiştirme|: **medyan ~2–4 µm, neredeyse tamamı < 7 µm** (max ~8 µm).
- **Metriğin 7 µm eşleşme toleransı gerçek hücre hareketinin tam üst sınırında.**
  → **Linking arama yarıçapı ≈ 7–8 µm** tüm gerçek bağlantıları yakalar; daha fazlası gereksiz FP riski.
- Hareket **anizotropik**: µm cinsinden |dz| bileşeni |dx|,|dy|'den belirgin büyük (hücre Z boyunca göç ediyor — [05_gt_spatial] XZ paneli z 0→63).

---

## 4. Detection sinyali — **çok güçlü** ✅ ([07b_gt_vs_random](../outputs/figures/07b_gt_vs_random_intensity.png))

- GT çekirdek merkezlerinde yoğunluk **medyan ~1750**; rastgele voxel **medyan ~200** (çoğu < 500).
- Çekirdekler parlak, arka plandan **temiz ayrışıyor** ([03b_mip](../outputs/figures/03b_mip_xyz.png), [07_patches](../outputs/figures/07_nucleus_patches.png)): elipsoid, ~15–25 px (~6–10 µm) blob'lar.
- **Sonuç:** Basit yoğunluk eşiği + local-maxima / blob detection bile iyi aday merkez üretir. Ultrack'in klasik foreground modu için ideal.

---

## 5. Görüntü özellikleri ([03_intensity](../outputs/figures/03_intensity_profiles.png))

- Yoğunluk 0–~3000 (uint16), arka plan ~100–200, çekirdekler 1500+.
- **Z-derinliği:** yüzeyde (z=0) düşük, orta-derin bölgede (z≈45–90 µm) tepe, uçta düşüş → doku orta Z'de; uçlarda sinyal zayıf. Derin Z'de detection biraz daha zor.
- **Zaman:** ortalama yoğunluk t≈30'da tepe, sona doğru düşüş (gelişim + olası photobleaching).

---

## Hafta 2 için somut kararlar

**Yaklaşım:** YOL A — Ultrack baseline (güçlü sinyal + küçük hacim buna çok uygun).

1. **Ön işleme:** quantile normalizasyon; arka plan ~200 civarı, eşik güvenli.
2. **Detection:** parlak çekirdekler → eşik + local-maxima (veya Ultrack foreground). Blob çapı ~15–25 px.
3. **Anizotropi:** Ultrack'e ölçeği `(z,y,x)=(1.625, 0.40625, 0.40625)` ver ki mesafeler µm olsun.
4. **Linking:** `max_distance ≈ 7 µm` (hareket < 7 µm). Bölünmeler nadir → önce bölünmesiz kur, sonra ekle.
5. **Skor önceliği:** edge %90 → sağlam linking; division %10 → sonra.
6. **Çıktı:** graf → `.geff` → CSV (`geffs_to_csv`), gönder.

## Hâlâ teyit edilecek (metin çıktısı)
- [ ] `shape (T,Z,Y,X)` ve `>> Kullanilacak olcek` (Z sayısı ve ölçek kesin değeri)
- [ ] `sample_submission.csv` kolonları/formatı ← gönderim şeması için şart
- [ ] `axes meta` (birim etiketi)
