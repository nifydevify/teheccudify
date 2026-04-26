# Teheccüdify — Proje Brifingi (Claude Code için)

> Bu dosya, projeye yeni katılan Claude Code instance'ı için yazıldı. Kullanıcı (Nify) ile bu projeyi başka bir Claude konuşmasında baştan kurduk; bu CLAUDE.md, o konuşmanın özeti ve devam eden iş için referans.

## Hızlı Özet

**Tek dosyalık bir web uygulaması** (`index.html`) — Türkiye'deki Müslüman kullanıcılara Diyanet İşleri Başkanlığı verisi ile namaz vakitleri ve özellikle **teheccüd vakti hesaplama** sunan PWA. iPhone'da Safari → "Ana Ekrana Ekle" ile bağımsız uygulama gibi çalışır.

- **Live URL:** https://nifydevify.github.io/teheccudify/
- **Hosting:** GitHub Pages (master/main branch root'undan servis)
- **Build sistemi yok**, framework yok, tek HTML dosyası

## Kullanıcı Hakkında

Kullanıcının adı **Nify**. Kendisi:
- Yazılım geliştirme geçmişi yok — kodu kendisi yazmıyor
- Türkçe konuşuyor, İstanbul Çekmeköy'de yaşıyor
- Hitap olarak sıkça **"kral"** kelimesini kullanıyor, sen de aynı şekilde hitap edebilirsin
- Pratiklik ister, uzun açıklamalar yerine somut adımlar tercih eder
- Dini hassasiyetleri var (pratisyen Müslüman, Hanefî mezhebi)
- Geri dönüşleri direkt: bug'ı görür, hemen söyler

## Mimari Kararlar (önemli — değiştirmeden önce nedenini sor)

### 1. Tek dosya yaklaşımı
Tüm uygulama tek bir `index.html` dosyasında — HTML, CSS, JS hepsi inline. **Sebep:** GitHub Pages'e zero-config deploy (root URL otomatik servis eder, dosya adı belirtmeye gerek yok), Safari "Ana Ekrana Ekle" sorunsuz çalışsın, kullanıcı dosyayı tek başına bile kullanabilsin.

### 2. Veri kaynağı: ezanvakti.imsakiyem.com
Diyanet İşleri Başkanlığı verisini sunan üçüncü taraf bir API. Resmi Diyanet API'si (awqatsalah.diyanet.gov.tr) form başvurusu ve 5 istek limiti gerektirdiği için kullanılamadı.

**Endpoint'ler:**
- `GET /api/locations/states?countryId=2` → Türkiye illeri
- `GET /api/locations/districts?stateId={id}` → İlin ilçeleri
- `GET /api/prayer-times/{districtId}/yearly` → 365 günlük vakit verisi (yıllık, tek istekte)

**Yanıt formatı:**
```json
{
  "success": true,
  "data": [
    {
      "date": "2026-01-01T00:00:00.000Z",
      "times": { "imsak": "06:49", "gunes": "08:21", "ogle": "13:12", "ikindi": "15:31", "aksam": "17:52", "yatsi": "19:18" },
      "hijri_date": { "full_date": "12 Recep 1447" }
    }
  ]
}
```

API key gerektirmiyor, CORS açık. Rate limit: 5 dakikada 100 istek.

### 3. Offline-first çalışma modeli
- İlk kurulum: internet gerekli (ilçe seç → yıllık veri indirip localStorage'a yaz)
- Sonraki tüm açılışlar: tamamen offline
- Manuel "Verileri Güncelle" butonuyla yenileme
- Kullanıcı başka şehre giderse "Konumu Değiştir" ile yeniden kurulum yapar

### 4. localStorage şeması
```js
{
  version: 3,
  location: { districtId, districtName, stateName, lat?, lng? },
  yearData: {
    year: 2026,
    times: { "MMDD": "imsak|gunes|ogle|ikindi|aksam|yatsi" },  // 365 entry
    hijri: { "MMDD": "12 Recep 1447" }
  },
  lastUpdate: timestamp,
  userLat: number | null,   // GPS varsa gerçek koordinat, yoksa STATE_COORDS fallback
  userLng: number | null
}
```

Storage key: `teheccud_v3`. Versiyon sadece **şema bozucu değişiklik** olduğunda bumplanır — bumplandığında eski state otomatik invalid olur ve setup tekrar tetiklenir, kullanıcı yıllık veriyi yeniden indirir. Bug fix gibi geriye uyumlu değişiklikler için bumplama (kullanıcı verisini boşuna silersin).

### 5. Kıble hesaplama
- Kabe koordinatı: 21.4225, 39.8262
- Bearing: standart great-circle formula
- Distance: Haversine
- 81 ilin yaklaşık merkez koordinatı kodda gömülü (`STATE_COORDS` objesi) — manuel ilçe seçiminde GPS yoksa kıble hesabı için il merkezi kullanılır

### 6. Pusula sensörü
- iOS: `deviceorientation` event + `webkitCompassHeading`
- iOS 13+ izin gerektirir: `DeviceOrientationEvent.requestPermission()` (button click içinde olmalı)
- Android: `deviceorientationabsolute` + `360 - alpha`
- **Smoothing:** Son 8 okumanın circular mean'i alınır (basit ortalama 359°+1° için 180° verir, yanlış)
- **Accuracy göstergesi:** iOS'taki `webkitCompassAccuracy` ile yeşil/altın/kırmızı durum gösterilir (>30° veya negatif = kalibrasyon gerek)

### 7. Reverse geocoding: Nominatim (OpenStreetMap)
GPS akışında ilçeyi doğru tespit etmek için Nominatim'e reverse geocode isteği atılır.
- **Endpoint:** `https://nominatim.openstreetmap.org/reverse?lat=X&lon=Y&format=jsonv2&accept-language=tr&zoom=12`
- **Hangi alan kullanılır:** `address.town` öncelikli (Türkiye için en güvenilir, ilçe seviyesi). Fallback: `city_district → county → municipality → village`. `suburb` mahalle düzeyindedir, kullanılmaz.
- **Match:** `normalizeDistrictName()` Türkçe karakter ve büyük/küçük farkını eler (Nominatim "Çekmeköy" → API "ÇEKMEKÖY"; bazen API ASCII verir: "ARNAVUTKOY"). Tam eşleşme öncelikli, sonra ön ek eşleşmesi.
- **Hata durumu:** 5sn timeout, fetch hatası → null döner, çağıran taraf eski "MERKEZ" tabanlı zayıf öneriye düşer. Kullanıcı asla mahsur kalmaz.
- **Rate limit:** Nominatim 1 req/sec; setup nadir yapıldığı için sorun değil.
- **UX:** Match olursa belirgin yeşil onay kartı ("Konum tespit edildi · Çekmeköy · Evet, devam et / Hayır, listeden seçeyim"). "Evet" → tek tıkla setup. "Hayır" → ilçe listesi.

### 8. Kerahat vakitleri
Hanefî mezhebine göre namazın haram/şiddetle mekruh olduğu üç vakit hesaplanır. Süreler **yaklaşık** değerlerdir (alimler arasında ufak farklar var); kodda sabit olarak tanımlı, ihtiyatlı seçildi:
- `KERAHAT_SUNRISE_MIN = 45` — Güneş doğumundan sonra mızrak boyu yükselene kadar
- `KERAHAT_ISTIVA_MIN = 10` — Öğle vakti girmeden önce (zeval/istiva, güneş tepe noktasında)
- `KERAHAT_GURUB_MIN = 40` — Akşam namazından önce (güneş sararma)

`getKerahatWindows(todayPT)` üç pencereyi `{name, start, end}` array'i olarak döner. `findActiveKerahat(now, windows)` o an aktif olanı (varsa) döner.

**UX üç katmanlı:**
- **Aktif kerahat uyarısı** (`.kerahat-warning`): Şu an kerahat içindeyse kırmızı banner, "Çıkışına X dk var" countdown.
- **Yaklaşan kerahat uyarısı** (`.kerahat-upcoming`): Aktif değilse, 15 dk içinde başlayacak kerahat varsa altın banner ("Başlamasına X dk var · Acele et"). `findUpcomingKerahat(now, windows, withinMin)` döndürür. Kullanıcının namaza başlamadan önce yaklaşan kerahatı görmesi için. Aktif olana öncelik verilir.
- **Liste** (`.kerahat-list`): Vakitler listesinin altında her zaman görünür. Bugünün 3 penceresi; geçenler `.passed` ile soluk, aktif olan `.active` ile kırmızı.

**Yan etki:** Eski `findCurrentPrayer` segment ismi `'Kerahat (Güneş)'` (sunrise → dhuhr) → `'Kuşluk'` olarak düzeltildi. Eski etiket yanıltıcıydı çünkü gerçek kerahat sadece ilk 45 dk; gerisi duha/kuşluk vakti, nafile namaza açık. `prayerList[].seg` haritalaması da `'Kuşluk'` olarak güncellendi (ikisi tutarlı olmalı).

### 9. Bildirim sistemi sınırlı
**Mevcut:** `Notification` API + `setTimeout` — sadece uygulama açıkken çalışır. Sayfa kapanınca timer ölür.

**Neden gerçek arka plan bildirim yok:** iOS 16.4+ web push destekliyor ama VAPID anahtarı + push sunucusu (FCM/APNs) gerekir. Bu proje GitHub Pages'da sunucusuz olarak host ediliyor; backend yok. OneSignal/Firebase gibi 3. taraf servis eklemek hem proje sadelik felsefesine ters hem de gizlilik politikasını bozar (README'de "hiçbir sunucuya kullanıcı verisi gönderilmez" diye yazıyor).

**UX kararı:** Butonun altında dürüst not (`.notify-hint`): "Bildirim sadece uygulama açıkken çalışır. Garantiye almak için iOS Saat alarmı kurabilirsin." Kullanıcıya yanlış beklenti yaratmıyoruz.

**Açık iş listesindeki "Service Worker" maddesi:** SW eklemek offline cache'i iyileştirir ama bildirim sorununu çözmez (push olmadan SW uyutulur).

## Çözülen Önemli Problemler

### Problem 1: Veri doğruluğu
**Sorun:** İlk denemelerde adhan-js kütüphanesi kullandım, Türkiye metodu Diyanet ile 10-15 dk uyumsuzluk verdi.
**Çözüm:** ezanvakti.imsakiyem.com API'si kullanıldı, doğrudan Diyanet verisi geliyor, fark yok.

### Problem 2: file:// CORS
**Sorun:** Yerel dosya olarak Safari'de açıldığında fetch engelleniyordu.
**Çözüm:** GitHub Pages üzerinden HTTPS servis edildi.

### Problem 3: GPS izni
**Sorun:** Netlify password protection açıkken Safari "Origin does not have permission" hatası verdi.
**Çözüm:** GitHub Pages'a geçildi, sorun çözüldü.

### Problem 4: GPS sadece il bulup direkt ilçe seçiyordu
**Sorun:** GPS → en yakın il → o ilin "merkez" ilçesi otomatik seçiliyordu, kullanıcı kendi ilçesini (Çekmeköy gibi) seçemiyordu.
**Çözüm:** GPS → en yakın il → ilçe **listesi** gösterilir, kullanıcı kendi ilçesini seçer.

### Problem 5: Her şehirde aynı kıble açısı
**Sorun:** `completeSetup`'ta `loc.lat || existingState.userLat || null` mantığı vardı; manuel ilçe seçiminde `loc.lat` null geldiği için eski şehrin GPS koordinatı tekrar kullanılıyordu, kıble açısı yanlış çıkıyordu.
**Çözüm:** `existingState`'e fallback tamamen kaldırıldı. Yeni mantık: GPS varsa onu kullan, yoksa **seçilen ilin** `STATE_COORDS[loc.stateName]` merkez koordinatına düş. Eski state'in koordinatı asla taşınmaz.

### Problem 6: Pusula sapması
**Sorun:** Sensör titrek, anlık 10-20° sıçramalar yapıyordu.
**Çözüm:** 8 sample'lık circular mean smoothing + accuracy indicator (yeşil/altın/kırmızı durum).

### Problem 7: Bildirim çift kuruluyordu
**Sorun:** "Bildirim Kur" butonuna ikinci kez tıklamak yeni bir `setTimeout` daha kuruyordu — kullanıcı aynı vakitte 2-3 bildirim alıyordu. `sessionStorage` flag'i sadece UI metnini değiştirmek için bakılıyordu, tıklama mantığında erken return yoktu.
**Çözüm:** Click handler'ın en başında `sessionStorage.getItem(key) === '1'` kontrolü → erken return. `sessionStorage.setItem(key, '1')` artık `setTimeout`'tan **önce** çağrılıyor (race önlemi).

### Problem 8: GPS sonrası ilçe önerisi alakasızdı
**Sorun:** GPS → en yakın il bulunduktan sonra, ilçe önerisi sadece `name === stateName || name === 'MERKEZ'` ile yapılıyordu. Kullanıcı Çekmeköy'de olsa bile İstanbul'un "Adalar" gibi bir ilçesi öneriliyordu (alfabetik sıraya bağlı). API ilçe koordinatı döndürmediği için haversine'le en yakın ilçeyi bulmak mümkün değildi.
**Çözüm:** Nominatim reverse geocoding entegre edildi (Bölüm 7'ye bak). GPS akışında `Promise.all` ile il listesi ve Nominatim paralel çağrılır. Match bulunursa belirgin onay kartı çıkar; match olmazsa eski zayıf öneri fallback olarak korunur.

### Problem 9: "Kerahat (Güneş)" etiketi sabah-öğle arası tüm vakti yanlış kapsıyordu
**Sorun:** `findCurrentPrayer` segments dizisinde `{ name: 'Kerahat (Güneş)', start: sunrise, end: dhuhr }` vardı. Kullanıcı "Şu an" kartında öğleye kadar 4-5 saat boyunca "Kerahat (Güneş) vakti" görüyordu — halbuki gerçek kerahat sadece güneş doğumundan sonra ~45 dk; gerisi kuşluk vakti, namaza açık.
**Çözüm:** Segment ismi `'Kuşluk'` olarak değiştirildi. Gerçek kerahat hesabı ayrı bir katman olarak `getKerahatWindows()` ile yapılıyor (Bölüm 8). `prayerList[].seg` mapping de `'Kuşluk'` olarak güncellendi (segments ile match tutması için).

### Problem 10: Yıl son günü öğleden sonra teheccüd vakti saçmalıyordu
**Sorun:** `tomorrowPT` yoksa (örn. 31 Aralık öğleden sonra, 2027 verisi henüz yok), `renderTimes` fallback olarak `calcTeheccud(todayPT, todayPT)` çağırıyordu. Bu durumda `nightMs = todayPT.fajr - todayPT.maghrib` negatif çıkıyor (~-12 saat), `lastThirdStart` saçma değer veriyordu (örn. öğleden sonra 10:00 sabah).
**Çözüm:** Fallback kaldırıldı. `tomorrowPT` yoksa `teh = null` ve kullanıcıya `error-msg` kartı + "Verileri Güncelle" butonu gösteriliyor.

### Problem 11: Tab geçişinde iOS'ta "İzin hatası" mesajı
**Sorun:** `startCompass` her çağrıldığında `DeviceOrientationEvent.requestPermission()` çağırıyordu. Tab geçişinde `renderQibla` `qiblaState.permGranted` true ise `startCompass`'i otomatik tetikliyor — bu user gesture dışında oluyor. iOS'ta `requestPermission` user gesture dışında exception fırlatır; catch bloğu kullanıcıya yanlışlıkla "İzin hatası" mesajı yazar.
**Çözüm:** `requestPermission` çağrısının başına `!qiblaState.permGranted` koşulu eklendi. İzin bir kez alındıktan sonra tab geçişinde tekrar sorulmaz.

## Açık İşler / Yapılabilecek Geliştirmeler

Kullanıcı sırasıyla şunlar üzerinde çalışmak isteyebilir (bunları kullanıcı **istediğinde** yap, kendiliğinden başlatma):

1. **Bildirim sistemi iyileştirme** — Şu an sadece sayfa açıkken `setTimeout` ile çalışıyor. iOS Safari PWA'larda gerçek push notification 16.4+ destekliyor ama service worker gerekiyor, henüz eklenmedi.
2. **Service worker eklemek** — Tam offline + arka plan bildirim için.
3. **Yıl geçişi otomasyonu** — Şu an 2026 verisi var, 2027 için manuel güncelleme gerekiyor. Yıl geçince otomatik refresh yapılabilir.
4. **Çoklu konum kaydetme** — Şu an tek konum saklıyor, ama kullanıcı seyahatte ise "Çekmeköy" + "Ankara" + "Mekke" gibi birden fazla saklamak isteyebilir.
5. **Tasbih/sayaç sekmesi** — Kullanıcı henüz istemedi ama olası bir feature.
6. **Hatim takibi** — Kullanıcı henüz istemedi ama olası bir feature.

Kullanıcı **"benim istediğim bir tane var onu sana söyleyeceğim"** demişti, ama sonra başka konulara geçtik. Soracaksan sor: "Kıble özelliği önce sen bana 'istediğim bir şey var' demiştin, o neydi?"

## Kod Stili

- **JavaScript:** vanilla JS, ES6+, framework yok. `'use strict'` üstte.
- **CSS:** CSS variables (`--gold`, `--ink`, `--bg` vs.) ile koyu altın tema. Animasyonlar yumuşak (`cubic-bezier`).
- **Türkçe:** Tüm UI metinleri Türkçe. `toLocaleLowerCase('tr')` ve `localeCompare(..., 'tr')` kullan, çünkü "İ" → "i" dönüşümü vs. doğru çalışsın.
- **Yorumlar:** Türkçe açıklamalar var, devam ettirirken aynı dilde yaz.
- **Fontlar:** Cormorant Garamond (serif, başlıklar) + Jost (sans, etiketler).
- **Renkler:** Koyu lacivert arka plan (`#0a0e1a`), altın aksan (`#d4a574`), krem metin (`#e8dcc4`).

## Deploy Akışı

```bash
# Değişiklik yap → commit → push
git add index.html
git commit -m "açıklayıcı mesaj"
git push origin main
```

GitHub Pages otomatik 1-2 dakika içinde yayınlar. Cache temizlemek için kullanıcının Safari'yi aşağı çekip yenilemesi gerekebilir.

## İlk Karşılaşma Mesajı

Kullanıcı seninle ilk konuştuğunda projeye hızlıca dahil olduğunu göster:
- "Selam Nify kral, projeyi okudum, hazırım."
- Hangi konu üzerinde çalışmak istediğini sor.
- Önce mevcut `index.html` dosyasını oku (`view` veya `cat index.html`), kodu anla, sonra değişiklik yap.

## Önemli Notlar

- **Kullanıcı geri dönüşü ciddiye al.** "Bug var" diyorsa kesin var, savunmaya geçme.
- **Test etmeden push etme.** Mümkünse önce node script ile syntax kontrolü yap.
- **localStorage versiyon bump unutma.** Şema değişirse `STORAGE_KEY` ve `state.version` ikisini de bumpla, yoksa eski kullanıcılar bozuk state'le açılır.
- **iOS Safari'nin tuhaflıkları** — `requestPermission()` mutlaka button click içinde, scroll behavior için `viewport-fit=cover`, `safe-area-inset-*` paddingleri.

İyi çalışmalar. Nify çok zeki ve direkt bir kullanıcı, lafı uzatmadan iş bitiren tarz seviyor.
