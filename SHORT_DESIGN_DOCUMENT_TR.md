# Kısa Tasarım Dokümanı — Drone Teşhis Asistanı

**Proje:** Baibars Task — Tarımsal Drone Öngörücü Bakım Sistemi  
**Yazar:** İbrahim Talha Demir  
**Tarih:** 2026-05-02  
**Versiyon:** 1.0  
**Yığın:** Python (veri üretimi) · Dart / Flutter (teşhis motoru + arayüz)  
**İlişkili Repolar:**  
- [baibars_interview_task_python_script](https://github.com/IbrahimTalha0/baibars_interview_task_python_script)  
- [baibars_task_mobile_app](https://github.com/IbrahimTalha0/baibars_task_mobile_app)

---

## İçindekiler

1. [Problem Modellemesi](#1-problem-modellemesi)
   - [1.1 Veri Yapısı ve Üretim Katmanı (Python)](#11-veri-yapısı-ve-üretim-katmanı-python)
   - [1.2 Teşhis Motoru Mimarisi (Dart)](#12-teşhis-motoru-mimarisi-dart)
   - [1.3 Arayüz Katmanı (Flutter)](#13-arayüz-katmanı-flutter)
2. [Temel Varsayımlar](#2-temel-varsayımlar)
   - [2.1 Termal Eşikler (model bazında)](#21-termal-eşikler-model-bazında)
   - [2.2 Elektriksel Eşikler](#22-elektriksel-eşikler)
   - [2.3 Otonom Eşikler](#23-otonom-eşikler)
   - [2.4 Bağlamsal Gevşetme Kuralları](#24-bağlamsal-gevşetme-kuralları)
   - [2.5 Kronik Tespit Eşikleri](#25-kronik-tespit-eşikleri)
3. [Teşhis Modeli Açıklaması](#3-teşhis-modeli-açıklaması)
   - [3.1 Yaklaşım](#31-yaklaşım)
   - [3.2 Token Optimizasyonu](#32-token-optimizasyonu)
   - [3.3 Neler Kırılabilir veya Yanlış Sonuç Üretebilir](#33-neler-kırılabilir-veya-yanlış-sonuç-üretebilir)
   - [3.4 Gerçek Sensör Verisiyle Motor Nasıl İyileşir](#34-gerçek-sensör-verisiyle-motor-nasıl-iyileşir)
4. [Sistem Tasarım Kararları](#4-sistem-tasarım-kararları)
   - [4.1 Eksik Sensör Verisi (NO_DATA)](#41-eksik-sensör-verisi-no_data)
   - [4.2 Test Modları Arasındaki Çelişkili Sinyaller](#42-test-modları-arasındaki-çelişkili-sinyaller)
   - [4.3 Düşük Güven Skoru Olduğunda Ne Olur](#43-düşük-güven-skoru-olduğunda-ne-olur)
5. [Daha Fazla Zaman Olsaydı Neler İyileştirilirdi](#5-daha-fazla-zaman-olsaydı-neler-iyileştirilirdi)
   - [5.1 Gerçek Sensör Verisi Boru Hattı](#51-gerçek-sensör-verisi-boru-hattı)
   - [5.2 Belirsizlik Yönetimi](#52-belirsizlik-yönetimi)
   - [5.3 YZ Yükseltme Mimarisinin Olgunlaştırılması](#53-yz-yükseltme-mimarisinin-olgunlaştırılması)
   - [5.4 Arayüz İyileştirmeleri](#54-arayüz-iyileştirmeleri)
   - [5.5 Filo Seviyesinde Çoklu-Drone Kronik Analizi](#55-filo-seviyesinde-çoklu-drone-kronik-analizi)
6. [Çıkarılan Dersler](#6-çıkarılan-dersler)

---

## Genel Bakış

Bu sistem, teknik uzmanlığı düşük saha teknisyenleri için hızlı ve açıklanabilir drone teşhisi sağlar. Üç boru hattı aşamasını kapsar: **sentetik telemetri üretimi** (Python), **üç katmanlı kural tabanlı teşhis motoru** (Dart) ve **yapılandırılmış sonuç arayüzü** (Flutter). Teşhis mimarisi kasıtlı olarak _yerel-öncelikli bir YZ vekili_ olarak tasarlanmıştır: tam deterministik ilk geçişi yapar ve büyük dil modeline yükseltme gerekip gerekmediğine karar vermeden önce kendi güven puanını hesaplar. Bu yaklaşım, vakaların çoğunda teşhis başına token maliyetini sıfıra yakın tutar.

> Referans: Tam kural kataloğu → [`baibars_task_mobile_app`](https://github.com/IbrahimTalha0/baibars_task_mobile_app) reposundaki `docs_of_core/drone_diagnostic_framework.md`  
> Eşik değerleri → [`baibars_task_mobile_app`](https://github.com/IbrahimTalha0/baibars_task_mobile_app) reposundaki `docs_of_core/thresholds.md`  
> Veri üretimi detayları → [`baibars_interview_task_python_script`](https://github.com/IbrahimTalha0/baibars_interview_task_python_script) reposundaki `en_docs/drone_simulator_technical_review.md` ya da `drone_simulator_teknik_inceleme.md`

---

## 1. Problem Modellemesi

### 1.1 Veri Yapısı ve Üretim Katmanı (Python)

Üretim scripti, motorların eşit düzeyde eskimemesi, aşınan motorun daha fazla akım çekmesi ve daha fazla ısınması gibi fiziksel gerçekler dikkate alınarak kurgulanmıştır. Her motor; kronik eğilimler ve rastgele üretim sapmaları nedeniyle farklı hızlarda yıpranırken, bu yıpranma doğrudan akım tüketimi ve sıcaklık hesaplarına yansıtılır. Özellikle bazı modellerde tek bir motorun kronik olarak daha zayıf olması senaryosu da modellenmiş, böylece sistematik arıza davranışları gerçekçi biçimde simüle edilmiştir. Motorlar arasındaki dengesizlik; titreşim artışı ve stabilite düşüşünü tetikleyen zincirleme bir etki yaratır ve bu ilişkiler birbirine bağlı formüllerle hesaplanır. Ayrıca rulman aşınması gibi tekil problemler tüm sistem davranışını etkileyebilecek şekilde ele alınmıştır. Yük artışıyla birlikte motor üzerindeki stresin doğrusal olmayan biçimde büyümesi, tarımsal drone kullanımındaki gerçek yük profillerine uygun olarak modele dahil edilmiştir.

Script yalnızca motor davranışlarını değil, çevresel ve operasyonel koşulları da gerçekçi biçimde yansıtır. Mevsimsellik sinüzoidal sıcaklık değişimleriyle modellenirken, rüzgar etkisi stokastik dağılımlarla üretilir ve sistemin tüm bileşenlerine etki edecek şekilde hesaplamalara dahil edilir. Batarya tüketimi, verimlilik kaybı ve otonom uçuş metrikleri; yük, rüzgar ve motor aşınmasının birleşik etkileriyle dinamik olarak belirlenir. Ayrıca testler arasında termal hafıza, Newton soğuma yasasına göre korunur ve uçuşlar arası bekleme süreleri gerçek operasyonlara uygun şekilde simüle edilir. Yaşam döngüsü boyunca biriken aşınma, tüm performans metriklerinin zamanla kötüleşmesine neden olurken, ani arıza ve eksik sensör verisi gibi edge-case senaryolar da sisteme dahil edilmiştir. Model bazlı eşik değerleri, operasyonel gerçekleri yansıtacak şekilde farklı drone tiplerine özel tanımlanmış; pilot, müşteri, kullanım geçmişi ve uçuş planlaması gibi operasyonel detaylar da veri bütünlüğünü koruyacak biçimde sabit ve tutarlı tutulmuştur.

---

### 1.2 Teşhis Motoru Mimarisi (Dart)

Teşhis motoru, 3 farklı senaryoda kullanılacak şekilde kurgulanmıştır. Bu kısım, senaryolar ve geliştirilen teşhis motoruyla alakalı bilgiler içerirken, asıl bu mimarinin çalışma şekli ve mantığı 3. [Teşhis Modeli Açıklaması](#3-teşhis-modeli-açıklaması) kısmında yapılmıştır.

```
Seviye 1 — SubTestDiagnosticService
Bu sadece bir tip (tek bir yüklü/otonom/yüksüz test) testin analizini yapmak için kullanılacak servistir.
  Girdi:  tek kaydın tek uçuş modu (UL / LD / AU)
  Çıktı:  tek mod bulguları (motor, elektrik, stabilite, otonom, düşüm)
  Kurallar: ~15 bağımsız + 3 zincir kuralı (Chain 3, 7, 8)

Seviye 2 — TestDiagnosticService
Bu sadece bir testin ( yüklü/otonom/yüksüz testlerin hepsi) analizini yapmak için kullanılacak servistir.
  Girdi:  tam test kaydı (üç modun tamamı)
  Çıktı:  modlar arası örüntü bulguları
  Kurallar: Chain 1, 4, 5, 6 + 4 bağımsız çok-mod örüntüsü

Seviye 3 — DroneDiagnosticService
Bu sadece bir drone'un (bütün ya da seçilen testlerin tamamı) testin analizini yapmak için kullanılacak servistir.
  Girdi:  bir drone'a ait tüm kayıtlar (tarihe göre sıralı)
  Çıktı:  kronik örüntüler, trendler, genel sağlık skoru, risk seviyesi
  Kurallar: Chain 2, 9, 10 + batarya trendi, uçuş saati eşiği,
           sahte iyileşme, aralıklı düşüm, M2–M1 artık, aşınma eğimi
```

Toplam: 3 servis seviyesinde **37 farklı teşhis kuralı**, çerçeve dokümanındaki 28 benzersiz isimli senaryoyu kapsar. Tüm kurallar; `findingId`, `severity`, `category`, `description`, `confidence` (0.0–1.0), `affectedComponents`, `isChainRule` ve `isChronic` alanlarıyla bir `DiagnosticFinding` üretir — arayüzün birebir bağlandığı alanlar bunlardır.

Sağlık skoru formülü, alt sistemleri kritikliğe göre ağırlıklandırır:

| Alt Sistem   | Ağırlık |
| ------------ | ------- |
| Motor        | 40%     |
| Batarya      | 25%     |
| Otonom       | 20%     |
| Konnektörler | 15%     |

Her durum alanı `0` (OK), `1` (WARNING), `2` (NO_DATA) veya `3` (CRITICAL) ceza puanı ekler.  
`Health Score = 100 − (weighted_penalty_sum / max_possible × 100)`, [0, 100] aralığına sıkıştırılır.

| Skor   | Risk Seviyesi | Aksiyon          |
| ------ | ------------- | ---------------- |
| 90–100 | Operasyonel   | Devam et         |
| 70–89  | Düşük Risk    | Yakından izle    |
| 50–69  | Yüksek Risk   | Bakım planla     |
| < 50   | Kritik        | Hemen yere indir |

> Referans: Sağlık skoru formülü → `drone_diagnostic_framework.md` §3  
> Zincir kural detayları → `drone_diagnostic_framework.md` §4

---

### 1.3 Arayüz Katmanı (Flutter)

Flutter arayüzü, açık havada güneş altında kullanıldığında daha rahat görünebilmesi için açık tema olarak herkes tarafından kullanılabilecek kolaylıkta geliştirilmiştir.
Reaktif durum yönetimi için **Riverpod** kullanır. Teşhis boru hattı widget ağacında senkron çağrılır (`DroneDiagnosticService().diagnose(tests)`) — mevcut uygulamada async sınır yoktur çünkü kural motoru I/O içermeyen saf Dart hesaplamasıdır. Bulgular `chronicFindings` ve `nonChronicFindings` olarak ayrılır, önem derecesine göre sıralanır (CRITICAL → WARNING → OK → NO_DATA) ve `DiagnosticFindingCard` bileşenleri olarak gösterilir. `DroneAnalysisPage` üstte sabitlenmiş bir sağlık skoru kartı ve risk rozeti, ardından özet şeridi (kronik sayısı, diğer sayısı, toplam bulgu, kayıt sayısı) ve sonrasında kaydırılabilir bulgu listesini sunar.

Navigasyon akışı: **Home** (model listesi) → **Model Detail** (seri listesi) → **Serial Drone** (test kayıtları) → **Test Analysis** (tek test, üç mod) → **Drone Analysis** (drone-seviyesi kronik/trend bulguları).

> Referans: UI bileşen stilleri → `docs_of_core/ui-styles.md`, `docs_of_core/ui-rules.md`

---

## 2. Temel Varsayımlar

Aşağıdaki eşikler ve kurallar alan bilgisinin yer gerçeği olarak kabul edilir. "Normal operasyonel stres" ile "aksiyon gerektiren arıza" arasındaki çizgiyi kodlarlar.

### 2.1 Termal Eşikler (model bazında)

| Model    | Motor Sıcaklığı UL OK / WARN (°C) | Motor Sıcaklığı LD OK / WARN (°C) | Dengesizlik Tetikleyici |
| -------- | --------------------------------- | --------------------------------- | ----------------------- |
| CT50     | 63 / 78                           | 76 / 90                           | spread > 12°C           |
| CT50-Pro | 63 / 79                           | 76 / 91                           | spread > 12°C           |
| CT75     | 66 / 82                           | 80 / 95                           | spread > 12°C           |
| CT100    | 68 / 84                           | 82 / 98                           | spread > 12°C           |

12°C dengesizlik yayılımı katı eşiktir: dört motorun her biri tek tek OK aralığında olsa bile yayılım > 12°C ise `motor_imbalance_flag = WARNING` olur. Bu, ortalama sıcaklık sınırı aşılmadan önce tek motor rulman arızası başlangıcını yakalar.

### 2.2 Elektriksel Eşikler

- Batarya deşarj oranı: CT50 → 2.5 / 4.0 %/dk · CT100 → 3.2 / 5.0 %/dk (platform ağırlığıyla ölçeklenir)
- Molex konnektör: 46 / 60°C (CT50) — kalıcı CRITICAL yangın riski doğurur
- Pin konnektör: 50 / 65°C (CT50)
- Yüklü verim düşüşü: 22 / 38% (CT50) — yüksek düşüş gizli iç direnç göstergesidir

### 2.3 Otonom Eşikler

- GPS sapması: 1.2 / 3.0 m — tüm modellerde ortak (GPS doğruluğu platformdan bağımsız)
- Rota tamamlama: 96% / 88% (ters yorum; < 88% = CRITICAL) — ortak
- Sensör anomali oranı: 2.0% / 7.0% — ortak

### 2.4 Bağlamsal Gevşetme Kuralları

- **Rüzgar > 5.0 m/s:** GPS ve stabilite eşikleri %15 gevşetilir (`ok_gps *= 1.15`; `ok_stability *= 0.85`). Sürekli fonksiyon değil, ikili kapıdır.
- **Maksimum yüke yakın (> max_payload_kg'nin %90'ı):** yüklü modda motor akımı WARNING, arıza sinyali değil _beklenen uyarı_ olarak sınıflanır.

### 2.5 Kronik Tespit Eşikleri

- **Kalıcılık (Chain 2):** Motor, tüm alt-test kontrollerinin > %60'ında en sıcaksa "kronik en sıcak" sayılır; dengesizlik oranı kayıtların %40'ını aşmalıdır.
- **Batarya trendi:** 4+ kayıt boyunca 3 modun en az 2'sinde monoton artan deşarj oranı.
- **Aşınma eğimi:** max UL motor sıcaklığında 100 uçuş saatinde ≥ 1.2°C (OLS doğrusal regresyon; min 5 veri noktası).
- **Sahte iyileşme:** Kayıtlı bakım olayı olmadan, art arda ≥ 2 CRITICAL kaydı ardından tam-OK.

> Referans: Tam eşik tabloları → `thresholds.md`

---

## 3. Teşhis Modeli Açıklaması

### 3.1 Yaklaşım

Diagnostik sistemi Dart dilinde, sahada karşılaşılabilecek çeşitli senaryoları erken aşamada tespit edebilecek şekilde kurguladım. Bu yaklaşımın temelinde, sistemi lokal ortamda (Cihaz içinde ya da baibars sunucularında) çalışan bir "ilk müdahale katmanı" olarak konumlandırma fikri yer alıyor. Bu doğrultuda, zaman kısıtı nedeniyle 28 farklı tekil ve zincirli senaryo tanımlanarak simüle edildi.

Kısacası, şu anki yazılan teşhis motoru aşağıdaki bahsedilen ML/AI'ı belirli durumlarda simüle etmek için konmuştur. Production'da bu motor kendini bir AI/ML'ye bırakmalıdır.

Hedef teşhis mimarisi bu motoru _ambulans_ gibi ele alır. Vakaları ilk ele alan, değerlendiren, güven puanını hesaplayan, bulguları özetleyen ve yalnızca gerçekten gerektiğinde büyük bulut modeline yükselten hızlı, düşük maliyetli ilk müdahaleci. Cihazda veya Baibars sunucularında çevrimdışı, çağrı başına çıkarım maliyetsiz ya da düşük maliyetli ve düşük gecikmeli çalışır. Ardından gereken durumlarda kendini daha yetenekli bir modele, ya da modellere bırakır.

Bu ilk değerlendirme sonrasında sistem, elde ettiği verileri özetleyerek yalnızca gerekli durumlarda daha yüksek kapasiteli kapalı modellere (örneğin Opus 4.7) iletir. Böylece hem çıktıların belirli bir formatta ve kullanıcı arayüzüyle uyumlu olması sağlanır, hem de her işlem için gereksiz token tüketiminin önüne geçilir. Lokal çalışan AI katmanı, veriyi sadeleştirip optimize ederek ilettiği için ek bir verimlilik kazancı da sağlar. Bu mimari sayesinde toplam token kullanımında yaklaşık 5 ila 10 kat arasında bir optimizasyon elde edilmesi hedeflenmektedir.

### 3.2 Token Optimizasyonu

En kritik maliyet azaltma prensibi iki farklı AI'ı konumlandırıp sadece gerekli bilgileri maliyetli olana göndermektir.

| Teknik                                                         | Etki                                      |
| -------------------------------------------------------------- | ----------------------------------------- |
| Ham sayısal değerler yerine yalnızca durum alanlarını gönder   | token maliyeti azalması                   |
| Yapılandırılmış teşhis özeti (alt sistem/durum başına sayılar) | Tam kayıt geçmişini göndermeyi önler      |
| Her LLM çağrısından önce güven kapısı (< 0.60)                 | Gereksiz API çağrılarını ortadan kaldırır |
| Rutin çıktı için şablon tabanlı rapor                          | LLM sadece karmaşık anlatılarda gerekir   |

> Referans: Token optimizasyonu detayları → `drone_diagnostic_framework.md` §6

### 3.3 Neler Kırılabilir veya Yanlış Sonuç Üretebilir

Mevcut kural motorunun üretebileceği başlıca yanlış teşhis senaryoları:

**Model-geneli sabit eşikler bireysel sapmaları gizler.** Tüm CT50'lere aynı eşik uygulanır; ancak 450 saatlik bir CT50 ile 50 saatlik bir CT50'nin "normal" aralığı fiilen farklıdır. Yüksek saatli drone eşiğin hemen altında sürekli gezinebilir ve teşhis onu sağlıklı sayarken gerçekte kritik aşınma eşiğine yaklaşıyor olabilir.

**Güven skorları ampirik değil, elle atanmış.** Chain 2 için `confidence=0.85` ve Chain 9 için `confidence=0.90` değerleri gerçek doğrulama verisine değil, uzman tahminlerine dayanır. Güven kapısının (< 0.60) LLM yükseltmeyi tetiklediği düşünüldüğünde, yanlış kalibre edilmiş bir skor ya gereksiz LLM çağrısına (maliyet artışı) ya da gerçekten belirsiz bir bulgunun bastırılmasına (teşhis kaçırma) yol açar.

**Bağlamsal gevşetme binary kapı, sürekli fonksiyon değil.** Rüzgar 4.9 m/s'de eşik gevşemez; 5.1 m/s'de aniden %15 gevşer. Gerçek ortamda rüzgar etkisi sürekli bir fonksiyon olduğundan bu sert geçiş sınır değerlerde yanlış alarm veya alarm bastırma üretebilir.

**Zincir kuralları ilk koşulun kaçırılmasına duyarlı.** Chain 7 (sıcaklık-akım tutarsızlığı) `motor_current_status = CRITICAL` koşuluna bağlıdır. Akım sensörü de dropout yaşarsa zincir hiç tetiklenmez ve yangın riskli gizli arıza görünmez kalır.

**Kronik tespit minimum kayıt sayısı gerektirir.** Chain 2 ve batarya trendi için 4–5 kayıt yeterlidir; ancak az uçmuş yeni bir drone için kronik motor sapması hiç tespit edilemeyebilir. Filo içinde aynı arıza davranışı başka drone'larda gözlemlenmiş olsa bile bu bilgi mevcut mimaride kullanılmaz.

**Sahte iyileşme tespiti bakım kaydı kaydına bağlı.** Bakım olayı veri tabanına işlenmemişse (`art arda ≥ 2 CRITICAL → tam-OK` örüntüsü) sahte iyileşme gerçek iyileşme gibi görünür ve kritik bir aralıklı arıza gözden kaçar.

**NO_DATA yayılımı zincir kurallarını bastırır.** Motor sıcaklık alanlarından biri `null` olduğunda `motor_imbalance_flag = NO_DATA` olur ve buna bağlı tüm zincir kuralları (Chain 2, 9) bastırılır. Rulman aşınmasının en erken sinyali olan imbalance tam da sensör arızasına en yatkın bileşenle iç içe geçtiğinden, gerçek bir arıza dropout ile örtüşürse sessizce kaçırılabilir.

### 3.4 Gerçek Sensör Verisiyle Motor Nasıl İyileşir

Gerçek telemetri verisi teşhis motoruna entegre edildiğinde aşağıdaki iyileşmeler mümkün olur:

**Drone bazlı dinamik taban çizgileri.** Her drone'un kendi geçmişindeki OK uçuşları, o birime özgü beklenti zarfını (ortalama ± σ) oluşturur. Eşik, modelin tamamına değil bireysel drone'a uygulanır. 50 saat normal olan bir okuma 400 saatte anomali sayılabilir; mevcut aşınma eğimi kuralı bunu doğrusal eğimle kısmen yakalarken bireysel taban çizgisi bunu doğrudan ölçer.

**Güven skorlarının ampirik kalibrasyonu.** Gerçek bakım sonuçları (arıza teyidi, yedek parça değişim kayıtları) teşhis bulgularıyla eşleştirildiğinde her kuralın gerçek doğruluk oranı ölçülebilir. Böylece `confidence=0.85` gibi tahmin değerleri, ampirik kesinlik ve geri çağırma oranlarına dayalı gerçek olasılık skorlarıyla değiştirilir.

**Bayesian güven güncellemesi.** Aynı bileşende tekrarlayan WARNING bulguları, gözlem sayısı arttıkça arıza olasılığını sistematik olarak yükseltir. Mevcut sistem bunu "kronik sayaç" mantığıyla yakalarken Bayesian güncelleme her yeni gözlemi önceki güven üzerine orantısal ekler — daha az kayıtla daha erken uyarı verir.

**Rüzgar gevşetme fonksiyonunun süreklilik kazanması.** Gerçek uçuş verisiyle rüzgar hızı–GPS sapması ve rüzgar hızı–stabilite ilişkisi regresyonla modellenebilir. Binary kapı (> 5.0 m/s → %15 gevşe) yerini rüzgar hızına sürekli olarak örneklenmiş bir düzeltme fonksiyonuna bırakır; sınır değerlerdeki yanlış alarm oranı düşer.

**Aralıklı arızalar için zaman serisi tutarlılığı.** Gerçek sensör kayıtlarında aynı alan tekrar tekrar `NO_DATA` oluyorsa, bu örüntü kablo/konnektör sorununa işaret eder. Zaman serisi tutarlılık skoru, hangi bileşenin ne sıklıkla dropout yaşadığını sarar; zincir kuralları bu skor yüksekken kendiliğinden düşük güvenle tetiklenerek bastırılmak yerine düşük güvenli bulgu üretebilir.

---

## 4. Sistem Tasarım Kararları

### 4.1 Eksik Sensör Verisi (NO_DATA)

Herhangi bir ham değer `null` ise `ThresholdCalculator` `HealthStatus.noData` döner. Motordaki yayılım:

| Koşul                                                             | Davranış                                                                                                                                                              |
| ----------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tek alan NO_DATA                                                  | `sensor_dropout` bulgusu üretilir (severity: `noData`, confidence: 0.95). Sağlık skoru ağırlık 2 ile cezalandırılır — WARNING'den kötü, doğrulanmış CRITICAL'dan iyi. |
| Aynı kayıtta ≥ 3 alan NO_DATA                                     | `multi_field_dropout` bulgusu (CRITICAL, confidence: 0.90). Kök neden "tekil sensör arızası"ndan "veri yolu veya logger arızası"na kayar.                             |
| < 2 geçerli motor sıcaklığı                                       | Dengesizlik bayrağı = `NO_DATA`; dengesizliğe bağlı tüm zincir kuralları bastırılır.                                                                                  |
| Aynı alan kayıtlar arasında OK ↔ NO_DATA değiştirirse (≥ 2 geçiş) | Seviye 3 `intermittent_dropout` bulgusu. Gevşek konnektör veya kablo süreksizliği ima eder — kalıcı sensör arızasından farklı aksiyon gerektirir.                     |

### 4.2 Test Modları Arasındaki Çelişkili Sinyaller

Seviye 2 (`TestDiagnosticService`) kuralları, Seviye 1'in tek başına yanlış yorumlayacağı modlar-arası çelişkileri çözmek için vardır:

| Çelişki Örüntüsü                                       | Doğru Yorum                                                                | Kural                      |
| ------------------------------------------------------ | -------------------------------------------------------------------------- | -------------------------- |
| UL motor sıcaklığı OK → LD motor sıcaklığı CRITICAL    | Genel motor arızası değil, yük mekanizması arızası                         | Chain 1                    |
| UL deşarj OK → LD deşarj CRITICAL                      | Batarya iç direnci yalnız yüksek akım altında görünür                      | Battery Load Inconsistency |
| UL+LD stabilite OK; AU GPS CRITICAL + anomali CRITICAL | Drone mekanik olarak sağlam; yalnız otonom modda navigasyon modülü arızası | Autonomous Nav Regression  |
| AU rota CRITICAL, GPS OK, anomali OK                   | Batarya erken bitti — görev çok uzun, navigasyon arızası değil             | Chain 6                    |
| Motor sıcaklığı OK, motor akımı CRITICAL               | Sıcaklık sensörü arızası — en tehlikeli gizli arıza, yangın riski          | Chain 7                    |

Çapraz-mod korelasyon olmadan yalnız Seviye 1, `motor_overheating_ld` CRITICAL bulgusu üretir ve gerçek arıza pervane pitch sistemindeyken yanlış motor değişimine yol açar.

### 4.3 Düşük Güven Skoru Olduğunda Ne Olur

Her `DiagnosticFinding` 0.0–1.0 aralığında `confidence` taşır. Mevcut uygulamada bu değer `DiagnosticFindingCard` üzerinde bilgilendirici alan olarak gösterilir fakat UI aksiyonlarını kapılamaz.

**Hedeflenen üretim davranışı** (§3.1'deki YZ yükseltme mimarisi):

```
confidence ≥ 0.75    → bulgu teknisyene doğrudan gösterilir; LLM çağrısı yok
confidence 0.60–0.74 → bulgu "belirsiz" etiketiyle gösterilir;
                        açıklama isteğe bağlı LLM ile zenginleştirilir

confidence < 0.60    → ham bulgu düşük uzmanlık teknisyenine gösterilmez;
                        kapalı modele yükseltilir (örn. Opus 4.7);
                        yalnız LLM'in ürettiği yapılandırılmış açıklama gösterilir
```

Bu kapı, Chain 10 pilot anomalisi (0.65) veya Chain 4 belirsiz titreşim (0.55) gibi bulguların belirsiz atıflarla teknisyeni yanıltmasını önler.

---

## 5. Daha Fazla Zaman Olsaydı Neler İyileştirilirdi

### 5.1 Gerçek Sensör Verisi Boru Hattı

Gerçek telemetride ilk değişiklik, **model-geneli sabit eşikleri bırakıp drone-bazlı taban çizgilerine geçmek** olurdu. Her drone'un OK aralığındaki tarihsel uçuşları kendi ortalama ± σ zarfını oluşturur. 50 uçuş saatindeki bir CT50 için normal olan okuma, 400 saatte anomali olabilir — mevcut sistem bunu yalnız aşınma eğimi kuralıyla kısmen yakalar.

Python simülatör, görev sonunda drone üzeri logger'dan (Bluetooth veya hücresel yükleme ile) telemetri alan **gerçek zamanlı ETL boru hattı** ile değiştirilirdi. Her kayıt, teşhis motoruna girmeden önce şema uyumu, zamansal sıralama ve alanlar arası fiziksel tutarlılık açısından doğrulanırdı.

### 5.2 Belirsizlik Yönetimi

Mevcut güven skorları elle atanmış sabitlerdir. Uygun bir belirsizlik sistemi:

1. **NO_DATA belirsizliğini yayar:** NO_DATA komşusu olan metrikten türetilen bulgunun güveni otomatik düşer.
2. **Zamana bağlı güven azalması:** 90 gün önceki kayıttan gelen bulgu, dünkü bulgudan daha az teşhis ağırlığı taşır.
3. **Kayıtlar arası Bayes güncellemesi:** Aynı bileşende tekrarlayan WARNING bulguları arıza olasılığını sistematik artırır; yalnız ayrı bir "kronik" kuralı tetiklemekle kalmaz.

### 5.3 YZ Yükseltme Mimarisinin Olgunlaştırılması

Mevcut sistem, Katman 1 (kural motoru) ile Katman 3 (LLM) arasına tam anlamıyla çalışan bir **Katman 2 ML modülü** yerleştirilmemiş iki katmanlı bir mimaridir. Bu ara katmanın geliştirilmesi hem teşhis kalitesini artırır hem de LLM çağrı sayısını azaltarak maliyet optimizasyonunu derinleştirir.

**Batarya degradasyon trendi için zaman serisi regresyonu.** Drain rate'in kayıtlar boyunca monoton artışını saptamak için şu an basit karşılaştırma kullanılmaktadır. Doğrusal veya polinom regresyon, hem eğim istatistiklerini hem de güven aralığını üretir — "bu hızla azalırsa X uçuşta kritik eşiğe ulaşır" gibi proaktif bir bakım tahmini mümkün olur.

**Pilot bazlı anomali tespiti için kümeleme.** Chain 10 şu an pilot bazlı sapmanın var olup olmadığını saymakla sınırlıdır. K-means veya DBSCAN, drain_rate ve stability_score gibi operasyonel metrikleri pilot başına kümeleyerek hangi pilotun sapma oluşturduğunu ve bu sapmanın hangi manevra türünden (sert iniş, aşırı yük, ani dönüş) kaynaklandığını belirleyebilir.

**Çevre koşulu–anomali korelasyonu.** Hava sıcaklığı ve rüzgar hızına göre anomali oranının nasıl değiştiğini regresyonla modellemek, "bu drone sıcak ve rüzgarlı koşullarda diğerlerinden daha fazla bozuluyor" gibi bileşene özgü çevre duyarlılığını ortaya çıkarır. Bu bilgi, mevcut binary bağlamsal gevşetme kuralını sürekli bir fonksiyona dönüştürür.

**Sahte iyileşme ve aralıklı arıza için dizi örüntüsü eşleme.** Bakım kaydı olmadan `[CRITICAL × N → OK]` dizisini yakalamak, şu anki eşik kontrol yaklaşımıyla kırılgandır. LSTM veya HMM tabanlı bir dizi modeli, her drone'un durum geçiş örüntüsünü öğrenerek "bu geçiş bu drone için alışılmamış — muhtemelen sahte iyileşme" şeklinde anormal durum değişimlerini işaretleyebilir.

**Filo içi karşılaştırmalı kronik tespit.** Kronik tespiti şu an tek drone geçmişiyle sınırlıdır. Aynı model ve kullanım koşulundaki dronları karşılaştırmak (örn. Hotelling T² veya yalıtım ormanı), bireysel drone'un filo medyanından ne kadar saptığını ölçer. Sapma sürekli büyüyen bir drone, henüz eşiği aşmasa bile kronik bozulma sinyali verebilir. Bu yaklaşım özellikle filo genelinde sistemik batch sorunlarını erken saptamak için kritiktir (bkz. 5.5).

### 5.4 Arayüz İyileştirmeleri

- **Bileşen kaplamalı 3D drone modeli:** M2 bayraklandığında 3D mesh üzerinde doğrudan vurgulansın. Saha teknisyeni hangi fiziksel parçayı kontrol edeceğini anında görür.
- **Metrik başına zaman serisi grafikler:** Uçuş geçmişinde motor sıcaklığı, batarya deşarj oranı ve titreşim çizilsin. Görsel trend, sayısal eğim metninden daha aksiyon alınabilirdir.
- **Onarım öneri akışı:** Spesifik bulgulara göre denetim adımlarını, tahmini onarım süresini ve gereken ekipmanı yönlendiren checklist.
- **LLM yükseltmesinde push bildirim:** Dart motoru Katman 3'e yükselttiğinde yapılandırılmış LLM yanıtı teknisyene asenkron push ile iletilsin.
- **Filo-seviyesi panel:** Tüm drone'ların sağlık skorları risk seviyesine göre sıralansın; arıza olmadan proaktif bakım planlansın.

### 5.5 Filo Seviyesinde Çoklu-Drone Kronik Analizi

Mevcut kronik tespit tek drone geçmişinde çalışır. Filo görünümüyle **drone'lar arası karşılaştırma** mümkün hale gelir: aynı müşteride çalışan 12 CT50'nin 8'inde aynı M2 önyargısı varsa bu bireysel drone sorunu değil, filo düzeyi sinyaldir (muhtemel tedarik batch problemi). Bu, hem teşhisi hem aksiyonu değiştirir — "bu drone'da M2 değiştir" yerine "tüm batch'te M2 kalite incelemesi yap".

---

## 6. Çıkarılan Dersler

**1. Sentetik veri, yalnızca dağılımı değil, nedensel yapıyı da yansıtmalıdır.**  
Simülatördeki en doğru karar UL → LD → AU zincirini ve Newton soğuma modelini kullanmak oldu. Satırların birbirinden bağımsız, rastgele üretilmesi; UL sonunda soğuk olan bir motorun LD başlangıcında sıcak görünmesi gibi fiziksel olarak imkansız durumlara yol açardı. Çapraz-mod korelasyon kurallarını anlamlı kılan temel unsur nedensel tutarlılıktır. Bu tutarlılık olmadan Seviye 2 servis, gerçek arızaları değil simülatör kaynaklı yapay örüntüleri yakalardı.

**2. Güven skorları atanmamalı, veriye dayalı olarak doğrulanmalıdır.**  
Chain 2 için `confidence=0.85`, Chain 9 için `confidence=0.90` vermek ölçümden çok uzman tahminidir. Üretim ortamında bu değerlerin, etiketli geçmiş sonuçlarla ampirik olarak kalibre edilmesi gerekir. Bu aşamaya kadar güven skorları, mutlak olasılık olarak değil, bulgular arasında göreli güven sıralaması olarak değerlendirilmelidir.

**3. Bu alanda üç katmanlı teşhis mimarisi doğru ayrımı sağlar.**  
Tek kayıt, modlar arası ve çoklu kayıt analizi şeklindeki ayrım, her katmanda kapsam taşmasını önledi. Tek kayıt katmanı akut olayları (ani motor arızası, konnektör ısı artışı), çapraz-mod katmanı test koşulları arasındaki yapısal uyumsuzlukları (yük mekanizması arızası, navigasyon regresyonu), çoklu kayıt katmanı ise uçuş bazında görünmeyen kademeli bozulmaları (kronik rulman aşınması, batarya hücre kapasite kaybı) yakalar. Bu yapıları tek bir monolitik analizöre toplamak hem kuralları yorumlamayı zorlaştırır hem de bağımsız test edilebilirliği azaltır.

**4. "İlk müdahaleci yerel YZ" yaklaşımı maliyet ve gecikme açısından güçlü bir çözümdür.**  
Tarlada, dronun yanında çalışan bir teknisyen için 3 saniyelik LLM gecikmesi pratik değildir. Dart motoru ağ bağımsız çalışarak sonuçları < 1 ms sürede üretebilir. Opus 4.7 gibi modellere yükseltme yalnızca gerçekten belirsiz vakalarda yapılır. Ayrıca Dart katmanı bulguları önceden özetlediği için LLM'e ham telemetri yerine daraltılmış ve yapılandırılmış bir girdi gönderilir. Bu yaklaşım hem hızı artırır hem de "her şeyi LLM'e gönder" modeline kıyasla token kullanımını yaklaşık 5×–10× azaltabilir.

**5. Açıklanabilirlik bu sistemde bir ek özellik değil, temel gereksinimdir.**  
Her teşhis bulgusu, belirli bir fiziksel bileşene ve uygulanabilir bakım aksiyonuna bağlanan Türkçe açıklama içerir. Bu gereksinim mimariyi doğrudan şekillendirdi; yeni arızalara duyarlılık potansiyeline rağmen ML tabanlı anomali tespitinin tercih edilmemesinin ana nedeni budur. Örneğin 0.87'lik bir anomali skoru teknisyen için tek başına anlamlı değildir; buna karşılık "M2 motor sıcaklığı diğer motorlara göre sistematik olarak yüksek — rulman aşınması olası" gibi bir ifade, doğrudan nerede kontrol yapılacağını net biçimde gösterir.

---

_Dokümanın kapsadıkları: `lib/services/drone_diagnostic_service.dart` · `lib/services/sub_test_diagnostic_service.dart` · `lib/services/test_diagnostic_service.dart` · `lib/services/threshold_calculator.dart` · `lib/features/drone_analysis/drone_analysis_page.dart` · `baibars_python_script/generate_data_v2.py` · `docs_of_core/drone_diagnostic_framework.md` · `docs_of_core/thresholds.md` · `baibars_python_script/generate_data_v2_inceleme.md`_
