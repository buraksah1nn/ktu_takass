# ktu-takas / backend & mimari notları

KTÜ içi öğrenci takas platformu (`ktutakas.tech` üzerinde canlıya alındı). Bu repo, projenin backend tarafında üstlendiğim takas algoritmalarını ve sistem mantığını özetlemek amacıyla hazırlanmıştır.

---

### 1. Problem ve Çözüm Yaklaşımı

Geleneksel ikinci el platformlarında takas mantığı *"A kişisi B'nin ürününü istiyorsa, B de mutlaka A'nın ürününü istemeli"* kuralına takılır. Gerçek hayatta bu ikili eşleşme nadiren gerçekleştiği için platform tıkanıyordu.

Bunu çözmek için backend'e **yönlü graf (directed graph)** ve **BFS (Breadth-First Search)** tabanlı bir döngü tespit mekanizması ekledim.

```text
Doğrudan Eşleşme (2'li):
[Kullanıcı A] <========================> [Kullanıcı B]
(Gitar verir, Çadır ister)             (Çadır verir, Gitar ister)

Zincir Eşleşme (3'lü Döngü):
[Kullanıcı A: Gitar] ────────> [Kullanıcı B: Çadır]
         ^                               │
         │                               v
         └───────────── [Kullanıcı C: Python Dersi]
```

Sisteme yeni bir ilan eklendiğinde `eslesmeleri_kontrol_et_ve_bildir()` tetiklenir:
1. İlanların aranan/sunulan metinlerinden anahtar kelimeler ayıklanarak kullanıcılar arası yönlü bir graf düğümü oluşturulur.
2. BFS kuyruğu ile önce en kısa yol olan **2'li takas (A <-> B)** aranır.
3. Bulunamazsa derinlik 3'e genişletilerek kapalı döngü (A -> B -> C -> A) taranır.
4. Eşleşme tespit edildiğinde otomatik bir `UcluTakasOdasi` oluşturulup taraflara eş zamanlı bildirim gider.

---

### 2. Kritik Kararlar & Uç Durum (Edge-Case) Yönetimi

Bu sistemi kurarken karşılaştığım ve çözdüğüm temel mantık hataları:

- **Döngü Sonsuzluğu & Tekrarlayan Eşleşmeler:**
  Kullanıcılar bir 3'lü takası reddettiğinde, sistemin aynı kombinasyonu tekrar tekrar önermemesi gerekiyordu. Bunun için bir `UcluTakasKaraliste` tablosu kurdum. İlan ID'leri sıralı (`sorted([id_a, id_b, id_c])`) tutularak reddedilen üçlü kombinasyonlar bir daha kuyruğa alınmaz.

- **Zaman Aşımı ve Askıda Kalan İlanlar:**
  Tekliflerden birine 24 saat içinde yanıt verilmezse tüm zincir kilitlenebilirdi. `check_uclu_takas_timeouts()` fonksiyonu ile süresi dolan odaları otomatik `iptal_edildi` durumuna çekip kilitli ilanları yeniden `aktif`e döndürdüm.

- **Veri Bütünlüğü ve Çift Onay:**
  Takasın tamamlanması için iki tarafın da (`teklif_eden_onayladi`, `ilan_sahibi_onayladi`) onay vermesi şart koşuldu. Taraflardan biri iptal ettiğinde bağlı tüm tekliflerin durumu atomik olarak güncellenir.

- **Dinamik Güven Puanı:**
  Kullanıcıların profilindeki puan ve rozetler veritabanında statik bir alan olarak tutulmak yerine model property'si olarak yazıldı. Kullanıcının aldığı puanların `Avg` agregasyonu ve tamamlanan takas sayısı anlık hesaplanır.

---

### 3. Veritabanı İlişkileri Özeti

Modellemede 10'a yakın ilişkili Django modeli kullandım:

```text
User ──(1:1)──> Profil (Güven puanı & rozet hesaplamaları)
 │
 ├──(1:N)──> Ilan ──(1:N)──> IlanResim
 │            │
 │            ├──(N:M)──> Favori
 │            └──(1:N)──> TakasTeklifi ──(1:1)──> Degerlendirme (1-5 Puan)
 │
 └──(1:N)──> Istek (Wishlist: Aranan kelime sisteme düşünce tetiklenir)

UcluTakasOdasi ──(3x User + 3x Ilan + 3x TakasTeklifi) ──> UcluTakasMesaj
```

---

### 4. Entegrasyonlar ve Ortam Kurulumu

- **İşlem E-postaları:** Hesap aktivasyonu, yeni teklif ve graf eşleşme bildirimleri için `django-anymail` üzerinden **Brevo API** kullanıldı.
- **Medya Dosyaları:** Render gibi geçici dosya sistemine sahip sunucularda görsellerin kaybolmaması için **Cloudinary Storage** entegre edildi.
