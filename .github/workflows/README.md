# Deprem Takip Uygulaması

AFAD, USGS, EMSC ve Kandilli kaynaklarından **seçilebilir** şekilde deprem
verisi çeken, haritada büyüklüğe göre renklendirilmiş ve kümelenen
marker'larla gösteren, etkilenen şehirleri hesaplayan, EMSC için gerçek
zamanlı WebSocket akışı olan bir mobil uygulama + backend aggregator.

## Özellik durumu

| Özellik | Durum |
|---|---|
| AFAD / USGS / EMSC / Kandilli seçilebilir veri kaynağı | ✅ Tamamlandı + test edildi (AFAD+Kandilli artık tek güvenilir API üzerinden) |
| Harita sağlayıcısı seçimi (OSM varsayılan / CartoDB / MapTiler) | ✅ Tamamlandı + test edildi |
| Kaynaklar arası tekilleştirme (dedup) | ✅ Tamamlandı + test edildi |
| Harita (marker + kümeleme + etki alanı daireleri) | ✅ Tamamlandı |
| Deprem detay paneli + etkilenen şehirler | ✅ Tamamlandı + test edildi |
| EMSC gerçek zamanlı WebSocket akışı | ✅ Tamamlandı + test edildi |
| Uygulama içi + native/tarayıcı bildirimi | ✅ Tamamlandı + test edildi |
| Backend cache (deploy sonrası upstream'i yormamak için) | ✅ Tamamlandı + test edildi |
| Uygulama ikonu + splash ekranı | ✅ Üretildi (`mobile/assets/`) |
| Render/Railway deploy konfigürasyonu | ✅ Hazır |
| Gerçek APK dosyası | ❌ Bu ortamda derlenemiyor — Android SDK yok (aşağıya bkz.) |
| Arka planda/uygulama kapalıyken push bildirim | ❌ Firebase kurulumu gerektirir (kendi hesabınız gerekli) |
| Felt reports ("hissettim" bildirimi) | ❌ Eklenmedi — istenirse ayrı bir backend endpoint'i gerekir |

## Klasör yapısı

```
deprem-app/
├── backend/              -> Node.js/Express aggregator API
│   ├── providers/        -> Her kaynak için ayrı modül (AFAD/USGS/EMSC)
│   ├── registry.js       -> Kaynak seçimi + tekilleştirme mantığı
│   └── server.js
└── mobile/               -> Capacitor projesi (APK'ya derlenecek kısım)
    └── www/               -> Harita + liste + ayarlar arayüzü
```

## 1. Backend'i çalıştır

```bash
cd backend
npm install
npm start
# -> http://localhost:3000 üzerinde ayakta
```

Test et:
```bash
curl "http://localhost:3000/earthquakes?sources=AFAD,USGS&minMagnitude=3&hours=24"
```

Backend'i ücretsiz deploy etmek istersen (telefon ile aynı ağda olmak
zorunda kalmamak için): **Render.com**, **Railway.app** veya **Fly.io**
üzerine birkaç dakikada deploy edebilirsin — hepsi Node.js'i native destekler.
Bu repoda her ikisi için de hazır konfigürasyon var (`render.yaml`,
`railway.toml`, `Procfile`).

### Render.com'a deploy (önerilen, ücretsiz plan var)

