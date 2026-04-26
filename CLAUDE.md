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

### 8. Bildirim sistemi (.ics export)

Kullanıcı arka plan bildirim derdine ana çözüm: **vakit-bazlı kural tanımı + yıllık `.ics` export**, kullanıcı dosyayı iPhone Takvimi'ne import eder, takvim native bildirim sistemini kullanır. Sunucu yok, gizlilik bozulmuyor.

**Kural şeması (`state.notifyRules`):**
```js
[{ id: 'rXXXXX', prayer: 'imsak'|'gunes'|'ogle'|'ikindi'|'aksam'|'yatsi'|'teheccud', offset: -40 /* dk; negatif=önce, pozitif=sonra */ }]
```

**ICS üretimi (`buildIcs(state)`):**
- `yearData.times` üzerinde iterate, her gün × her aktif kural = bir `VEVENT`.
- `getRuleAnchor(prayer, date, yearData)` o günün vakit Date'ini döner. `'teheccud'` özel: ertesi günün imsakı yoksa null, `calcTeheccud(...).lastThirdStart` kullanılır.
- `turkeyToUTC(date)` Türkiye yerel saatini (UTC+3 sabit, DST yok) UTC'ye çevirir; ICS'e `YYYYMMDDTHHMMSSZ` formatında yazılır. Kullanıcı yurt dışında bile olsa Türkiye'ye denk gelen anda hatırlatır.
- Her event'te `VALARM ACTION:DISPLAY TRIGGER:PT0M DURATION:PT1M REPEAT:2` — bildirim 1 dk arayla 3 kez (orijinal + 2 tekrar) gelir. iOS Takvim REPEAT'i destekler.
- Deterministik UID: `teh-{YYYYMMDD}-{ruleId}@teheccudify`. Aynı kullanıcı yeni .ics import edince eski event'lerin UID'leri değişmediği için takvim güncelleyebilir (ama kural ID değişirse yeni UID olur ve duplicate olur — kullanıcıya "eski takvimi sil" rehberi gösteriliyor).
- RFC 5545 escape: `icsEscape()` backslash, virgül, noktalı virgül, newline kaçırır. Satır folding gerekmiyor (her satır <75 oktet).
- Geçmiş günler atlanır (`targetUTC < Date.now() - 1h`).

**UX kararları:**
- 3. tab "Bildirim" — `renderNotify(state)`.
- Kural ekleme formu: vakit dropdown + önce/sonra select + dakika input. Aynı kuralın duplicate'i engelleniyor. Listede vakit sırası + offset'e göre sıralanır.
- "Takvime Aktar" butonu — `Blob` + `URL.createObjectURL` ile dosya indirme.
- **Dürüst rehber kart:** "Bu Saat alarmı değil, Takvim bildirimidir. Sessize alınınca çalmaz." Ayrıca `Ayarlar > Bildirimler > Takvim > Sesler aç + Banner Stili: Sürekli` adımları. Sessizde garantili alarm için iPhone Saat uygulaması yönlendirmesi (Apple programatik Saat alarmı kurmaya izin vermiyor; bunu kabullenip dürüstçe söylemek doğru yaklaşım).
- `completeSetup` lokasyon değişiminde `notifyRules`'ı korur (kullanıcı kurallarını kaybetmesin); ama yeni .ics indirip eski takvimi silmesi gerekir, açıklama UI'da yazılı.

**Eski "Bildirim Kur" butonu Vakitler sekmesinde duruyor** — sayfa açıkken çalışan tek seferlik teheccüd hatırlatıcısı. Yeni .ics sistemi bunu süpersettir ama buton kalsın çünkü "şu an kurmak istiyorum" senaryosu hâlâ geçerli. notify-hint metni kullanıcıyı Bildirim sekmesine yönlendiriyor.

### 9. Kerahat vakitleri
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

### 10. Web Push neden eklenmedi (referans)
iOS 16.4+ web push destekliyor ama VAPID anahtarı + push sunucusu (FCM/APNs) gerekir. Bu proje GitHub Pages'da sunucusuz host ediliyor; backend yok. OneSignal/Firebase gibi 3. taraf servis eklemek hem proje sadelik felsefesine ters hem de gizlilik politikasını bozar ("hiçbir sunucuya kullanıcı verisi gönderilmez" sözünü). `.ics` export çözümü (Bölüm 8) bu kısıtla yaşamanın yolu — sunucu gerektirmiyor, alarm yine native takvim üzerinden çalışıyor.

**Service Worker eklemek** offline cache'i iyileştirir ama bildirim sorununu **çözmez** (push olmadan SW uyutulur). SW konusu açık iş listesinde duruyor; sadece offline UX'i için.

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
**Çözüm:** Segment ismi `'Kuşluk'` olarak değiştirildi. Gerçek kerahat hesabı ayrı bir katman olarak `getKerahatWindows()` ile yapılıyor (Bölüm 9). `prayerList[].seg` mapping de `'Kuşluk'` olarak güncellendi (segments ile match tutması için).

