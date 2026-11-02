This is an Expo/React Native mobile application. Prioritize mobile-first patterns, performance, and cross-platform compatibility.

## Expo has changed — do not trust your training data

Expo ships breaking changes every SDK release. APIs you remember are likely renamed, moved, or removed. Before writing any code that touches an Expo, EAS, or React Native API:

1. Read the major version of the `expo` package in `package.json`.
2. Fetch the matching versioned docs: `https://docs.expo.dev/versions/v<major>.0.0/`
3. For anything else, fetch https://docs.expo.dev/llms.txt — an index of all Expo docs with corrections to common LLM misconceptions. Follow its links to the specific page you need; never answer from memory.

## Commands

Use `bunx` instead of `npx` if the project uses bun (`bun.lock` present).

```bash
npx expo install <package>  # ALWAYS use instead of npm/yarn/pnpm/bun add — resolves SDK-compatible versions
npx expo start              # start the dev server
npx expo lint               # lint
npx tsc --noEmit            # typecheck
npx expo-doctor             # diagnose dependency and config issues
npx expo install --fix      # fix incompatible package versions
```

Run lint and typecheck before declaring any task done.

## Navigation & Routing

- Use **Expo Router** for all navigation. Routes live in `src/app/` — every file there is a screen, `_layout.tsx` files define navigators. Keep non-route code (components, hooks, utils) outside `src/app/`.
- Import `Link`, `router`, and `useLocalSearchParams` from `expo-router`.
- Docs: https://docs.expo.dev/router/introduction.md

## Building with EAS

Use EAS to build, sign, and submit the app in the cloud (`eas build`, `eas submit`) and to ship over-the-air updates (`eas update`) — no local Xcode or Android Studio required. Run EAS CLI as `bunx eas-cli <command>` in Bun projects, or `npx eas-cli@latest <command>` otherwise; substitute that for bare `eas` in docs examples.
Docs: https://docs.expo.dev/eas/index.md

## Rules

- If `ios/` and `android/` directories do not exist, they are generated (Continuous Native Generation). Never create or edit them by hand — configure native behavior in `app.json` and config plugins.
- Expo Go only includes its bundled native modules. After adding a library with native code, the app needs a development build: `npx expo run:ios|android` locally, or `eas build --profile development`.
- Prefer recommended Expo modules over third-party libraries, and check your available skills before adding dependencies. Docs: https://docs.expo.dev/versions/latest/index.md


---

# AGENTS.md — QrScannerApp (öğrenci + kiosk uygulaması)

Önce kökteki [`../AGENTS.md`](../AGENTS.md) dosyasını okuyun: protokol, API sözleşmesi, iş kuralları ve **referans kaynak kodu (§0)** oradadır ve **bağlayıcıdır**.
Bu dosya uygulamanın **her ekranını, her `lib` dosyasını ve kurallarını** anlatır.

**Uyarlamalar:** kök AGENTS.md §0.2'deki farkları uygulayın
(uygulamada: görünen okul adı `İzmir Fen`, logo harfleri `İF`, APK adı `izmir-fen-kiosk.apk`; geri kalan her şey aynı).

**Prompt önceliği:** görev referansta olmayan bir ekran/özellik veya var olan bir ekranın değişmesini isterse prompt esastır (kök AGENTS.md §0.3). Değişmeyenler: BLE yapısı (`lib/ble.ts` tarama modeli, `modules/kiosk-beacon` yayını, jeton biçimi, servis UUID), sunucuyla yalnızca `lib/api.ts` → `/api/mobile/*` üzerinden konuşma, `protocol.ts`, `device.ts` cihaz kimliği/güvenli depo anahtarları, kioskta QR'ın cihazda üretilmesi. Yeni ekran eklenirse §2 ağacı ve §4 tablosu aynı işte güncellenir.

## 1. Teknoloji