1. Bu backend klasörünü kendi GitHub reponuza push et.
2. [render.com](https://render.com) → **New +** → **Blueprint** → reponu seç.
   `render.yaml` otomatik algılanır, "Frankfurt" bölgesinde (Türkiye'ye en
   yakın seçenek) bir web servisi kurar.
3. Deploy bitince sana `https://deprem-aggregator-backend.onrender.com`
   gibi bir adres verir. Bunu mobil uygulamadaki **Ayarlar > Backend
   adresi** alanına yapıştır.
4. **Not**: Render'ın ücretsiz planı 15 dakika istek gelmezse sunucuyu
   uyutur, ilk istek 30-50 saniye gecikebilir. Kesintisiz hız istersen
   ücretli plana geçmek veya Railway kullanmak gerekir.

### Railway.app'e deploy (uyku modu yok, aylık ücretsiz kredi var)

1. [railway.app](https://railway.app) → **New Project** → **Deploy from
   GitHub repo** → backend klasörünü içeren reponu seç.
2. `railway.toml` otomatik algılanır, build/start komutları otomatik ayarlanır.
3. **Settings > Networking > Generate Domain** ile public bir URL al.
4. Bu URL'i mobil uygulamanın Ayarlar ekranına gir.

### Deploy sonrası kontrol

```bash
curl https://<senin-adresin>/earthquakes?sources=AFAD,USGS,KANDILLI&minMagnitude=3
```
Boş bir `earthquakes` dizisi de dönse önemli değil (o an büyük deprem
olmayabilir) — asıl kontrol ettiğin şey 200 OK dönmesi ve JSON şeklinin
doğru olması.

## 2. Mobil uygulamayı APK'ya çevir

Bu ortamda gerçek bir APK derleyemiyorum (Android SDK/Gradle yok), ama proje
tamamen hazır — senin Android Studio kurulu bir bilgisayarında şu adımlarla
derlenir:

```bash
cd mobile
npm install
npx cap add android      # android/ klasörünü oluşturur (ilk seferde)
npx cap sync android

# Android Studio ile aç:
npx cap open android
# Android Studio içinde: Build > Build Bundle(s)/APK(s) > Build APK(s)
```

Ya da terminalden direkt debug APK almak istersen:
```bash
cd android
./gradlew assembleDebug
# Çıktı: android/app/build/outputs/apk/debug/app-debug.apk
```

### Backend adresini ayarlama

- **Android emülatörde test**: `www/js/providers-config.js` içindeki
  `apiBase: 'http://10.0.2.2:3000'` zaten emülatörden host makineye
  gitmeni sağlar, değiştirmene gerek yok.
- **Gerçek telefonda test**: Uygulama içindeki Ayarlar sekmesinden backend
  adresini bilgisayarının yerel IP'sine çevir (örn `http://192.168.1.5:3000`)
  — telefon ve bilgisayar aynı Wi-Fi'da olmalı.
- **Yayına alırken**: Backend'i Render/Railway gibi bir servise deploy edip
  oradan gelen `https://...` adresini Ayarlar'a gir.

## 3. Yeni bir deprem sağlayıcısı eklemek

`backend/providers/` altına `base.js`'deki `EarthquakeProvider` sınıfını
extend eden yeni bir dosya ekle, `fetchData()` içinde ortak şemaya
(`registry.js` başındaki yorum) normalize et, sonra `registry.js`
içindeki `ALL_PROVIDERS` listesine ekle. Mobil taraf otomatik olarak
`/providers` endpoint'inden yeni kaynağı çekip Ayarlar ekranında gösterir.

## Türkiye verisi hakkında önemli not (AFAD + Kandilli)

AFAD ve Kandilli verileri, kendi ham scraper'larımız yerine
**[api.orhanaydogdu.com.tr](https://api.orhanaydogdu.com.tr/deprem/api-docs/)**
üzerinden çekiliyor — aktif bakımı yapılan, Swagger dokümantasyonlu, açık
kaynaklı bir topluluk projesi ([GitHub](https://github.com/orhanayd/kandilli-rasathanesi-api)).
Bunu tercih etmemizin sebebi:

- **Tek, temiz JSON kaynağı**: Hem Kandilli hem AFAD verisini aynı şemada,
  MongoDB+Redis cache ile hızlı şekilde sunuyor — kendi yazdığımız düz metin
  parser'ından (Kandilli'nin `lst0.asp` sayfası) çok daha güvenilir.
- **Resmi "en yakın şehir" bilgisi**: `location_properties.closestCity` alanı
  sayesinde depremin gerçek en yakın şehrini, mesafesini ve nüfusunu bize
  hazır veriyor. Mobil uygulamadaki detay panelinde bu bilgi "✓ (kaynak API
  doğrulaması)" etiketiyle, bizim kendi hesapladığımız (81 il merkezine göre
  yaklaşık) listenin üstünde ayrıca gösteriliyor.
- **Sunucu taraflı filtreleme**: `/deprem/data/search` endpoint'i büyüklük ve
  tarih filtresini bizim yerimize sunucuda yapıyor, gereksiz veri çekmiyoruz.

**Bilinmesi gerekenler:**
- **Lisans**: Bu API ticari olmayan kullanım için ücretsizdir. Ticari kullanım
  için `info@orhanaydogdu.com.tr` ile iletişime geçilmesi gerekir. Projenizde
  "Kandilli Rasathanesi API" referansı verilmesi lisans şartıdır (bkz. proje
  README'sindeki lisans bölümü).
- **Rate limit**: IP başına dakikada 40 istek. Backend'imizdeki 20 saniyelik
  cache katmanı (`server.js`) bunu zaten koruyor; deploy ettiğinizde bu
  limiti aşmamak için cache süresini düşürmeyin.
- Bir sorun/kesinti olursa `registry.js`'deki hata izolasyonu sayesinde
  diğer kaynaklar (USGS, EMSC) etkilenmeden çalışmaya devam eder.

Varsayılan olarak Ayarlar ekranında AFAD, USGS ve Kandilli işaretli geliyor;
istersen tek başına Kandilli+AFAD ikilisiyle "sadece Türkiye" moduna da
geçebilirsin.

## Uygulama İkonu ve Splash Ekranı

`mobile/assets/` altında hazır üretilmiş PNG dosyaları var (sismogram dalgası +
episantr halkası temalı, uygulamanın renk paletiyle tutarlı):
- `icon.png` (1024×1024, düz/Play Store için)
- `icon-foreground.png` + `icon-background.png` (Android adaptive icon katmanları)
- `splash.png` / `splash-dark.png` (2732×2732)

Bunları gerçek Android kaynak dosyalarına (`mipmap-*` klasörleri, farklı
yoğunluklar) dönüştürmek için:

```bash
cd mobile
npm install
npx cap add android
npm run assets   # @capacitor/assets tüm boyutları otomatik üretir ve android/ içine yerleştirir
npx cap sync android
```

Markayı değiştirmek istersen `mobile/generate-assets.js` dosyasındaki renk
sabitlerini (`NAVY`, `LOW`/`MID`/`HIGH`/`SEVERE`) veya `waveformPath()`
fonksiyonunu düzenleyip `node generate-assets.js` ile yeniden üret (bu script
`sharp` paketi gerektirir: `npm install sharp --save-dev`).

## Harita Performansı (Marker Clustering)

Aynı bölgede çok sayıda deprem birikirse (özellikle dünya geneli görünümde)
`leaflet.markercluster` ile otomatik kümeleniyor. Küme ikonunun rengi, içindeki
en yüksek büyüklüğe göre belirleniyor — yani bir kümeyi açmadan da içinde
"önemli" bir deprem olup olmadığını renkten anlayabilirsin. Etki alanı
daireleri kümeye dahil edilmiyor (ayrı katmanda), böylece harita hem
performanslı hem okunaklı kalıyor.

## Bu sürümde eklenen özellikler (LastQuake/EMSC gibi referans uygulamalar incelenerek)

- **EMSC gerçek zamanlı WebSocket akışı** (`js/emsc-realtime.js`): EMSC seçiliyse
  60 saniyelik polling'i beklemeden `wss://www.seismicportal.eu/standing_order/websocket`
  üzerinden anlık deprem bildirimleri alınır — bu, referans aldığımız LastQuake
  uygulamasının "gerçek zamanlı" hissini veren temel mekanizmalardan biri.
- **Deprem detay paneli** (bottom sheet): Haritada veya listede bir depreme
  dokununca büyüklük, derinlik, tahmini etki yarıçapı ve **en yakın/etkilenen
  şehirler** (81 il merkezine göre mesafe hesaplı) gösterilir — orijinal
  istekteki "konumunu etkilediği şehir bilgileri" özelliği bu.
- **Deprem güvenlik önerileri** ekranı (LastQuake'in "post-earthquake safety
  tips" özelliğinden esinlenilmiştir).
- **Yeni deprem bildirimi**: eşik üstü ve yakın zamanlı depremler için hem
  native/tarayıcı bildirimi hem de her durumda çalışan uygulama içi banner.

## Harita Sağlayıcısı

**Güncelleme (Ağustos 2026):** CARTO, ücretsiz raster tile servisini de API
key zorunluluğuna bağladı — önceden anahtarsız çalışan `basemaps.cartocdn.com`
artık key olmadan "API KEY REQUIRED" filigranıyla dönüyor. Bu yüzden
varsayılan sağlayıcıyı **gerçekten anahtarsız olan standart OpenStreetMap**
tile sunucusuna çevirdik.

- **Varsayılan: OpenStreetMap** (`tile.openstreetmap.org`) — hiçbir kurulum
  gerektirmez, tamamen ücretsiz. Tile'lar doğası gereği açık renkli
  geldiğinden, koyu temayla uyumlu olması için CSS `invert` filtresi
  uyguluyoruz (`osm-dark-invert` sınıfı, `style.css` içinde) — Home
  Assistant'ın harita bileşeninin de kullandığı yöntem.
- **Opsiyonel: CartoDB** — artık ücretsiz bir key gerektiriyor:
  [carto.com/basemaps/apikey](https://carto.com/basemaps/apikey) üzerinden
  saniyeler içinde alınabilir (aylık 5 milyon istek ücretsiz, kredi kartı
  istemiyor). Gerçek koyu tile'lar (invert hilesi olmadan) ister isen bu
  seçenek daha temiz görünür.
- **Opsiyonel: MapTiler** — kendi key'inle (aylık 100.000 yükleme ücretsiz).

Ayarlar ekranında CartoDB veya MapTiler seçilip key girilmezse, harita boş/
filigranlı kalmasın diye otomatik olarak OpenStreetMap'e düşer
(`map.js` içindeki `resolveTileConfig` fallback mantığı test edildi).

## Notlar

- Haritadaki "etki alanı" daireleri (`map.js` içindeki
  `approxFeltRadiusKm`) **bilimsel bir sismik model değildir** — sadece
  büyüklüğe göre kaba bir görsel ölçek verir. Gerçek hissedilme alanı
  derinlik, zemin yapısı ve yöne göre çok değişir.
- **Bildirimler yalnızca uygulama açıkken/arka planda çalışırken** (polling
  veya EMSC WebSocket sırasında) tetiklenir. Uygulama tamamen kapalıyken
  de bildirim almak (gerçek "push") için backend'in Firebase Cloud
  Messaging'e event gönderdiği ayrı bir sunucu bileşeni ve bir Firebase
  projesi (google-services.json) gerekir — bu depoda yok, çünkü bu sizin
  kendi Firebase hesabınıza bağlı bir kurulum gerektiriyor.
- AFAD ve Kandilli verisi artık `api.orhanaydogdu.com.tr` üzerinden çekiliyor
  (bkz. yukarıdaki "Türkiye verisi hakkında önemli not" bölümü); bu API'nin
  şema veya endpoint değişikliği yapması ihtimaline karşı `turkeyApiProvider.js`
  içindeki `normalize()` fonksiyonunun güncellenmesi gerekebilir.
