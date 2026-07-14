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
- [ ] `sample_submission.csv` kolonları/formatı ← gönderim şeması için şart
- [ ] `axes meta` (birim etiketi)

---

# Detaylı EDA (`01b_eda_detailed`) — dataset geneli

Figürler: `outputs/figures/D02`–`D10`. Analizler artık **tek örnek değil, tüm train**.

## A. Yoğunluk & yapı ([D02](../outputs/figures/D02_gt_density.png), [D03](../outputs/figures/D03_lineage.png))
- **Kare başına ~5–6 etiketli node** (tipik 2–15, max ~30) — ilk örnek (1/kare) atipikmiş.
- **Örnek başına ~15–25 soy** (max ~80); node ≈ soy × ~25.
- Soy uzunluğu/süresi: çoğu 10–50 kare, **t=0→99 tam kapsayan** belirgin bir grup var.
- **Zamansal boşluk YOK:** uzunluk ≈ süre (veya bölünmeyle üstü). → **gap-closing gerekmez.**

## B. Linking — havuzlanmış hareket ([D04](../outputs/figures/D04_motion_pooled.png), [D05](../outputs/figures/D05_persistence.png))
- |yer değiştirme|: medyan ~2–3 µm, **95p ~5.5 µm, 99p ~8 µm, ~%95 < 7 µm**. İnce kuyruk 15–60 µm (nadir; hızlı hücre/olası bölünme).
- → **`max_distance ≈ 8 µm`** (95–99p'yi kapsar).
- Yön kalıcılığı **kısmi**: hızlı hücreler düz gider (cos→+1), yavaşlar jitter yapar. Hareket modeli **marjinal** fayda; NN-linking yeterli.

## C. Eşleşme belirsizliği — kalabalıklık ([D06](../outputs/figures/D06_crowding.png)) ✅
- Kare-içi **GT-GT en yakın komşu medyan ~25 µm**; **<7 µm sadece ~%1–2**.
- → **Metrik eşleşmesi neredeyse belirsizliksiz**; 7 µm tolerans güvenli. (Çünkü GT seyrek/yayılı; her çekirdek değil.)

## D. Detection ([D09](../outputs/figures/D09_detection.png))
- Çekirdek ~**10 µm çap** (yarı-maks yarıçap ~5 µm), her Z derinliğinde GT var ([D10](../outputs/figures/D10_zbehavior.png)).
- Global foreground sinyali güçlü ([07b], ~8×) **ama yerel komşu-kontrastı düşük (~1.5× SNR)** — çekirdekler paketli/temas halinde.
  → **Zor kısım instance ayrımı**, foreground değil. Tam da **Ultrack'in çoklu-hipotez segmentasyonunun** çözdüğü problem.
- **T_true tahmini (düzeltilmiş sayaç):** ~**213 çekirdek/kare** (medyan; örnekler arası ~50–730 geniş dağılım).
  - Örnek başına toplam gerçek hücre ≈ **~21.000** (213 × 100 kare).
  - **GT etiketli oran ~%2–3** (kare başına ~5–6 etiketli / ~213 gerçek). Yani takip edilecek gerçek hücre sayısı çok yüksek, GT bunun küçük bir örneklemi.
  - **Fazla-tahmin cezası:** `T_pred ≈ 200/kare` hedefle; binlerce gürültü-tespiti cezalandırılır.

## E. Bölünme ([D07](../outputs/figures/D07_division.png))
- ~120 bölünme; zamanda yayvan. **ebeveyn-kız ~6 µm**, kız-kız ~11 µm.
- ebeveyn-kız < linking radius → **aynı ~8 µm radius bölünme kenarlarını da yakalar.** Biraz büyük radius bölünmeleri kaçırmaz.

## F. Track doğuş/ölüm ([D08](../outputs/figures/D08_endpoints.png))
- Çoğu track **t0→t99 tam kapsam**; mid-movie başlangıç/bitiş de var.
- **Bitişler alan kenarında yoğun** → hücreler görüş alanından çıkıyor. → tracker'da **appearance/disappearance maliyeti** gerekli.

---

## Güncellenmiş baseline kararları (Ultrack)
1. **Ölçek:** anizotropik `(1.625, 0.40625, 0.40625)` ver (mesafeler µm).
2. **Detection:** foreground eşiği düşük (tissue parlak) **ama instance için contour/çoklu-hipotez** kullan (asıl zorluk ayrım).
3. **Linking:** `max_distance ≈ 8 µm`; **gap-closing kapalı** (boşluk yok); **appearance/disappearance açık**; **division açık** (ebeveyn-kız ~6 µm).
4. **Fazla-tahmin cezası:** `T_pred ≈ 200 çekirdek/kare` hedefle (T_true ~213/kare). Gürültü-tespitlerini eşikle; binlerce yanlış node cezalandırılır.
5. **Öncelik:** edge %90 → linking; division %10 sonra.
