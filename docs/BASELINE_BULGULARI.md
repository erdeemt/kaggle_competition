# Baseline Bulguları — `erd_exp/baseline`

Hafta 2. Klasik hat: **blob detection + Hungarian linking**. Ultrack değil — gerekçe aşağıda.

---

## 0. Yarışmanın gerçek yapısı (submit'i belirleyen kısıtlar)

| Gerçek | Sonuç |
|---|---|
| **Code Competition** — submit = notebook | Kaggle, defteri **gizli test setiyle yeniden çalıştırır** |
| Görünen `test/` = **placeholder** (4 dataset, train'den kopya, `.geff` yok) | Dataset isim/sayısı **asla hardcode edilmez**; `test/` dinamik gezilir |
| **İnternet KAPALI zorunlu**, ama `zarr` base imajda **yok** | `zarr`'ı Kaggle Dataset'i (`pip install --target`) ile taşı → `sys.path` |
| Çıktı **`submission.csv`** olmalı | `/kaggle/working/submission.csv` |
| Koordinatlar **tamsayı voxel** | `int(round(...))` |
| **Her test dataseti** submission'da olmalı | assert ile korunuyor |

> Doğrulandı: temiz kernel + internet kapalı → `zarr <- sys.path: .../cell-tracking-libs/pylibs`.

## 1. ⚠️ En kritik ders: metrik **MICRO-average**, macro değil

`metrics.md`: *"Metrics are aggregated via **micro-averaging** across all videos"*
→ TP/FP/FN **videolar arası havuzlanır**, dataset başına jaccard'ın ortalaması **alınmaz**.

Bu, sıralamayı **tersine çeviriyor**:

| konfig | macro (yanıltıcı) | **micro (gerçek)** |
|---|---|---|
| **(1,2,2)\|(3,11,11)\|otsu** ← baseline | 0.682 | **0.781** ✅ |
| (0.5,1,1)\|(3,11,11)\|otsu | 0.686 | 0.744 |
| (0.5,1,1)\|(3,7,7)\|otsu | 0.665 | 0.742 |

Sebep: küçük dataset'ler (44b6: ~50 kenar) macro'da 6bba (845 + 1183 kenar) kadar ağırlık alıyor.
**Micro'da 44b6_0b24845f toplam kenarların yalnızca %2.3'ü** — onu tamamen düzeltmek +0.017 getirir.

## 2. `T_true` uydurma değil — veride yazıyor
`.geff → attrs["geff"]["extra"]["estimated_number_of_nodes"]`

| dataset | T_true | baseline tespit | oran |
|---|---|---|---|
| 44b6_0113de3b | 25.755 | 26.019 | 1.01 |
| 44b6_0b24845f | 32.795 | 34.737 | 1.06 |
| 6bba_05b6850b | 6.362 | 6.002 | 0.94 |
| 6bba_05db0fb1 | 69.800 | 66.152 | 0.95 |

T_true dataset başına **11× değişiyor** (6.362 → 69.800) ve Otsu bunu **kendiliğinden tutturuyor**
→ fazla-tahmin cezası ≈ **sıfır**. Yoğunluğu kovalamaya gerek yok.
⚠️ Test'te `.geff` yok → çıkarımda T_true **okunamaz**; Otsu'nun doğal kalibrasyonuna güveniyoruz.

## 3. Çöken modeller (deneyle çürütüldü)

**`jaccard ≈ recall²` — YANLIŞ.** Yoğunluk artınca linking çöküyor:

| konfig | recall | ham jaccard |
|---|---|---|
| baseline (313/kare) | 0.784 | **0.680** |
| bombardıman `(0.5,1,1)\|(1,5,5)` (3413/kare) | **0.988** | **0.303** |

`44b6_0113de3b`: recall **1.0** ama TP=13/FN=37 (baseline TP=47). Node'lar bulunuyor,
**birbirine bağlanmıyor** — Hungarian, GT'ye eşleşen node'u komşu sahte tespitlere atıyor.
Üstelik cezayla birlikte (12× fazla tahmin) bu konfig **0** alırdı.

> **Recall optimize edilecek hedef DEĞİL.** 7 µm tolerans, ~11 µm çekirdek aralığına göre geniş:
> 3200 nokta/kare'de dokudaki *herhangi* bir noktanın 7 µm'sinde ~14 tespit düşer → recall şişer.

**Detection tuning tükendi:** 27 konfigin hepsi 0.665–0.686 (macro). Parametre uzayında kazanç yok.

## 4. Baseline sonucu

**Yerel micro Edge Jaccard = 0.781** (ceza çarpanı 1.001 → adj ≈ 0.781)

```
TP=1659  FP=1  FN=468
FN dağılımı:  6bba_05db0fb1: 295 (%63) | 6bba_05b6850b: 130 (%28)
              44b6_0b24845f:  40 (%9) | 44b6_0113de3b: 3
```
**Kaybın %91'i iki büyük 6bba dataset'inde.**

`Final = adj_edge_jaccard + 0.1 × division_jaccard` → bölünme tahmin etmiyoruz ⇒ **+0.1 masada**.

## 5. Sıradaki öncelikler (micro'ya göre)
1. **Bölünme ekle** — +0.1 tavan, şu an 0. Hungarian 1-1 olduğu için hiç üretmiyoruz.
   EDA: ebeveyn-kız ~6 µm (linking yarıçapımızın içinde) → ikinci eşleştirme turu yeterli.
2. **6bba FN'lerini azalt** — 468 kaybın %91'i orada.
3. ~~44b6_0b24845f~~ — micro'da %2.3, **bırak**. (Not: GT→tespit ofseti dy=+10.8 voxel;
   test/train görüntüleri birebir aynı, yani "farklı pencere" değil.)
4. **Linking kalitesi** — yoğunlukla bozuluyor; asıl darboğaz burada olabilir.

## 6. Metodolojik dersler
- **Sadece gerçek metrik sayar.** `recall`, `recall²×ceza` gibi vekil ölçütler **iki kez** yanlış
  sıralama verdi (bombardıman konfigini birinci, gerçekte sonuncuydu).
- **Doğru agregasyon kritik** — macro/micro farkı hem skoru (0.68 vs 0.78) hem sıralamayı değiştirdi.
- Kontrol grubu (mevcut baseline) her karşılaştırmaya dahil edilmeli.