| Konu | Seçim |
| --- | --- |
| Çatı | Expo SDK 57, React Native 0.86, React 19, TypeScript strict, **Expo Router** (dosya tabanlı; ekranlar `src/app/`) |
| Kamera / QR okuma | `expo-camera` (`CameraView`, `useCameraPermissions`, `barcodeScannerSettings: { barcodeTypes: ["qr"] }`) |
| QR gösterme | `react-native-qrcode-svg` (+ `react-native-svg`) |
| Kripto | `@noble/hashes` (HMAC-SHA256; içe aktarma yolları `@noble/hashes/hmac.js`, `sha2.js`, `utils.js`), `expo-crypto` (rastgele bayt, UUID) |
| Güvenli depo | `expo-secure-store` (`AFTER_FIRST_UNLOCK_THIS_DEVICE_ONLY`) |
| BLE tarama | `react-native-ble-plx` (yalnızca geliştirme/mağaza derlemesinde; **Expo Go'da yok** → koşullu `require`) |
| BLE yayını | yerel Expo modülü `modules/kiosk-beacon` (Android Kotlin; iOS "desteklenmiyor") — içe aktarma `@modules/kiosk-beacon` |
| Ekran açık kalsın | `expo-keep-awake` (`useKeepAwake`, kiosk) |
| İkonlar | `react-native-svg` ile `src/components/icons.tsx` |
| Derleme ayarı | `app.config.ts` (hazır; `expo-build-properties`: adres `http://` ise Android düz HTTP izni) |

Tüm paketler **kurulu**; yeni paket eklenmez. Yol takma adları (`tsconfig.json`, hazır): `@/*` → `src/*`, `@modules/*` → `modules/*`.
Sunucu adresi: `.env` → `EXPO_PUBLIC_API_URL` (derlemede uygulamaya gömülür; değişince Metro'yu yeniden başlatın). Canlıda `https://…`.

## 2. Klasör yapısı (hedef ağaç)

✅ = depoda hazır · ⏳ = ekip geliştirecek. Ekran yolları (Expo Router) ve dosya adları değiştirilmez.

```
QrScannerApp/
├── AGENTS.md, CLAUDE.md ✅ · README.md ⏳ · CIHAZ_TESTI.md ⏳ · TESTFLIGHT.md ⏳
├── package.json ✅ · app.json ✅ · app.config.ts ✅ · tsconfig.json ✅ · eslint.config.js ✅ · .env.example ✅ · assets/ ✅ (uygulama adı `DOQR`; ikonlar + logo.png)
├── eas.json ✅                     profiller: development · preview · production · kiosk-lan (gerçek IP/kimlik yazılmaz)
├── modules/kiosk-beacon/ ✅        Android BLE yayın modülü
│   ├── index.ts                           KioskBeacon { isSupported, lastError, start, stop }
│   ├── expo-module.config.json
│   ├── android/build.gradle · android/src/main/AndroidManifest.xml
│   ├── android/src/main/java/expo/modules/kioskbeacon/KioskBeaconModule.kt
│   └── ios/KioskBeaconModule.swift · ios/KioskBeacon.podspec
├── tools/                                 mac-kiosk-beacon.swift · ios-kur.sh · android-apk.sh ⏳
└── src/
    ├── app/                               EKRANLAR
    │   ├── _layout.tsx ✅ (SessionProvider + Stack)
    │   ├── index.tsx ✅              açılış (logo + oturum kontrolü)
    │   ├── welcome.tsx ✅            öğrenci / yönetici seçimi
    │   ├── student-login.tsx ✅      telefon → şifre oluştur | şifre
    │   ├── student-home.tsx ✅       son hareketler, "QR okut", çıkış yap
    │   ├── scan.tsx ✅       kapıdaki ilk okutmada yön seçimi → kamera (+BLE) → sonuç
    │   ├── admin-login.tsx ✅        kiosk hesabıyla giriş
    │   ├── kiosk-select.tsx ✅       kiosk seçimi
    │   ├── kiosk/[id].tsx ✅ dönen QR + BLE yayını + son okutan
    ├── components/
    │   ├── BrandHeader.tsx           BrandHeader — karşılama ekranının ana renk üst alanı (uygulama logosu + okul amblemi + başlık + slogan)
    │   ├── FeedbackSheet.tsx         öğrenci ana ekranındaki geri bildirim penceresi
    │   ├── ui.tsx ✅                 mono, Logo, SchoolCrest, useKeyboardHeight, Screen, Title, Button, Field, ErrorBox, InfoBox, TopBar, KeyboardForm, TextLink, LegalLinks, OptionCard, OptionRow, styles
    │   ├── icons.tsx ✅              IconBack, IconCheck, IconIn, IconOut, IconQr, IconBluetooth, IconBluetoothOff, IconInfo, IconAlert, IconX, IconPhone, IconShield, IconMail, IconLogout, IconDoor, IconMessage, IconRefresh, IconClock
    │   └── BluetoothGate.tsx ✅      Bluetooth kapalı/izin yok uyarısı (QR okutma ve kiosk ekranlarının altında)
    └── lib/
        ├── config.ts ✅ API_URL, SPLASH_DURATION_MS, colors
        ├── api.ts ✅                 ApiError, api<T>(), yanıt tipleri
        ├── device.ts ✅              getDeviceIdentity, clearBinding, tokens
        ├── session.tsx ✅            SessionProvider, useSession
        ├── protocol.ts ✅            PROTOCOL_VERSION, slotAt, buildQrPayload, isOurQr, bleToken, requestSignature, __test
        └── ble.ts ✅                 bleLog, getBleLog, clearBleLog, subscribeBleLog, isBleAvailable, subscribeBtState, openBluetoothSettings, extractKioskToken, startBleScan (+ tipler BtState, Found, BleStatus, BleLogEntry, SeenDevice)
```
`ios/` ve `android/` klasörleri **üretilir** (git'e girmez); elle oluşturulmaz/düzenlenmez.

**Katmanlar:** ekran (`src/app/`) yalnızca gösterir ve kullanıcı etkileşimini yönetir → sunucuyla konuşma yalnızca `lib/api.ts` üzerinden → gizli değerler yalnızca `lib/device.ts` → protokol hesapları yalnızca `lib/protocol.ts` → Bluetooth yalnızca `lib/ble.ts` / `modules/kiosk-beacon`.

## 3. `lib` dosyaları

### `config.ts`
`API_URL` = `EXPO_PUBLIC_API_URL` ?? `expoConfig.extra.apiUrl` ?? `http://localhost:3000` (sondaki `/` silinir).
`colors` anahtarları (tek koyu yeşil + sıcak sarı vurgu + beyaz zemin; kırmızı yalnızca uyarı/hata): `brand #006160, brandDark #004746, brandSoft #DDEFEE, accent #F5B73D, ink #14211A, inkSoft #3D433F, muted #6B7570, line #E2DFD6, lineSoft #ECEAE3, lineStrong #A8A294, bg #FFFFFF, danger #D04A3B, dangerSoft #FCE8E6, dangerInk #8F1C13, dangerLine #F1B8B2, warn #9A5B00, warnSoft #FBF0DC, warnInk #7A4700, white #ffffff`. `SPLASH_DURATION_MS = 1000` (açılış ekranı süresi).

### `api.ts`
- `class ApiError(status, message, code)`.
- `api<T>(path, { method = "GET", body, token, timeoutMs = 15000 })`: `fetch(API_URL + path)`, başlıklar `Accept: application/json`, gövde varsa `Content-Type: application/json`, token varsa `Authorization: Bearer`. `AbortController` ile zaman aşımı.
  - Ağ hatası → `console.warn("[api] <METHOD> <path> başarısız:", e)` sonra `ApiError(0, "Sunucuya ulaşılamadı. İnternet bağlantınızı kontrol edin.", "NETWORK")`; zaman aşımı (`AbortError`) → `ApiError(0, "Sunucu zamanında yanıt vermedi. Tekrar deneyin.", "TIMEOUT")`.
  - Başarısız yanıt → `ApiError(res.status, data.error ?? "Beklenmeyen hata (<status>).", data.code ?? "ERROR")`.
- Yanıt tipleri (kök AGENTS.md §5 ile aynı): `StudentSummary, AuthResponse, MeResponse, ScanResponse, AppConfig, KioskInfo, KioskStart, KioskFeed` (`KioskFeed` yalnızca `serverTime` ve `events` içerir).

### `device.ts`
Güvenli depo anahtarları: `fb.deviceId, fb.deviceSecret, fb.boundEmail, fb.studentToken, fb.adminToken` (protokol; değiştirmeyin).
- `getDeviceIdentity()` → `{ deviceId, deviceSecret }`; ilk çağrıda `Crypto.randomUUID()` + 32 rastgele bayt (hex, 64 karakter) üretilip saklanır, bellek içinde önbelleğe alınır, **bir daha değişmez**.
- `clearBinding()` → `fb.boundEmail` ve `fb.studentToken` silinir (cihaz kimliği silinmez).
- `tokens` → `getStudent/setStudent/clearStudent/getAdmin/setAdmin/clearAdmin`.

### `session.tsx`
`SessionProvider` + `useSession()` → `{ ready, studentToken, adminToken, student, config, setStudentSession(token, student), setStudent, clearStudentSession, setAdminSession(token), clearAdminSession, refreshConfig }`.
Açılışta iki token depodan okunur → `ready = true` → arka planda `GET /api/mobile/config` (hata olursa `config` null kalır; uygulama yine açılır). Context dışında kullanılırsa "useSession, SessionProvider içinde kullanılmalı" hatası.

### `protocol.ts`
Sunucudaki `AdminPanel/src/lib/protocol.ts` ile **aynı kurallar** (kök AGENTS.md §4): `PROTOCOL_VERSION = "FB2"`, `slotAt(nowMs, slotSeconds)`, `buildQrPayload(secretHex, kioskId, slot)` (base64url elle, dolgusuz), `isOurQr(payload)` (`"FB2."` ile başlar ve 4 parça), `bleToken`, `requestSignature(deviceSecretHex, qr, ble|null, timestamp)`, `__test = { base64url }`.
Test vektörleri: kök AGENTS.md §4 (sunucu ve mobil aynı sonucu vermeli).

### `ble.ts`
- `react-native-ble-plx` **koşullu `require`** ile yüklenir (Expo Go'da yoksa `available = false`, uygulama çökmez, durum `"unavailable"`).
- Log: `bleLog(msg)` → `[BLE] ` önekiyle `console.log` + 300 satırlık bellek arabelleği; `getBleLog`, `clearBleLog`, `subscribeBleLog`.
- `subscribeBtState(fn)` (ilk değer hemen gelir; iOS'ta ilk çağrı izin penceresini açar), `openBluetoothSettings(state)` (Android: kapalıysa `REQUEST_ENABLE` isteği, değilse uygulama ayarları; iOS: uygulama ayarları).
- Android izinleri: API ≥ 31 → `BLUETOOTH_SCAN` + `BLUETOOTH_CONNECT`; daha eski → `ACCESS_FINE_LOCATION`.
- `extractKioskToken(device, serviceUuid)`: önce `serviceData[serviceUuid]` (base64 → hex, 16 karakter → `source: "service-data"`), sonra yerel ad `^FB([0-9a-fA-F]{16})$` (`source: "local-name"`).
- `startBleScan(serviceUuid, onUpdate, { onDevices?, verbose? })` → durdurma fonksiyonu döner. **Filtresiz** tarama (`startDeviceScan(null, { allowDuplicates: true }, …)`), Bluetooth açılınca başlar/kapanınca durur, 5 sn'de bir özet logu, `onDevices` verilirse 1 sn'de bir cihaz listesi (kiosklar önce, sonra RSSI'ye göre). Durumlar: `unavailable | no-permission | off | scanning | found`.

## 4. Ekranlar (akış ve metinler referansla aynı)

Tüm ekranlar `Screen` içinde; üst gezinme `TopBar` (solda geri oku, sağda "Adım n / m"). Ana düğme hep altta (klavye açıksa klavyenin üstünde), ikincil seçenekler `TextLink`. Giriş formları (`student-login`, `admin-login`) `KeyboardForm` (`ui.tsx`) kullanır: alanlar kayan bölümde, ana düğme (ve altındaki bağlantılar) kayan bölümün dışında altta sabit; klavye açılınca düğme klavyenin hemen üstüne çıkar, alanlar onun üstünde kalır. Klavye yüksekliği `Keyboard` olaylarından okunur (iOS'ta alt güvenli alan düşülür; Android'de RN gezinme çubuğunu zaten düşer) — `KeyboardAvoidingView` ekran açılış animasyonunda kendi konumunu yanlış ölçebildiği için kullanılmaz. `keyboardShouldPersistTaps="always"`: dokunuş klavyeyi kapatıp yerleşimi kaydırmasın. İlk adımda geri oku, önceki ekran yoksa `/welcome`'a gider. Hatalar `ErrorBox`, bilgiler `InfoBox`. Sunucu hata mesajları (`e.message`) **aynen** gösterilir.

| Ekran | API | Davranış ve metinler |
| --- | --- | --- |
| `index.tsx` Açılış | — | Yalnızca `Logo` (ana renk zeminde, yazı yok). Oturum `ready` olunca ve en az `SPLASH_DURATION_MS` geçince: `adminToken` → `<Redirect href="/kiosk-select" />`; `studentToken` → `/student-home`; yoksa `/welcome` |
| `welcome.tsx` Karşılama | — | `Screen` kullanmaz. Üstte ekran kenarına (durum çubuğunun arkasına) uzanan, alt köşeleri yuvarlak ana renk alan (durum çubuğu yazıları ekran odaktayken beyaz): solda `Logo`, sağda beyaz daire içinde aynı boyutta (64) okul amblemi (`SchoolCrest`, `assets/ifl-logo.png`); büyük "İZMİR FEN LİSESİ", büyük başlık "Hoş geldiniz", altında sarı italik slogan “Her Zaman, Her Alanda En İyisi”. Ortada (kayar) "Öğrenci giriş-çıkış uygulaması" / "Uygulamada sizi bu adımlar bekliyor:" ve numaralı, dikey çizgiyle birbirine bağlı 3 adım (zaman çizelgesi; seçenek gibi görünmez): "Telefonunuzla giriş yapın", "Kapıdaki QR kodu okutun", "Veliniz bilgilendirilir". Altta sabit **Öğrenci girişi** → `/student-login`; altında küçük bağlantı "Yönetici olarak devam et" → `/admin-login`; en altta `LegalLinks` ("Gizlilik Politikası" · "Destek" → `API_URL/gizlilik`, `/destek`; mağaza kuralı — uygulama içinde yalnızca burada) |
| `student-login.tsx` | `POST /student/check-phone`, `/student/set-password`, `/student/login` | Adımlar `phone → create-password | password` (aynı ekran); TopBar sağında "Adım 1 / 2" (telefon) ve "Adım 2 / 2" (şifre). **Telefon**: başlık "Telefon numaranızı girin" (alt başlık yok), alan "Cep telefonu" (ipucu `05XX XXX XX XX`, rakam klavyesi, açılınca odakta, boşlukları kendisi ekler), **Devam** numara `05XXXXXXXXX` / `5XXXXXXXXX` / `905XXXXXXXXX` olana kadar pasif (gri); numara gönderilirken boşluklarıyla gider, biçimi sunucu çevirir; diğer girişe geçiş ve gizlilik/destek bağlantısı yok (geri oku karşılama ekranına döner). **Şifre oluşturun**: InfoBox "Şifreyi kaydettiğinizde hesabınız bu telefona bağlanır. Başka telefondan girmek için okul yönetimine başvurmanız gerekir.", alanlar "Şifre (en az 8 karakter)" + "Şifre (tekrar)", canlı kural satırları "En az 8 karakter" / "İki şifre aynı" (yuvarlaksız; sağlanınca yanında tik, renk ana renk), **Şifreyi kaydet ve giriş yap** iki kural sağlanana kadar pasif; gönderilen `platform` = `ios`/`android`/`unknown`. **Şifre**: başlık "Şifrenizi girin", alan "Şifre", hemen altında "Şifrenizi unuttuysanız okul yönetiminden sıfırlama isteyin.", **Giriş yap** şifre boşken pasif. Başarı → `setStudentSession` → `router.replace("/student-home")`. Geri: telefon adımındaysa önceki ekran, değilse telefon adımı. Sunucu hatası yoksa "Beklenmeyen bir hata oluştu.". **Yerel "bu telefon başka hesaba bağlı" kontrolü yapılmaz** |
| `student-home.tsx` | `GET /student/me` | Ekran her odaklandığında (`useFocusEffect`) yüklenir; aşağı çekince yenile. En üstte solda `Logo`, sağda küçük çıkış ikonu; altında "Merhaba," + ad soyad. Çıkış ikonu (onay: "Çıkış yap" / "Uygulamadan çıkış yapılsın mı? Tekrar girişte şifreniz sorulur." / "Vazgeç") → oturumu sil → `/`. **Durum kartı yoktur** (öğrencinin içeride/dışarıda olduğu gösterilmez). "Son hareketler": Giriş/Çıkış ikonlu satırlar, kiosk adı veya **"Manuel kayıt"**, tarih-saat (`gg.aa ss:dd`); boşsa "Henüz hareket yok" / "Giriş ve çıkışlarınız burada görünecek.". Altta tam genişlik düğme **"QR okut"** (alt yazı yok; yön kapıya göre değiştiği için okutmadan önce gösterilmez) → `/scan`, altında ikincil düğme **"Geri bildirim gönder"**; iki düğme kayan listenin dışında sabittir (liste uzasa da kapanmaz). Geri bildirim ekran üstünde alttan açılan pencerede (`FeedbackSheet`): "Geri bildirim", "Uygulamayla ilgili sorun veya önerinizi yazın. Okul yönetimi inceleyecek.", çok satırlı alan (en fazla 1000 karakter), **Gönder** (boşken pasif) / **Vazgeç**; arka plana dokunma veya Android geri tuşu kapatır; klavye açılınca pencere klavyenin üstüne çıkar; `POST /api/mobile/feedback` başarılıysa kapanır ve "Teşekkürler" / "Geri bildiriminiz okul yönetimine iletildi." uyarısı; hata mesajı pencerede. `DEVICE_REVOKED` → `clearBinding` + oturumu sil + uyarı "Cihaz bağlantısı kaldırıldı" / "Okul yönetimi bu telefonun bağlantısını kaldırdı. Telefon numaranızla giriş yapıp yeni şifre oluşturabilirsiniz." → `/`. Diğer 401 → oturumu sil → `/student-login`. Diğer hata: mesaj veya "Bilgiler yüklenemedi." |
| `scan.tsx` | `GET /student/me`, `POST /scan` | Ayrıntı §4.1 |
| `admin-login.tsx` | `POST /admin/login` | İki adım: **e-posta** ("E-postanızı girin", alan "E-posta", **Devam** geçerli e-posta yazılana kadar pasif) → **şifre** ("Şifrenizi girin", alan "Şifre", **Giriş yap**); TopBar sağında "Adım 1 / 2" / "Adım 2 / 2", açılınca alan odakta. Geri: ikinci adımdaysa e-posta adımı. Başarı → `setAdminSession` → `router.replace("/kiosk-select")`. Hata: mesaj veya "Giriş yapılamadı.". Geniş ekranda (≥ 720 px, tablet) solda ana renk panelde `Logo` ve "Bu tableti kapı ekranına dönüştürün", sağda form |
| `kiosk-select.tsx` | `GET /kiosks` | Telefonda tek sütun: üstte `Logo`, "Kiosk kurulumu", sağda yenile ve çıkış ikonları (yenile: kiosk listesini yeniden yükler — panelde yeni eklenen kiosk uygulamadan çıkmadan gelir; yüklenirken dönen gösterge; seçili kiosk listede yoksa seçim ilk kioska döner; çıkış: onay "Çıkış yapılsın mı?" / "Yönetici oturumu kapatılacak.") → oturumu sil → `/`; başlık "Bu tablet hangi kapıda duracak?", "Seçtiğiniz kapı için QR ekranı açılır. Ekran kapanmaz.". Geniş ekranda (≥ 720 px, tablet) solda ana renk panelde `Logo`, "Kiosk kurulumu", başlık ve alt başlık; sağda üstte yenile + çıkış ikonları, altında kapı listesi. Her kiosk bir `OptionRow` (kapı ikonu, ad, "Giriş ve çıkış", seçim yuvarlağı; ilk kiosk seçili gelir). Altında "Tablet şarja takılı ve Bluetooth açık kalmalı." ve **QR ekranını başlat** → `/kiosk/[id]`. 401 → oturumu sil → `/admin-login`. Hata: "Kiosklar yüklenemedi." |
| `kiosk/[id].tsx` | `POST /kiosks/:id/start`, `GET /kiosks/:id/feed` | Ayrıntı §4.2 |

### 4.1 `scan.tsx` (öğrenci QR okutma)
- Durumlar (`phase`): `scan | sending | done | error`. `busy` ref'i ile aynı anda tek istek.
- Kamera izni ayrı ekranda değil, ekran açılınca doğrudan sistem penceresiyle bir kez istenir. Verilmezse kamera yerine koyu alan ve altında "Kamera izni gerekli. QR kodu okutmak için ayarlardan kamera iznini açın." + **Ayarları aç**; Ayarlar'dan dönünce durum yenilenir.
- Açılışta `GET /student/me` → `needsDirection`. **`needsDirection` (hiç kaydı yok) veya okutma `DIRECTION_REQUIRED` döndüyse (bu kapıda ilk okutma) ve yön seçilmemişse**: yalnızca geri oku; "Şu an ne yapıyorsunuz?", "Bu kapıda ilk kez okutuyorsunuz. Bir kez seçmeniz yeterli.", iki `OptionRow` **Okula giriyorum** (varsayılan) / **Okuldan çıkıyorum**, altta seçime göre **Girişi kaydet** / **Çıkışı kaydet**.
- Kamera ekranı (koyu zemin): üstte geri oku (yön rozeti yok; her kapının durumu ayrı olduğundan yön okutmadan sonra sonuç ekranında görünür), "Kapıdaki ekranda görünen QR kodu çerçeveye alın", "Kod kendiliğinden okunur.", çerçeve köşeleri, gönderilirken "Doğrulanıyor…". Altta küçük BLE durum satırı (`unavailable` "Bluetooth doğrulaması bu sürümde kapalı", `scanning` "Kapı sinyali aranıyor · Kapıya yaklaşın", `found` "Kapı sinyali alındı"; Bluetooth kapalı/izin yoksa satır yerine `BluetoothGate` uyarısı), uyarı kutusu (`hint`). Vazgeç düğmesi yoktur; çıkış geri okuyladır.
- BLE: ekran açıkken `startBleScan(config.bleServiceUuid, …)`; son jeton bir ref'te tutulur. QR okununca jeton **8 sn'den yeniyse** (`BLE_FRESH_MS = 8000`) gönderilir.
- QR `isOurQr` değilse istek gitmez, `hint` = "Bu QR kod okul kapısına ait değil.". `config.bleRequired` ve taze jeton yoksa istek gitmez, `hint` = "Kapı sinyali bekleniyor. Bluetooth'un açık olduğundan emin olup kapıya yaklaşın.".
- İstek: `{ qr, ble: taze ? { token, rssi } : null, timestamp: Date.now(), signature: requestSignature(deviceSecret, qr, token|null, timestamp), direction: chosenDirection }`.
- `DIRECTION_REQUIRED` → yön seçimine dön (`hint` = sunucu mesajı). Diğer hata → hata ekranı: "Okutma kabul edilmedi" + mesaj + **Tekrar dene** / Geri.
- Başarı ekranı (giriş ve çıkış aynı ana renk zeminde): ✓, "Giriş kaydedildi" / "Çıkış kaydedildi", büyük saat (`result.time`) + "Saat", "Bluetooth doğrulaması: sinyal yok / kapı doğrulandı ✓ / doğrulanamadı ✗", `duplicate` ise "Bu işlem az önce zaten kaydedilmişti; velinize tekrar WhatsApp mesajı gönderilmedi.", **Tamam** → geri.
- Her adım `bleLog(...)` ile loglanır (QR okundu + gönderilecek jeton, sunucu kabul/red).

### 4.2 `kiosk/[id].tsx` (kiosk ekranı)
- `useKeepAwake()`. Açılışta `POST /kiosks/:id/start` → `offset = serverTime - Date.now()`, `since = serverTime (ISO)`. 401 → yönetici oturumunu sil → `/admin-login`; diğer hata: mesaj veya "Kiosk başlatılamadı." + Geri.
- 250 ms'de bir saat; `serverNow = now + offset`, `slot = slotAt(serverNow, slotSeconds)`. QR = `buildQrPayload(secret, kioskId, slot)` — **cihazda üretilir**, internet kesilse de sürer.
- Yerleşim: yatay ekranda solda yeşil bilgi paneli + sağda QR; dikeyde üstte panel. Panel yukarıdan aşağıya: (1) üst satır — solda yuvarlak **geri düğmesi** (dokununca onay "Kiosk modundan çık" / "QR ekranı kapatılsın mı?" → geri), kapı adı (büyük harf, küçük yazı), saat (ss:dd); (2) öğrenci kartı alanı (kart yokken boş); (3) tablette ve yatay telefonda 3 adım ("Okul uygulamasını açın", "\"QR okut\"a basıp bu kodu okutun", "Adınızı ekranda görünce geçin"); (4) en altta ince bir çizgiyle ayrılmış iki durum satırı: "Sunucu bağlantısı var" / "Sunucuya ulaşılamıyor" ve Bluetooth durumu. Ekran `Screen` kullanmaz: yeşil panel ekranın kenarlarına kadar uzanır, güvenli alan (çentik, ana ekran çizgisi) payı `useSafeAreaInsets` ile panelin ve QR alanının iç boşluğuna eklenir (dışarıda beyaz şerit kalmaz). Başlık ("GİRİŞ · ÇIKIŞ"), okul adı, kod numarası (`#…`) ve açıklama paragrafı yoktur.
- QR: `react-native-qrcode-svg`, `ecl="M"`, `quietZone={8}`; boyut QR alanının ölçülen boyutundan (`onLayout`) hesaplanır (en fazla telefonda 480, tablette 720). Kısa kenarı ≥ 600 olan ekran tablet sayılır: büyük yazılar, adımlar dikeyde de görünür; telefonda yazılar küçülür, adımlar yalnızca yatayda (küçük) görünür. QR'ın altında yalnızca ince kalan süre çubuğu vardır (yazı yok); çubuk dilimin kalan süresi boyunca doğrusal ve akıcı azalır (saat adımlarıyla değil), sunucu saatiyle her yoklamada yeniden eşitlenir.
- Akış: 3 sn'de bir `GET /kiosks/:id/feed?since=` (zaman aşımı 5 sn) → `offset` güncellenir; daha önce gösterilmemiş olaylar zaman sırasıyla bir kuyruğa girer; ekranda tek kart görünür (solda renkli yan panel: giriş için ana renk + beyaz giriş oku, çıkış için kırmızı + beyaz çıkış oku — yazı yok; sağda ad soyad ve altında sınıf; sınıf boşsa gösterilmez; saat yoktur, üstte zaten var; profil resmi yoktur, şemada fotoğraf alanı da yok): kart sağdan girer (300 ms, hafif büyüyerek ve belirerek), 4 sn kalır, sola çıkar (küçülerek ve solarak), ardından sıradaki gelir. Hata → "Sunucuya ulaşılamıyor" (kırmızı nokta), başarı → "Sunucu bağlantısı var".
- BLE yayını: her dilimde `KioskBeacon.start(bleServiceUuid, bleToken(secret, kioskId, slot))`; Android 12+ (API ≥ 31) `BLUETOOTH_ADVERTISE` + `BLUETOOTH_CONNECT` izni bir kez istenir. Durum metinleri: `off` "Bluetooth kapalı veya yayın kullanılamıyor" (iOS: "BLE yayını yok"), `starting` "Bluetooth başlatılıyor", `on` "Bluetooth yayınlanıyor", `unsupported` "Bluetooth yayını yok (iOS kiosk)", `error` "Bluetooth hatası: <kod>". Android'de yayın yoksa QR'ın üstünde kırmızı uyarı şeridi. Çıkışta `KioskBeacon.stop()`. Her dilim `bleLog("Kiosk yayında: jeton … (dilim …)")`.
- Çıkış: yalnızca panelin sol üstündeki geri düğmesi (uzun basma yoktur).

### 4.3 `BluetoothGate.tsx` (`scan.tsx` ve `kiosk/[id].tsx` ekranlarının altında)
Bluetooth `PoweredOff` veya `Unauthorized` ise ekranın altında (tam ekran pencere değil; geri oku kullanılabilir) uyarı kartı: "Bluetooth kapalı" / "Bluetooth izni gerekli", "Kapının yanında olduğunuzu doğrulamak için Bluetooth'un açık olması gerekiyor." + platforma göre kısa talimat, düğme iOS+kapalı "Ayarları aç" / Android+kapalı "Bluetooth'u aç" / izin "İzin ver". Kapatma düğmesi yoktur; Bluetooth açılınca kendiliğinden kaybolur. Expo Go'da (`unavailable`) görünmez.

## 5. `modules/kiosk-beacon`

Referanstaki 7 dosyanın aynısı. `index.ts`: `requireOptionalNativeModule<KioskBeaconNative>("KioskBeacon")` (Expo Go'da null) ve sarmalayıcı `KioskBeacon`: `isSupported()` (hata yakalanır), `lastError()` (modül yoksa `"NATIVE_MODULE_MISSING"`), `start(serviceUuid, tokenHex)`, `stop()`.
Android (`KioskBeaconModule.kt`): `BluetoothLeAdvertiser`; `ADVERTISE_MODE_LOW_LATENCY`, `ADVERTISE_TX_POWER_MEDIUM`, bağlanılamaz, süresiz; veri: cihaz adı/güç yok, `addServiceData(ParcelUuid(uuid), 8 bayt jeton)`; her başlatmada önce eski yayın durdurulur; hata `ADVERTISE_FAILED_<kod>`; `OnDestroy` → durdur. iOS: `isSupported` false (Apple service data yayınlayamaz).
Yerel modül değişince **yeniden derleme** gerekir (`npx expo run:android --device`); Metro yenilemesi yetmez.

## 6. Cihaz kimliği ve güvenlik

- `deviceId` ve `deviceSecret` bir kez üretilir, SecureStore'da saklanır; değişmez. (iOS'ta Keychain kayıtları uygulama silinse de kalır.)
- `deviceSecret` yalnızca `set-password` isteğinde sunucuya gider; sonra yalnızca imza üretmekte kullanılır.
- Token'lar yalnızca SecureStore'da. AsyncStorage'a gizli bilgi yazılmaz.
- Şifre, token, `deviceSecret`, kiosk `secret` loglanmaz (BLE jetonları kısa ömürlüdür, loglanabilir).
- Galeriden QR seçme yoktur; yalnızca canlı kamera.

## 7. Komutlar ve test

```bash
npm install
cp .env.example .env                         # EXPO_PUBLIC_API_URL=http://<bilgisayarın yerel IP'si>:3000  (ipconfig getifaddr en0)
npm run typecheck && npx eslint src          # HER işten sonra — hatasız/uyarısız olmalı
npx expo start --go                          # Expo Go ile hızlı arayüz denemesi (BLE YOK, yerel modül YOK)
npx expo run:ios --device                    # geliştirme derlemesi (BLE var, loglar terminalde)
npx expo run:android --device                # Android (Java 17 + Android SDK + NDK 27.1 gerekir)
TEAM_ID=XXXXXXXXXX bash tools/ios-kur.sh     # bağımsız Release derlemesi → iPhone
bash tools/android-apk.sh                    # kiosk APK'sı (arm64-v8a) → izmir-fen-kiosk.apk
swift tools/mac-kiosk-beacon.swift --email KIOSK_EPOSTA   # Mac'i test kioskuna çevirir (şifre gizli sorulur)
```
Uygulamanın her ekranı için bilgisayarda **AdminPanel `npm run dev` açık** olmalı; telefon ve bilgisayar aynı Wi-Fi'da olmalı.
Hangi işin hangi derlemede denenebileceği: arayüz + API (öğrenci girişi, ana ekran, kamera ile okutma, kiosk QR) → Expo Go yeter; BLE tarama/yayın → geliştirme derlemesi (`expo run:*`) gerekir.

## 8. Yapılmaması gerekenler

- Sunucu kararlarını istemcide tekrar vermek (yön, cihaz bağlılığı, QR geçerliliği) — istemci yalnızca gösterir ve gönderir.
- `protocol.ts`'i sunucuyu güncellemeden değiştirmek; güvenli depo anahtar adlarını değiştirmek.
- BLE kütüphanesini koşulsuz `import` etmek (Expo Go'da çöker) — `ble.ts`'teki gibi `require` + `try/catch`.
- Kiosk ekranında QR'ı sunucudan çekmek: QR cihazda `secret` ile üretilir.
- `ios/` / `android/` klasörlerini elle düzenlemek veya git'e eklemek; paket eklemek; `app.json` paket kimliklerini değiştirmek.
- Referanstaki `eas.json` `projectId`/`owner` değerlerini veya gerçek IP adreslerini kopyalamak.

## 9. Sık karşılaşılan hatalar

| Belirti | Neden / çözüm |
| --- | --- |
| "Sunucuya ulaşılamadı" | AdminPanel `npm run dev` kapalı, telefon farklı Wi-Fi'da, `.env` adresi yanlış (localhost değil, bilgisayarın IP'si) veya `.env` değişti ama Metro yeniden başlatılmadı (`npx expo start -c`) |
| BLE durumu hep "Bluetooth doğrulaması bu sürümde kapalı" | Expo Go kullanılıyor; `npx expo run:ios --device` / `run:android --device` ile geliştirme derlemesi gerekir |
| Kiosk "BLE yayını yok (iOS kiosk)" | iPhone/iPad kiosk olamaz (service data yok) — Android tablet veya Mac test kioskü kullanın |
| `Cannot find native module 'KioskBeacon'` | Modül değişti ama uygulama yeniden derlenmedi |
| Tüm okutmalar `BAD_SIGNATURE` | `deviceSecret` sunucudakiyle farklı (ör. uygulama silinip yeniden kuruldu, Android) → panelden **Cihazı sıfırla**, yeniden giriş |
| Tüm okutmalar `EXPIRED_QR` | Kiosk saat farkını düzeltmiyor (`offset`) veya QR `slotAt` hesabı yanlış — test vektörleriyle karşılaştırın |
| `Cannot find module '@noble/hashes/hmac'` | Yol `.js` uzantılı olmalı: `@noble/hashes/hmac.js` |