### Problem 10: Yıl son günü öğleden sonra teheccüd vakti saçmalıyordu
**Sorun:** `tomorrowPT` yoksa (örn. 31 Aralık öğleden sonra, 2027 verisi henüz yok), `renderTimes` fallback olarak `calcTeheccud(todayPT, todayPT)` çağırıyordu. Bu durumda `nightMs = todayPT.fajr - todayPT.maghrib` negatif çıkıyor (~-12 saat), `lastThirdStart` saçma değer veriyordu (örn. öğleden sonra 10:00 sabah).
**Çözüm:** Fallback kaldırıldı. `tomorrowPT` yoksa `teh = null` ve kullanıcıya `error-msg` kartı + "Verileri Güncelle" butonu gösteriliyor.

### Problem 11: Tab geçişinde iOS'ta "İzin hatası" mesajı
**Sorun:** `startCompass` her çağrıldığında `DeviceOrientationEvent.requestPermission()` çağırıyordu. Tab geçişinde `renderQibla` `qiblaState.permGranted` true ise `startCompass`'i otomatik tetikliyor — bu user gesture dışında oluyor. iOS'ta `requestPermission` user gesture dışında exception fırlatır; catch bloğu kullanıcıya yanlışlıkla "İzin hatası" mesajı yazar.
**Çözüm:** `requestPermission` çağrısının başına `!qiblaState.permGranted` koşulu eklendi. İzin bir kez alındıktan sonra tab geçişinde tekrar sorulmaz.

### Problem 12: PWA kapalıyken bildirim gelmiyordu
**Sorun:** Eski "Bildirim Kur" butonu sadece `Notification` API + `setTimeout` kullanıyordu. Sayfa kapanınca timer ölüyor, kullanıcı uygulamayı arka planda kapatınca hatırlatma hiç gelmiyordu. Web Push çözümü için sunucu (VAPID + FCM/APNs proxy) gerekli; bu projenin "sunucusuz, gizli" felsefesini bozuyor.
**Çözüm:** **Bildirim sekmesi + `.ics` export** (Bölüm 8). Kullanıcı vakit-bazlı kurallar tanımlar, 365 günlük .ics dosyasını iPhone Takvimi'ne import eder; takvim native bildirim sistemini kullanarak uygulama kapalıyken bile alarmı çalar. Sunucu gerektirmez, gizlilik bozulmaz.

### Problem 13: Takvim "alarm"ının Saat alarmı sanılması (beklenti netliği)
**Sorun:** İlk commit'te kullanıcıya "Takvim alarmı uygulama kapalıyken bile çalar" denildi. Test edince kullanıcı: "alarm çalmıyor sadece bildirim olarak geliyor." Çünkü iOS Takvim'in `VALARM`'ı bildirim kanalını kullanır — Saat alarmı gibi sessize alınmış telefonda çığlık atan bir şey **değildir**. Apple bunu programatik olarak yapmaya izin vermiyor (Saat alarmı sadece elle kurulur).
**Çözüm:**
- `VALARM`'a `REPEAT:2 + DURATION:PT1M` eklendi → bildirim 1 dk arayla 3 kez gelir, kaçırma şansı azalır.
- UI'da dürüst not: "Bu Saat alarmı değil, Takvim bildirimidir. Sessize alınınca çalmaz."
- Rehberde adım: `Ayarlar > Bildirimler > Takvim > Sesler aç + Banner Stili: Sürekli`. Çoğu kullanıcıda bu varsayılan kapalıdır.
- Sessizde garantili çalan alarm için iPhone Saat uygulaması yönlendirmesi.

**Ders:** Beklentiyi over-promise etme. "Bu çözüm şunu yapmaz" demeyi "şunu yapar" demek kadar net yaz.

## Açık İşler / Yapılabilecek Geliştirmeler

Kullanıcı sırasıyla şunlar üzerinde çalışmak isteyebilir (bunları kullanıcı **istediğinde** yap, kendiliğinden başlatma):

1. **Bildirim sistemi v2** — `.ics` export çözümü eklendi (Bölüm 8). Geliştirme alanları: kullanıcı kuralları kurduğunda otomatik .ics yenileme hatırlatması, lokasyon değişiminde "yeni .ics indir + eskiyi sil" akışını otomatikleştirme, multi-konum kuralları.
2. **Service worker eklemek** — Tam offline cache için (font fallback, ikinci-el ziyarette anında açılış). Push bildirim **çözmez** çünkü sunucu yok.
3. **Yıl geçişi otomasyonu** — Şu an 2026 verisi var, 2027 için manuel güncelleme gerekiyor. Yıl geçince otomatik refresh yapılabilir. Aynı zamanda yeni .ics dosyasının da yeniden indirilmesi gerek (kullanıcıya hatırlatma).
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
