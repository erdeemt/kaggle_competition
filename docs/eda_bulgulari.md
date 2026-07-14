# EDA Bulguları — Biohub Cell Tracking (Experiment: `bet_exp/eda`)

İlk keşif turu. Kaynak: [notebooks/01_eda.ipynb](../notebooks/01_eda.ipynb) (Kaggle'da koştu),
özet veri: [outputs/figures/eda_summary.csv](../outputs/figures/eda_summary.csv) (199 train örneği).

> Detay figürler (MIP, overlay, hareket, yoğunluk) `.gitignore` gereği repoda tutulmuyor;
> notebook yeniden çalıştırılınca `/kaggle/working` altında üretiliyor.

---

## 1. Veri geometrisi — tek tip

- **Gerçek yol doğrulandı:** `/kaggle/input/competitions/biohub-cell-tracking-during-development/{train,test}/`
- **Bütün 199 train hacmi aynı boyutta:** `(T, Z, Y, X) = (100, 64, 256, 256)`, `uint16`.
- Fiziksel boyut (anizotropik voxel Z=1.625, Y=X=0.40625 µm):
  - Z ≈ 64 × 1.625 = **104 µm**, Y = X ≈ 256 × 0.40625 = **104 µm** → ~104³ µm'lik küp, 100 kare.
- **Çıkarım:** Tek-tip şekil batching / patch mantığını basitleştirir. Z ekseni 4× kaba
  (64 dilim vs 256) → mesafeleri **µm** cinsinden hesapla, konvolüsyonda/patch'te anizotropiyi hesaba kat.

## 2. İki ayrı domain (embriyo/koşul): `44b6` vs `6bba`

| Grup | Örnek | Medyan node | Medyan node/kare | Toplam bölünme |
|------|-------|-------------|------------------|----------------|
| `44b6` | 71 | 214 | **2.1** | 26 |
| `6bba` | 128 | 826 | **8.4** | 125 |
| **Tümü** | 199 | 659 | 6.6 | 151 |

- İki grup **etiket yoğunluğunda ~4× farklı**. `44b6` çok seyrek (bazıları kare başına 1 node =
  neredeyse tek bir soy takip edilmiş), `6bba` daha yoğun.
- **Çıkarım — CV kritik:** Cross-validation'ı **prefix'e göre** böl (bir grubu tümüyle validation'a
  ayırma leakage'ı önler) ve **her iki domaini de** hem train hem val'de temsil et. Yol haritasındaki
  test/train isim çakışması (aynı filmin farklı pencereleri) da bu yüzden dikkat gerektiriyor.

## 3. Seyrek ground-truth — DOĞRULANDI (en kritik bulgu)

- Toplam **133 318 node**, **128 883 edge** / (199 × 100 = 19 900 kare-örneği) → ortalama **6.6 etiketli node/kare**.
- Ama MIP ve overlay görselleri kare başına **yüzlerce** çekirdek gösteriyor. `overlay_..._t0.png`'de
  kalabalık alanda **yalnızca 1 node** etiketli. Yani görünür hücrelerin çok küçük bir kısmı etiketli.
- Zamansal olarak da seyrek: örnek `44b6_0113de3b`'de kareler 0–2 etiketli, 3–25 boş, 26–76 etiketli,
  sonrası boş (`gt_density` figürü).
- **Çıkarım:**
  1. Eğitimde **yalnızca etiketli kenarlarla** backprop; etiketsiz hücreleri "negatif" sayma.
  2. Metrik yalnızca etiketli node'lara bakıyor → **fazla-tahmin cezası** gerçek risk. "Her şeyi tespit et"
     stratejisi `T_pred >> T_true` yapıp puanı düşürür. Detection eşiğini GT yoğunluğuna göre ayarla.

## 4. Bölünmeler çok nadir

- Toplam **151 bölünme / 133 318 node = %0.11**. Örneklerin **112/199'unda hiç bölünme yok**; max 5.
- **Çıkarım:** Skorun zaten %10'u olan division'ı **en sona bırak**. ±1 kare toleransı + nadirlik →
  erken optimizasyon getirisi düşük. Önce edge Jaccard (%90).

## 5. Kare-arası hareket küçük — linking kolaylaşıyor

- `motion` figürü: yer değiştirme ağırlıkla **2–4 µm**, neredeyse tamamı **7 µm toleransının altında**
  (tek bir uç ~7.7 µm).
- **Çıkarım:** 7 µm'lik gate ile **nearest-neighbor / Hungarian** linking edge'lerin büyük kısmını
  doğru bağlar. Yani %90'lık edge skoru **iyi detection + basit linking** ile büyük ölçüde erişilebilir.
  Karmaşık transformer linking'e koşmadan önce baseline'ı bununla kur (YOL A).

## 6. Görsel gözlemler

- **Çekirdekler** elipsoid, sıkı paketli; kalabalık doku. XZ/YZ MIP'leri Z boyunca belirgin bulanık
  (anizotropi görsel olarak da net).
- **Track yapısı:** seyrek GT nedeniyle bağlı bileşenler ayrık, seyrek örneklenmiş bireysel soylar
  (örnekte ~48 node'luk uzun bir track + ~3 node'luk kısa bir track).

---

## Sonraki experiment için yapılacaklar (EDA genişletme)

- [ ] **Tüm datasetlerde** hareket/seyreklik dağılımını topla (şu an figürler tek örnekten).
- [ ] Etiketli-hücre / toplam-görünür-hücre oranını tahmin et (basit blob sayımı ile) → fazla-tahmin bütçesi.
- [ ] Yoğunluk normalizasyonu (quantile) öncesi/sonrası karşılaştır; `44b6` vs `6bba` yoğunluk profili.
- [ ] Bölünme anlarını görselleştir (nadir ama %10 skor).
- [ ] Kare-arası hareketi Z ve XY bileşenlerine ayır (anizotropik hareket var mı?).
- [ ] Prefix-bazlı CV foldlarını sabitle ve kaydet.
- [ ] `05_evaluation` sanity check: metrik koduna GT→GT verip **1.0** aldığını doğrula.
