# Teheccüdify

Türkiye'deki Müslümanlar için Diyanet İşleri Başkanlığı verisiyle namaz vakitlerini ve özellikle **teheccüd vaktini** hesaplayan, tek dosyalık bir Progressive Web App (PWA).

**Canlı:** [nifydevify.github.io/teheccudify](https://nifydevify.github.io/teheccudify/)

---

## Ne Yapar?

- **Günün namaz vakitlerini** Diyanet'ten alıp gösterir (imsak, güneş, öğle, ikindi, akşam, yatsı).
- **Teheccüd vaktini hesaplar:** akşam ile imsak arasındaki gecenin **son üçte birini** otomatik bulur — sünnet üzere en faziletli vakti gösterir.
- **Şu an hangi vakit?** O an aktif vakti (örn. "Öğle vakti") ve çıkmasına ne kadar kaldığını canlı olarak söyler.
- **Kerahat vakti uyarısı:** Hanefî mezhebine göre namazın kılınmadığı üç vakti (güneş doğumu sonrası, istiva, güneş batımı) hesaplar. Üç katmanlı uyarı: o an kerahatteysen kırmızı banner, 15 dk içinde kerahat başlayacaksa altın "yaklaşıyor" uyarısı, ayrıca bugünün tüm kerahat pencereleri listelenir.
- **Kıble pusulası:** GPS + telefon sensörüyle Kabe yönünü gösterir, telefon kıbleye dönünce yeşil parıldar.
- **Hicri tarih** her gün otomatik güncellenir.
- **Bildirim kurma:** Teheccüd vakti gelince hatırlatma. Sadece uygulama açıkken çalışır (sunucusuz PWA olduğu için iOS arka plan bildirim mümkün değil); garantiye almak için iOS Saat uygulamasından alarm kurmanız önerilir.

## Neden Var?

- Diyanet'in resmi sitesinde teheccüd vakti **doğrudan listelenmez** — kullanıcının akşam ve imsak vakitlerinden kendisinin hesaplaması gerekir.
- Var olan namaz vakti uygulamalarının çoğu reklam, izin, hesap zorunluluğu istiyor; gizlilik ve sadelik yok.
- Bu uygulama: **tek dosya, hesap yok, reklam yok, takip yok, internet bir kez gerek.**

## Kurulum

İndirip kurmak gibi bir şey yok. Kullanmanın iki yolu:

### iPhone (önerilen)
1. Safari'de [https://nifydevify.github.io/teheccudify/](https://nifydevify.github.io/teheccudify/) adresini aç.
2. Paylaş tuşuna bas → **"Ana Ekrana Ekle"**.
3. Ana ekranda Teheccüd ikonu çıkar; bağımsız bir uygulama gibi açılır.
4. İlk açılışta konumunu seç (GPS otomatik veya manuel) — yıllık vakitler tek seferde indirilir.
5. Sonrası tamamen offline çalışır.

### Android
Chrome / Firefox üzerinden aynı şekilde "Ana ekrana ekle" seçeneğiyle yüklenebilir.

### Bilgisayar
Tarayıcıda direkt aç, browser sekmesi olarak kullan.

## Nasıl Çalışır?

| Bileşen | Detay |
|---|---|
| Veri kaynağı | [ezanvakti.imsakiyem.com](https://ezanvakti.imsakiyem.com) — doğrudan Diyanet İşleri Başkanlığı vakitleri |
| Çalışma modeli | İlk kurulumda yıllık 365 günlük veriyi indirir, sonrası offline |
| Saklama | `localStorage` (tarayıcıda yerel, hiçbir sunucuya gönderilmez) |
| Konum | GPS (otomatik) veya il/ilçe seçimi (manuel). GPS kullanılıyorsa OpenStreetMap Nominatim ile ilçe ismi tespit edilir |
| Kıble | Great-circle bearing formülüyle Kabe yönü; cihaz pusulasıyla eşleşir |
| Pusula yumuşatma | 8 örneklik dairesel ortalama — sensör titremesine karşı |
| Kerahat | Hanefî mezhebine göre üç vakit hesaplanır: güneş doğumu sonrası 45 dk, istivaya 10 dk, güneş batışına 40 dk |
| Tema | Koyu altın, Cormorant Garamond + Jost fontları |

## Teknoloji

- **Tek `index.html` dosyası** — HTML, CSS, JS hepsi inline.
- **Vanilla JavaScript**, framework yok, build sistemi yok, npm yok.
- **GitHub Pages** üzerinden statik servis.
- Sıfır bağımlılık — sadece Google Fonts CDN'i (o da `display=swap` ile fallback'lı).

## Gizlilik

- Hiçbir sunucuya kullanıcı verisi gönderilmez.
- Konum tarayıcıda kalır, sadece üç servise istek gider:
  - **ezanvakti.imsakiyem.com** — vakit verisi indirme (ilk kurulum + manuel güncelleme)
  - **nominatim.openstreetmap.org** — GPS kullanılırsa ilçe ismi tespiti (opsiyonel)
  - **fonts.googleapis.com** — fontlar (offline'da fallback fontla çalışır)
- Reklam, analytics, çerez yok.

## Geliştirme

Tek dosya olduğu için katkı süreci basit:

```bash
git clone https://github.com/nifydevify/teheccudify.git
cd teheccudify
# index.html'i düzenle
git add index.html
git commit -m "açıklayıcı mesaj"
git push origin main
```

GitHub Pages 1-2 dakika içinde otomatik yayınlar.

`CLAUDE.md` dosyası kod tabanına dair detaylı brifing içerir (mimari kararlar, çözülen problemler, açık işler).

## Lisans

Açık kaynak, kişisel kullanım için. Vakit verisi telif hakkı Diyanet İşleri Başkanlığı'na aittir.

## Geri Bildirim

Bug bildirimi veya öneri için GitHub Issues kullanın.
