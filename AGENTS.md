# AGENTS.md — İzmir Fen Okul Giriş-Çıkış Sistemi (monorepo)

Bu dosya, bu depoda çalışan yapay zekâ kodlama ajanları (Claude Code, Codex, Cursor vb.) ve geliştiriciler içindir.
Projenin **ne yaptığını, iki uygulamanın birbiriyle nasıl konuştuğunu ve değişmemesi gereken kuralları** anlatır.
Alt projelerin kendi ayrıntıları: [`AdminPanel/AGENTS.md`](AdminPanel/AGENTS.md), [`QrScannerApp/AGENTS.md`](QrScannerApp/AGENTS.md).

> **Bu dosyadaki protokol, API sözleşmesi, veri modeli ve iş kuralları birebir uygulanmalıdır** —
> mobil uygulama ile sunucu ancak böyle birbirini anlar.

---

## 0. Hedef: referans projeyle birebir aynı yapı

Bu depo, okulun mevcut **referans projesinin** (Fen Bahçeleri giriş-çıkış sistemi) aynısını yeniden geliştirmek içindir.
Geliştirme bittiğinde aşağıdakiler referansla **birebir aynı** olmalıdır:

- **Dosya ve klasör yapısı**: `AdminPanel/AGENTS.md` §2 ve `QrScannerApp/AGENTS.md` §2'deki **hedef ağaç** — aynı yollar, aynı dosya adları.
- **Dışa aktarılan fonksiyon/tip adları** (`repo.ts`, `auth.ts`, `api.ts`, `scan.ts`, `protocol.ts`, mobil `lib/*`), **route yolları**, **ekran yolları** (Expo Router).
- **Protokol, API sözleşmesi, hata kodları, hata mesajları, iş kuralları** (§4–§6), **tablo/kolon adları** (`db/schema.sql`), **ortam değişkeni adları**.
- **Davranış**: aynı istek → aynı yanıt (aynı Türkçe mesaj dahil) ve aynı veritabanı değişikliği.
- **Arayüz**: ekranlar, alanlar, düğme metinleri ve renk paleti referanstaki gibidir. Yalnızca okul adı/logo harfleri değişir (§0.2). Görünümde küçük farklar hata sayılmaz, ama **akış ve metinler** aynı olmalıdır.

Hedef ağaçta olmayan dosya, klasör veya fonksiyon adı **uydurulmaz** — istisna: §0.3'e göre prompt/görev yeni bir özellik istiyorsa, yeni dosyalar mevcut yapıya uygun yere ve aynı adlandırma kurallarıyla eklenir.
Geliştirme sırası ve her adımın ayrıntısı: proje sahibinin Jira görevleri (Web: `WEB-1 … WEB-13`, Uygulama: `APP-1 … APP-7`).

### 0.1 Kurallar bu depodadır

Referans projenin kaynak koduna bu depodan erişilmez. Kurallar için **yalnızca bu depodaki** AGENTS.md / CLAUDE.md dosyaları geçerlidir.
Bu depoda **zaten hazır** olan dosyaları (`src/lib/db/*`, `db/schema.sql`, `scripts/db-*.mjs`, `scripts/create-admin.mjs`, yapılandırma dosyaları, `infra/`) görev açıkça istemedikçe **değiştirmeyin**; bunlar ortak veritabanı için bilerek böyledir.

### 0.3 Referansta olmayan veya referanstan farklı istenen özellikler — prompt esastır

Referans, **varsayılan** davranıştır; bağlayıcı olan, kullanıcının verdiği görev/prompt'tur:
- Prompt/görev **referans depoda olmayan bir özellik** isterse veya **var olan bir özelliğin değiştirilmesini** isterse, **prompt baz alınır**. Referanstaki eski davranış gerekçe gösterilerek prompt'a direnilmez; referans yalnızca kod stili ve yapı için örnek olarak kullanılır.
- Yeni dosya, ekran, sayfa, route, repo fonksiyonu gerekiyorsa mevcut katman yapısına (§3) ve adlandırma kurallarına (§8) uygun eklenir; ilgili AGENTS.md hedef ağacı aynı işte güncellenir.
- Prompt ile referans çelişiyorsa ve prompt açıkça değişiklik istiyorsa prompt uygulanır; belirsizse (referansla aynı mı kalsın, değişsin mi?) kullanıcıya sorulur.

**Prompt istese bile değişmeyen ana yapılar** (bunları etkileyen bir istek gelirse uygulamadan önce durun, etkisini açıklayın ve proje sahibinin onayını isteyin):
1. **Bluetooth yapısı:** BLE jetonu üretimi ve doğrulaması, `BLE_SERVICE_UUID`, service data / `"FB"` yerel ad biçimi, `modules/kiosk-beacon` yayın modeli, `lib/ble.ts` tarama modeli.
2. **Panel ↔ uygulama bağlantısı:** mobil uygulamanın yalnızca `/api/mobile/*` HTTP JSON API'si üzerinden konuşması, mevcut uç noktaların yolları/istek/yanıt biçimleri ve hata biçimi `{ error, code }` (§5), JWT rolleri ve süreleri, mobilin veritabanına asla bağlanmaması.
3. **Protokol:** QR biçimi `FB2`, dilim süresi ve toleranslar, HMAC kuralları, istek imzası (§4) — sunucu ve mobil `protocol.ts` birlikte.
4. **Güvenlik çekirdeği:** cihaz bağlama (bir öğrenci = bir cihaz), okutma doğrulama sırası (§6), kiosk anahtarının cihazda QR üretmesi, gizli bilgilerin saklanma yeri.
5. **Veri erişim katmanı ve şema:** SQL yalnızca `repo.ts`'te; `db/schema.sql` yalnızca proje sahibi tarafından değişir.

Bu yapılara **dokunmadan** eklenen özellikler (yeni panel sayfası, yeni ekran, mevcut uç noktaya geriye uyumlu yeni alan, yeni rapor, metin/tasarım değişikliği vb.) serbesttir ve prompt'a göre yapılır. Yeni bir mobil uç noktası gerekiyorsa `/api/mobile/` altında aynı kalıpla (handler + zod + ApiError) eklenir ve §5 tablosu güncellenir.

### 0.4 Geçici (dummy) veri — FE/APP backend'i beklemez

Panel (FE) ve uygulama (APP) görevleri, henüz yazılmamış backend parçasını beklemeden **geçici veriyle** yapılır; ilgili BE görevi bitince geçici kısım silinir.
- İşaret: her geçici satır `// GEÇİCİ (BE-n): <ne kalkacak>` yorumunu taşır (`BE-n` = gerçeğe çevirecek görev). Kontrol: `grep -rn "GEÇİCİ (BE-" AdminPanel/src QrScannerApp/src`.
- Biçim gerçeğin aynısıdır: aynı uç nokta yolu, alan adları, fonksiyon adı ve imzası (§5, `AdminPanel/AGENTS.md` §4.2). Gerçek gelince ekran değişmez, yalnızca geçici kısım silinir.
- **Panel:** listeler hazır repo okuma fonksiyonlarından **gerçek** okunur. Henüz olmayan Server Action'lar son adlarıyla boş yazılır ve "Bu işlem henüz hazır değil (BE-n)." döner. Henüz verisi olmayan listelerin örnek satırları yalnızca `AdminPanel/src/lib/dummy.ts`'te durur; yalnızca geliştirmede (`NODE_ENV !== "production"`) ve gerçek liste boşken gösterilir.
- **Uygulama:** ekranlar her zaman gerçek `api()` çağrısı yapar. Sunucuda henüz olmayan uç noktaların örnek yanıtları `QrScannerApp/src/lib/dummy.ts`'te durur ve yalnızca `lib/api.ts`'teki tek bir `__DEV__` kontrollü satırla döndürülür. Örnek kiosk §4 test anahtarını (`kiosk_ana`) kullanır.
- Geçici veri veritabanına hiçbir şey yazmaz; adlar açıkça sahtedir ("Örnek Öğrenci"). Gizli bilgi içermez.
- Canlıya çıkmadan önce arama boş dönmeli ve iki `dummy.ts` silinmiş olmalıdır.

### 0.2 Referanstan bilinçli farklar (uyarlama tablosu)

| Konu | Referans (Fen Bahçeleri) | Bu proje (İzmir Fen) |
| --- | --- | --- |
| Görünen okul adı (panel girişi, panel menüsü, uygulama açılışı, kiosk ekranı, sayfa başlığı) | `Fen Bahçeleri` | `İzmir Fen` |
| Logo kutusundaki harfler (panel + uygulama) | `FB` | panel: menüde uygulama logosu (`AdminPanel/public/logo.png` = `QrScannerApp/assets/logo.png`; giriş sayfasında henüz `İF`) · uygulama: okul logosu (`QrScannerApp/assets/logo.png`, `Logo` bileşeni; uygulama ikonu da bu logo; telefondaki uygulama adı `DOQR`) |
| Panel `<title>` | `Fen Bahçeleri · Giriş-Çıkış Paneli` | `İzmir Fen · Giriş-Çıkış Paneli` |
| Veritabanı | Yerel Docker (`AdminPanel/docker-compose.yml`, `db:up`, `db:reset`, `db:psql`, `db:import`, `POSTGRESQL_KURULUM.md`, `json-to-postgres.mjs`) | **AWS ortak geliştirme veritabanı** (`infra/dev-db/`, `npm run db:check`). Referanstaki bu yerel veritabanı dosyaları/komutları **bu projeye eklenmez** |
| Bağlantı havuzu | 10 | 3 (`pg.ts`'te hazır) |
| Şemayı uygulayan | `DATABASE_URL` | yalnızca proje sahibi, `DATABASE_OWNER_URL` (hazır) |
| Örnek uç nokta | yok | `/api/health` + `dbHealth()` (hazır, kalır) |
| `AUTH_SECRET` üretimde yoksa | `config.ts` uyarı yazar, devam eder | `auth.ts` **hata fırlatır** (§5.1) |
| Panel sayfalarında oturum kontrolü | yalnızca `(panel)/layout.tsx` | layout **ve** her `(panel)/**/page.tsx` `requireWebAdmin()` çağırır (layout istemci içi gezinmede çalışmaz); `getWebAdmin` React `cache()` ile |
| `handler()` beklenmeyen hata (500) | hatanın mesajını döner | her zaman "Beklenmeyen hata."; ayrıntı yalnızca sunucu logunda (`[api]`) |
| `RejectReason` tipi | `WRONG_STATE` içerir | `WRONG_STATE` **yok** → `REJECT_MESSAGES`'a da eklenmez (yoksa tip hatası) |
| Uçtan uca test (`scripts/e2e-test.mjs`) | Örnek öğrenci `ali.yilmaz@…` ve kiosk şifresini kullanır | **Kendi verisini oluşturur ve siler** (ortak veritabanı, §7.1) |
| `Dockerfile` | `pnpm` + `pnpm-lock.yaml` | `npm ci` + `package-lock.json` |
| Android APK adı (`tools/android-apk.sh`) | `fen-bahceleri-kiosk.apk` | `izmir-fen-kiosk.apk` |
| `app.json` adları / paket kimliği | `com.fenbahceleri.giris` | `com.izmirfen.giris` (hazır) |
| `eas.json` / EAS `projectId`, `owner` | referansın hesabı | proje sahibinin kendi EAS hesabı (referanstaki kimlikler **kopyalanmaz**) |
| Canlı ortam adları (`deploy/`) | `fenbahceleri`, `fb_app` | `izmirfen`, `izmirfen_app` |
| Canlı veritabanı | `deploy/docker-compose.yml` içinde ayrı `db` konteyneri | **Aynı Lightsail sunucusundaki `infra/dev-db` veritabanı** (proje sahibinin kararı): `deploy/` yalnızca `app` + `caddy` çalıştırır, `izmirfen-devdb_default` ağına katılır; şemayı `kur.sh` uygular. Rehber: `CANLIYA_ALMA.md` |
| Veli bildirimi | SMS (`sms_messages`, yalnızca `mock`) | **WhatsApp** (`wp_messages`); geliştirmede `WHATSAPP_PROVIDER=mock`, gerçek gönderim `insider` (Insider şablon API'si, `INSIDER_API_KEY` + `WHATSAPP_TEMPLATE`). Öğrenci başına en fazla 2 veli; her birine mesaj gider |
| Öğrenci giriş kimliği | e-posta (`students.email`, `POST /student/check-email`, gövdelerde `email`) | **cep telefonu** (`students.phone`, tekil, `+905XXXXXXXXX`; `POST /student/check-phone`, gövdelerde `phone`; `src/lib/phone.ts` yazım biçimlerini çevirir). `students.email` isteğe bağlı bilgi |
| Hareket inceleme | `/hareketler`'de "Uygun"/"Şüpheli" düğmeleri ve filtreleri, `reviewEventAction`, `EVENT_REVIEWED` denetim kaydı, `ReviewBadge` | **Yok** (karar: Alperen Eser). Kabul edilmiş kayıt sonradan incelenmez; sahte okutma baştan reddedilir: kayıtlı cihaz (`DEVICE_NOT_BOUND`, `BAD_SIGNATURE`), QR süresi (`EXPIRED_QR`), BLE (`BLE_MISSING`, `BLE_MISMATCH` — geliştirmede `BLE_REQUIRED=false`, canlıda `true`). `review_status`/`reviewed_by` şemada kalır, her kayıt `UNREVIEWED` |
| Örnek veri (`seed.ts`, `@fenbahceleri.test` hesapları) | Boş veritabanına otomatik eklenir | **Yok** — yöneticiler `npm run admin:create`, öğrenciler panelden/veritabanından eklenir |

**Değişmeyenler (protokol — dokunmayın):** `FB2`, `BLE_SERVICE_UUID`, BLE yerel ad öneki `"FB"`, panel çerezi `fb_admin`, mobil güvenli depo anahtarları `fb.deviceId / fb.deviceSecret / fb.studentToken / fb.adminToken / fb.boundEmail`, kimlik önekleri (`stu_`, `kiosk_` …).

## 1. Ürün

Öğrencinin okula giriş ve çıkışını kayıt altına alan sistem:

1. Okul kapısında bir **kiosk** (Android tablet) durur; ekranında **5 saniyede bir değişen imzalı bir QR kod** gösterir ve aynı anda **Bluetooth (BLE) ile kısa ömürlü bir jeton** yayınlar.
2. Öğrenci kendi telefonundaki uygulamayla QR'ı okutur. Telefon, kioskun BLE jetonunu da duyduğunu kanıtlar (QR'ın fotoğrafını eve götürüp okutmayı engeller).
3. Sunucu okutmayı doğrular, **giriş veya çıkış** kaydı açar ve veliye WhatsApp mesajı kaydı oluşturur.
4. Okul yönetimi **web panelinden** öğrencileri, cihazları, kayıtları ve reddedilen okutmaları izler.

**Yönsüz QR, kapı bazında durum:** Kioskun giriş/çıkış ayrımı yoktur. Birden fazla kapıya (ana kapı, yurt kapısı …) ayrı kiosk konabilir ve **her kioskun giriş-çıkış durumu birbirinden bağımsızdır**. Öğrencinin bir kiosktaki **ilk** okutmasında uygulama "Giriş mi, çıkış mı?" diye sorar; o kiosktaki sonraki her okutmada yönü **sunucu** belirler (o kapıda içerideyse çıkış, dışarıdaysa giriş).

## 2. Mimari

```
 Android tablet (KIOSK)            Öğrenci telefonu (iOS/Android)          Tarayıcı (yönetici)
 QrScannerApp — kiosk modu         QrScannerApp — öğrenci modu             AdminPanel — web paneli
  • dönen QR (FB2)                  • QR okut + BLE jetonunu dinle
  • BLE yayını (service data)       • isteği cihaz anahtarıyla imzala
          │                                   │                                     │
          └──────────── HTTPS, JSON ──────────┴──────────── HTTPS (çerez oturumu) ──┘
                                              ▼
                          AdminPanel (Next.js) — web paneli + /api/mobile/*
                                              ▼
                                   PostgreSQL (yalnızca sunucu erişir)
```

- **Mobil uygulama veritabanına asla doğrudan bağlanmaz.** Tek bildiği şey sunucu adresidir (`EXPO_PUBLIC_API_URL`).
- Panel ve mobil API **aynı Next.js projesindedir** (`AdminPanel`).
- Tek mobil uygulama iki modda çalışır: **Öğrenci girişi** ve **Yönetici girişi (kiosk)**.

**Geliştirme ortamı (bu depo):** Veritabanı kimsenin bilgisayarında değildir. Ekipteki herkesin yerelde çalışan AdminPanel'i,
**AWS'deki ortak geliştirme veritabanına** bağlanır (SSL zorunlu, ekip kullanıcısı `izmirfen_app`). Telefonlar ise yerel AdminPanel'e istek atar:

```
 Telefon ──HTTP──▶ Geliştiricinin AdminPanel'i (npm run dev, laptop) ──SSL──▶ AWS: ortak geliştirme PostgreSQL'i (5433)
```
Bu veritabanı **canlı (production) veritabanı değildir**; gerçek öğrenci verisi içermez. Ayrıntı: §7.1 ve `infra/dev-db/README.md`.

## 3. Depo yapısı (hedef — referans projeyle birebir aynı)

Geliştirme bittiğinde depo **tam olarak** aşağıdaki gibidir (§0). ✅ = depoda hazır · ⏳ = ekip geliştirecek.
Yeni dosya/klasör açmadan önce burada yerinin olduğundan emin olun; yoksa önce proje sahibine sorun. Ayrıntılı ağaçlar: `AdminPanel/AGENTS.md` §2, `QrScannerApp/AGENTS.md` §2.

```
FenBahceleri_IzmırFen/
├── AGENTS.md ✅, CLAUDE.md ✅, README.md ✅, .gitignore ✅
├── CANLIYA_ALMA.md ✅                     canlı ortam rehberi (maliyet dahil)
├── infra/dev-db/ ✅                        ortak geliştirme veritabanı (yalnızca proje sahibi çalıştırır)
│
├── AdminPanel/                            Next.js 16 — web paneli + mobil API
│   ├── AGENTS.md, CLAUDE.md ✅ · README.md ⏳ · .env.example ✅ · package.json ✅ · next.config.ts ✅ · tsconfig.json ✅
│   ├── Dockerfile ✅, .dockerignore ✅, deploy/ ✅     canlı imaj ve sunucu dosyaları
│   ├── db/schema.sql ✅                    tek şema dosyası
│   ├── scripts/                           db-check ✅ · db-setup ✅ · create-admin ✅ · e2e-test ⏳
│   └── src/
│       ├── lib/
│       │   ├── db/  pg.ts ✅ · repo.ts ✅ · types.ts ✅              TÜM veritabanı erişimi burada
│       │   ├── config.ts ⏳ · auth.ts ⏳ · session.ts ⏳ · api.ts ⏳         ayarlar, JWT, panel oturumu, route yardımcıları
│       │   └── protocol.ts ⏳ · scan.ts ✅ · whatsapp.ts ⏳ · phone.ts ✅       protokol, okutma kuralları, WhatsApp, telefon biçimi
│       ├── components/  AutoRefresh.tsx ⏳ · Badges.tsx ⏳
│       └── app/
│           ├── layout.tsx ✅ · globals.css ✅ · page.tsx ✅ (GEÇİCİ) · api/health/ ✅
│           ├── login/ ⏳                   page · LoginForm · actions
│           ├── (panel)/ ⏳                 layout · NavLinks · actions · page (Canlı durum) · hareketler · ogrenciler (+StudentForms)
│           │                              kiosklar (+AccountForms, DeleteKioskButton) · denemeler · whatsapp · geri-bildirimler · denetim
│           └── api/mobile/ ⏳              config · student/{check-phone,set-password,login,me} · scan · feedback · admin/login · kiosks[/:id/{start,feed}]
│
└── QrScannerApp/                          Expo SDK 57 — öğrenci + kiosk uygulaması
    ├── AGENTS.md, CLAUDE.md ✅ · README.md ⏳ · CIHAZ_TESTI.md ⏳ · TESTFLIGHT.md ⏳
    ├── package.json ✅ · app.json ✅ · app.config.ts ✅ · eas.json ⏳ · tsconfig.json ✅ · assets/ ✅
    ├── modules/kiosk-beacon/ ⏳            Android BLE yayın modülü (Kotlin) + iOS taslağı
    ├── tools/ ⏳                           mac-kiosk-beacon.swift · ios-kur.sh · android-apk.sh
    └── src/
        ├── app/                           _layout ✅ · index ✅ (açılış) · welcome ✅ · student-login ✅ (telefon) · student-home ✅ · scan ⏳
        │                                  admin-login ⏳ · kiosk-select ⏳ · kiosk/[id] ⏳
        ├── components/ ⏳                  ui.tsx · icons.tsx · BluetoothGate.tsx
        └── lib/                           config ✅ · api ⏳ · session ⏳ · device ⏳ · protocol ⏳ · ble ⏳
```

**Katmanlar (her iki proje):** ekran/route **ince** kalır → iş kuralı `lib/` içinde → veritabanı yalnızca `lib/db/repo.ts`, sunucuyla konuşma yalnızca mobil `lib/api.ts`.

## 4. Protokol (değiştirmeyin; değiştirirseniz iki tarafı aynı commit'te güncelleyin)

| Sabit | Değer |
| --- | --- |
| `PROTOCOL_VERSION` | `"FB2"` |
| `SLOT_SECONDS` | `5` — QR ve BLE jetonu her 5 sn'de değişir |
| Dilim | `slot = floor(unixMilisaniye / 1000 / 5)` (sunucu saatine göre; kiosk sunucu saatiyle farkını düzeltir) |
| Sunucu toleransı | geçmiş **2** dilim, gelecek **1** dilim |
| `BLE_SERVICE_UUID` | `6f1b0000-5a1e-4c1a-9b9e-fb0000000001` |
| HMAC | HMAC-SHA256; anahtarlar **hex** metin olarak saklanır, kullanılırken bayta çevrilir |

**QR içeriği**
```
FB2.<kioskId>.<slot>.<imza>
imza = base64url( HMAC(kioskSecret, "FB2|<kioskId>|<slot>") ilk 16 bayt )   // dolgu (=) yok
```
**BLE jetonu** (8 bayt → 16 hex karakter)
```
bleToken = hex( HMAC(kioskSecret, "BLE|<kioskId>|<slot>") ilk 8 bayt )
```
- Android kiosk jetonu `BLE_SERVICE_UUID` altında **service data** olarak yayınlar.
- Apple cihazlar service data yayınlayamaz; test amaçlı iOS/macOS yayıncılar jetonu **yerel ad** olarak yayınlar: `"FB" + 16 hex`. Tarayıcı ikisini de kabul eder.
- Sunucu jetonu QR dilimi ve **±1 komşu dilim** için kabul eder.

**İstek imzası** (öğrencinin okutma isteği)
```
signature = hex( HMAC(deviceSecret, "<qr>|<bleToken veya boş>|<timestamp>") )   // 64 hex
```
`deviceSecret`: telefonun ilk girişte ürettiği 32 baytlık rastgele anahtar (64 hex), güvenli depoda (Keychain/Keystore) saklanır ve şifre oluşturulurken sunucuya bir kez gönderilir.

**Test vektörleri** — sunucu (`AdminPanel/src/lib/protocol.ts`) ve mobil (`QrScannerApp/src/lib/protocol.ts`) **aynı** sonucu vermelidir:
```
kioskSecret  = 00112233445566778899aabbccddeeff00112233445566778899aabbccddeeff
deviceSecret = a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90
kioskId      = kiosk_ana

buildQrPayload(kioskSecret, kioskId, 358175649) = FB2.kiosk_ana.358175649.ivyQsF9B06xMvaPuD6iMnw
buildQrPayload(kioskSecret, kioskId, 358175650) = FB2.kiosk_ana.358175650.ICWLy0KU5DhU-qSqdH3iWA
bleToken(kioskSecret, kioskId, 358175649)       = 548e111d76ce75ae
bleToken(kioskSecret, kioskId, 358175650)       = 9fae1e1ae653d81d
requestSignature(deviceSecret, <ilk QR>, "548e111d76ce75ae", 1791125763537) = b1d5809f5f7aa836a6d9eaa99ea69471dde822176d6539d82f1b286b7484c53b
requestSignature(deviceSecret, <ilk QR>, null, 1791125763537)               = a05cdbeae0232aa7b98e6f0b3c105c1436077b56f903dcf2d5718f12000b3d63
currentSlot(1791125763537) = slotAt(1791125763537, 5) = 358225152
```

## 5. Mobil API sözleşmesi

Taban: `${EXPO_PUBLIC_API_URL}/api/mobile`. Gövdeler JSON. `phone`: Türkiye cep numarası; `05XX XXX XX XX`, `5XX…`, `+90 5XX…` yazımları kabul edilir (geçersizse `400 VALIDATION`). Kimlik: `Authorization: Bearer <JWT>`.
**Hata biçimi her yerde aynıdır:** HTTP durum kodu + `{ "error": "Türkçe, kullanıcıya gösterilebilir mesaj", "code": "MAKINE_KODU" }`.
Doğrulama hatası: `400 { error, code: "VALIDATION", issues }`. Sık deneme: `429 RATE_LIMIT`.

| Uç nokta | Kimlik | İstek | Başarılı yanıt |
| --- | --- | --- | --- |
| `GET /config` | — | — | `{ bleRequired, bleServiceUuid, slotSeconds, duplicateWindowSeconds, serverTime }` |
| `POST /student/check-phone` | — | `{ phone, deviceId }` | `{ next: "create-password" \| "password", firstName }` · Hatalar: `404 NOT_FOUND`, `409 DEVICE_TAKEN` (bu telefon başka öğrenciye bağlı), `403 WRONG_DEVICE` (hesap başka telefona bağlı) |
| `POST /student/set-password` | — | `{ phone, password (≥8), deviceId, deviceSecret (64 hex), platform: "ios"\|"android"\|"web"\|"unknown" }` | `{ token, student: { id, firstName, lastName, className } }` · Hatalar: `409 ALREADY_HAS_PASSWORD`, `409 DEVICE_TAKEN`, `403 WRONG_DEVICE`, `404 NOT_FOUND` |
| `POST /student/login` | — | `{ phone, password, deviceId }` | `{ token, student }` · `401 BAD_CREDENTIALS`, `403 WRONG_DEVICE` |
| `GET /student/me` | öğrenci | — | `{ student: {…, presenceStatus: "IN"\|"OUT"}, needsDirection: boolean, nextDirection: "IN"\|"OUT"\|null, events: [{ id, direction, occurredAt, kioskName }] }` (son 10) · `presenceStatus` öğrencinin herhangi bir kapıdaki son hareketidir; `needsDirection` yalnızca hiç kaydı yokken `true` (başka bir kapıdaki ilk okutmada sunucu `DIRECTION_REQUIRED` döner) · `401 DEVICE_REVOKED` |
| `POST /scan` | öğrenci | `{ qr, ble: { token, rssi } \| null, timestamp, signature, direction?: "IN"\|"OUT"\|null }` | `{ eventId, direction, occurredAt, time: "HH:mm", duplicate, studentName, bleVerified: boolean\|null }` · Red: `422 { code: <RejectReason> }` |
| `POST /admin/login` | — | `{ email, password }` | `{ token, admin: { id, fullName } }` — ADMIN veya KIOSK rolü |
| `POST /feedback` | öğrenci | `{ message (1–1000 karakter) }` | `201 { id }` · panelde "Geri bildirimler" sayfasında görünür · `401 DEVICE_REVOKED`, 5 istek / 10 dk |
| `GET /kiosks` | kiosk-admin | — | `{ kiosks: [{ id, name }] }` (yalnızca aktifler) |
| `POST /kiosks/:id/start` | kiosk-admin | — | `{ kiosk: { id, name }, secret, slotSeconds, bleServiceUuid, bleRequired, serverTime }` |
| `GET /kiosks/:id/feed?since=ISO` | kiosk-admin | — | `{ serverTime, events: [{ id, name, className, direction, occurredAt, time }], attempts: [{ id, time, studentName, result, rejectReason, bleToken, bleRssi, bleOk }] }` |

**JWT** (HS256, `AUTH_SECRET`): öğrenci `{ sub: studentId, role: "student", deviceId }` 30 gün · kiosk `{ sub: adminId, role: "kiosk-admin" }` 180 gün · web paneli `{ sub, role: "web-admin" }` 12 saat, `fb_admin` httpOnly çerezinde.

### 5.1 Kimlik doğrulama ve `AUTH_SECRET`

**Durum:** BE-1 ile geliştirildi (`config.ts`, `auth.ts`, `session.ts`). Geliştirmede `AUTH_SECRET` boşsa sabit geliştirme anahtarı kullanılır; yine de herkes kendi anahtarını `.env.local`'e yazar.

**Ne işe yarar:** Sunucunun verdiği tüm "giriş kartlarını" (JWT, HS256) imzalar ve doğrular — öğrenci token'ı (30 gün), kiosk token'ı (180 gün), panel çerezi `fb_admin` (12 saat). Bu anahtarı bilen herkes, şifre bilmeden herhangi bir öğrenci/yönetici adına geçerli token üretebilir.

**Kurallar:**
- **Her geliştirici kendi anahtarını kendisi üretir** (`openssl rand -hex 32`) ve yalnızca kendi `.env.local`'ine yazar. Proje sahibi anahtar dağıtmaz; anahtar kimseyle paylaşılmaz, commit'lenmez, loglanmaz.
- Token'lar veritabanında saklanmaz; farklı geliştiricilerin farklı anahtar kullanması ortak veritabanında sorun çıkarmaz (telefon başka geliştiricinin sunucusuna bağlanırsa yeniden giriş ister — beklenen).
- **Canlı sunucunun anahtarı ayrıdır**, yalnızca sunucudaki `deploy/.env`'de durur. Değiştirilirse tüm oturumlar düşer (veri kaybı yok).

**Uygulama (geliştirilecek, referansla aynı):**
- `src/lib/config.ts` → `authSecret: process.env.AUTH_SECRET ?? "<yalnızca geliştirme için sabit anahtar>"`.
- `src/lib/auth.ts` → `signToken(claims, expiresIn)`, `verifyToken(token)`, `bearerClaims(req)` (`jose`, HS256). Anahtar **ilk kullanımda** okunur (derleme sırasında gerekmez).
- **Üretimde zorunlu (bu projede referanstan farklı, daha sıkı):** `NODE_ENV=production` iken `AUTH_SECRET` tanımlı değilse `auth.ts` token imzalamayı ve doğrulamayı **reddeder** (hata fırlatır). Bilinen geliştirme anahtarıyla canlıda asla çalışılmaz; yalnızca uyarı yazıp devam etmek yeterli değildir.

**RejectReason** (`scan_attempts.reject_reason`): `INVALID_QR, EXPIRED_QR, UNKNOWN_KIOSK, BAD_SIGNATURE, DEVICE_NOT_BOUND, BLE_MISSING, BLE_MISMATCH, REPLAY` (+ yalnızca yanıt kodu olarak `DIRECTION_REQUIRED`; eski kayıtlarda `WRONG_STATE` bulunabilir).

## 6. İş kuralları (sunucu uygular; istemciye güvenilmez)

**Hesap ve cihaz**
- Kayıt ol yoktur. Yalnızca panelde tanımlı, aktif öğrencilerin telefon numaraları giriş yapabilir.
- İlk giriş: telefon numarası → şifre oluştur (en az 8 karakter). Bu anda telefon (`deviceId` + `deviceSecret`) hesaba **bağlanır**.
- **Bir öğrenci = bir aktif cihaz; bir cihaz = bir öğrenci.** Veritabanında kısmi tekil indeksle de korunur.
- Telefon-hesap bağlantısının **tek kaynağı sunucudur**; istemci yerel olarak "bu telefon başka hesaba bağlı" kararı vermez.
- Şifre sıfırlama yalnızca panelden: **Şifreyi sıfırla** (cihaz bağlı kalır) / **Cihazı sıfırla** (cihaz iptal + şifre silinir; yeni telefonda baştan).
- Şifreler bcrypt (maliyet 10) ile saklanır; düz metin şifre hiçbir yerde tutulmaz/loglanmaz.

**Okutma doğrulama sırası** (`AdminPanel/src/lib/scan.ts`) — her adım başarısızsa `scan_attempts`'a `REJECTED` yazılır:
1. Cihaz bu öğrenciye aktif bağlı mı? → `DEVICE_NOT_BOUND`. İstek imzası doğru mu? → `BAD_SIGNATURE`
2. QR biçimi + kiosk var ve aktif mi + kiosk imzası doğru mu? → `INVALID_QR` / `UNKNOWN_KIOSK`
3. Dilim tazeliği (−2…+1) → `EXPIRED_QR`
4. BLE: jeton geldiyse eşleşme kontrolü (`ble_ok`); `BLE_REQUIRED=true` ise jeton yok → `BLE_MISSING`, eşleşmiyor → `BLE_MISMATCH`
5. Aynı cihaz + kiosk + dilim daha önce **kabul** edildiyse → `REPLAY` (veritabanında kısmi tekil indeksle de korunur)
6. **Yön** (her kiosk/kapı ayrı; yalnızca öğrencinin **bu kiosktaki** kayıtlarına bakılır):
   - Bu kiosktaki son kayıttan bu yana `DUPLICATE_WINDOW_SECONDS` (varsayılan 120) geçmediyse → **çift okutma**: yeni kayıt/WhatsApp mesajı yok, `duplicate: true` ile son kayıt döner.
   - Öğrencinin bu kioskta hiç kaydı yoksa yön istekteki `direction`'dır; yoksa `422 DIRECTION_REQUIRED` (bu durum **kaydedilmez**).
   - Bu kioskta kaydı varsa yön = o kiosktaki son hareketin tersi (IN → OUT, OUT → IN). İstekteki `direction` **yok sayılır**.
   - `students.presence_status` öğrencinin herhangi bir kapıdaki son hareketini tutar (uygulamadaki durum kartı); yön kararında kullanılmaz.
7. Kabul: tek transaction'da `attendance_events` + `students.presence_status` + her veliye `wp_messages` (bir olay için bir veliye tek WhatsApp mesajı).

**WhatsApp**: veli bildirimleri SMS değil WhatsApp mesajıdır. Geliştirmede gerçek gönderim yok (`WHATSAPP_PROVIDER=mock`, varsayılan; veritabanı ortak olduğu için böyle kalmalı): mesaj kaydedilir, durumu `MOCK_SENT` olur. `WHATSAPP_PROVIDER=insider` iken Insider WhatsApp API'sine (`https://whatsapp.useinsider.com/v1/send`) onaylı şablonla (`WHATSAPP_TEMPLATE`) gönderilir: başarılı `SENT`, HTTP hatası/zaman aşımı/anahtar yoksa `FAILED`. `INSIDER_API_KEY` gizlidir: yalnızca sunucudaki `.env.local` / `deploy/.env`'de durur, `NEXT_PUBLIC_` öneki almaz, loglanmaz. Şablon parametreleri sırasıyla: veli adı, öğrenci ad soyad, tarih (`27 Ekim 09:59`), konum (kiosk adı), işlem (`giriş`/`çıkış`). Panelde görünen metin: `Merhabalar <Veli>, <Ad Soyad> öğrencimiz <tarih> tarihinde <Kiosk adı> konumundan <giriş|çıkış> yapmıştır. Bilgilerinize sunarız.` Öğrencinin telefonu girilmiş her velisine (en fazla 2) ayrı mesaj gider. Her kioskun **WhatsApp bildirimi** ayrı açılıp kapatılır (`kiosks.whatsapp_enabled`, varsayılan açık); kapalı kioskta giriş-çıkış yine kaydedilir, yalnızca veliye mesaj yazılmaz (manuel kayıtlar da seçilen kioskun ayarına uyar).

**Panelden manuel giriş/çıkış**: telefonu olmayan öğrenci için; **kapı (kiosk) seçilerek** yapılır, `source = MANUAL`, `kiosk_id` = seçilen kapı, aynı WhatsApp kuralları. Yön öğrencinin o kapıdaki durumuyla aynıysa (içerideyken "giriş") hiçbir şey yapılmaz.

**Kiosklar ve kiosk tablet hesapları** (panel → Kiosklar)
- Kiosk eklenir (yalnızca ad; gizli anahtarı sunucu üretir), WhatsApp bildirimi açılır/kapatılır, devre dışı bırakılır/etkinleştirilir veya **kalıcı silinir**. Silinen kioskun geçmiş hareket ve okutma kayıtları korunur (`kiosk_id` NULL olur, FK `ON DELETE SET NULL`).
- **Kiosk tablet hesabı** = `admin_users` tablosunda `role = 'KIOSK'` olan kullanıcı. Tablette "Yönetici girişi" ile kullanılır, **web paneline giremez**. Panelden yeni hesap açılır (ad, e-posta, şifre **≥ 10** karakter) ve şifresi sıfırlanır (eski şifre gösterilmez; yalnızca yenisi verilir). Tablette açık oturum (180 günlük token) şifre değişince düşmez; bir sonraki girişte yeni şifre gerekir.
- `ADMIN` rolündeki hesaplar panelden oluşturulmaz; yalnızca `npm run admin:create` ile.

**Saat dilimi**: gösterimler ve "bugün" hesabı `Europe/Istanbul`.

## 7. Güvenlik kuralları (açık kaynak proje)

- **Hiçbir gizli bilgi depoya girmez**: `.env`, `.env.local`, `infra/dev-db/.env`, veritabanı adresi+şifresi, anahtarlar, gerçek şifreler. Şablonlar `*.env.example` olarak, boş değerlerle tutulur. Veritabanı şifresi sohbete/issue'ya/ekran görüntüsüne de yazılmaz.
- Kod, betik veya belgelere gerçek/örnek şifre yazmayın; komut satırı argümanı olarak şifre vermeyin (kabuk geçmişine girer). Gizli değerler ortam değişkeninden veya gizli girişle alınır.
- Kiosk gizli anahtarı (`kiosks.secret`) ve cihaz anahtarı (`devices.device_secret`) API yanıtlarında yalnızca belgelenen yerlerde döner; loglara yazılmaz.
- Veritabanı portu internete açılmaz. Canlıda yalnızca HTTPS.
- Kullanıcıya dönen hata mesajları Türkçe ve kısa; iç hata ayrıntısı (stack, SQL) istemciye dönmez.

### 7.1 Ortak geliştirme veritabanı kuralları

- Herkes **aynı** veritabanına bağlıdır: yaptığınız her yazma işlemini ekibin geri kalanı da görür.
- Ekip kullanıcısı (`izmirfen_app`) yalnızca `SELECT/INSERT/UPDATE/DELETE` yapabilir; tablo oluşturamaz, silemez, `TRUNCATE` yapamaz. Bu bilinçli bir kısıttır.
- **Şema değişikliği**: `AdminPanel/db/schema.sql`'e (ve `types.ts`, `repo.ts`'e) PR olarak eklenir; birleştirildikten sonra **proje sahibi** uygular (`infra/dev-db/kur.sh` veya `npm run db:setup` + `DATABASE_OWNER_URL`). Uygulanana kadar yeni kolonu kullanan kod `main`'e girmez.
- Başkasının kayıtlarını silmeyin/değiştirmeyin. Test verisi oluştururken kendinize ait, tanınır kayıtlar kullanın (ör. öğrenci adı `TEST <adınız>`, telefon `0500 000 XX XX` aralığı, e-posta `test+<adınız>-<rastgele>@izmirfen.test`, kiosk adı `TEST <adınız>`) ve testin sonunda temizleyin.
- `DELETE`/`UPDATE` sorgularını **her zaman `WHERE`** ile yazın; elle SQL çalıştırmadan önce `SELECT` ile etkilenecek satırları görün.
- Geliştirici başına bağlantı havuzu küçük tutulur (`DATABASE_POOL_MAX=3`); kullanmadığınız `npm run dev` süreçlerini kapatın.
- Geliştirme veritabanına gerçek kişi verisi eklenmez.

## 8. Ortak çalışma kuralları

- **Dil**: arayüz metinleri, hata mesajları, kod yorumları ve belgeler **Türkçe**; tanımlayıcılar (değişken, fonksiyon, tablo, kolon) **İngilizce**.
- Veritabanında `snake_case`, kodda `camelCase`. Kimlikler metin ve önekli: `stu_…`, `gua_…`, `adm_…`, `kiosk_…`, `evt_…`, `att_…`, `wpm_…`, `aud_…` (önek + 16 hex).
- Zamanlar veritabanında `timestamptz`, API'de ISO 8601 metin.
- Protokol veya API sözleşmesini değiştiren her iş, **sunucu + mobil + bu dosya + uçtan uca test** birlikte güncellenmeden bitmiş sayılmaz.
- Her yönetici işlemi `audit_logs`'a yazılır.
- Commit mesajları Türkçe, ilk satır ≤ 72 karakter, "ne ve neden".

### 8.1 Kod yazım kuralları — gereksiz kod yok, okunabilir kod

**Gereksiz kod yazmayın**
- Yalnızca görevin istediğini yazın. "İleride lazım olur" diye fonksiyon, parametre, ayar, soyutlama veya dosya eklemeyin.
- Var olanı kullanın: veritabanı → `repo.ts`; route yardımcıları → `api.ts`; mobil istek → `lib/api.ts`; arayüz parçaları → `components/ui.tsx`. Aynı işi yapan ikinci bir yardımcı yazmayın.
- Yorum satırına alınmış kod, kullanılmayan import/değişken/fonksiyon/bağımlılık, `console.log` artıkları, boş `try/catch` bırakmayın.
- Tek satırı saran (yalnızca başka bir fonksiyonu çağıran) sarmalayıcı fonksiyon yazmayın.
- Yeni paket eklemeden önce mevcut paketlerle yapılabildiğini kontrol edin; gerekiyorsa PR'da nedenini yazın (mobilde `npx expo install`).

**Okunabilir yazın**
- Bir dosya = bir sorumluluk. Route, Server Action ve ekranlar ince; iş kuralı `lib/` içinde.
- Açıklayıcı İngilizce adlar (`findStudentByPhone`, `createEventWithWhatsApp`); kısaltma yok (`stu`, `tmp`, `x` yalnızca çok kısa kapsamlarda).
- Kısa fonksiyonlar, erken dönüş (`if (!x) return …`), en fazla 2–3 iç içe blok. Uzun koşulları adlandırılmış değişkenlere bölün.
- Her dosyanın başında amacını anlatan 1–2 satırlık Türkçe yorum. Yorumlar **neden**'i anlatır, kodun ne yaptığını tekrar etmez.
- TypeScript strict: `any` yok (zorunluysa nedenini yorumla), `!` (non-null) yalnızca garanti varsa, API ve veritabanı tipleri `types.ts` / `api.ts`'ten.
- Sihirli sayı yok: süreler, sınırlar `config.ts` veya `protocol.ts` sabitlerinde (ör. `SLOT_SECONDS`, `duplicateWindowSeconds`).
- Hatalar: sunucuda `throw new ApiError(durum, "Türkçe mesaj", "KOD")`; sessizce yutulmaz. Loglar önekli ve amaçlı: `[scan]`, `[api]`, `[db]`, `[whatsapp]`, `[BLE]`.
- Biçim mevcut kodla aynı: 2 boşluk girinti, çift tırnak, noktalı virgül, çok satırlı listelerde sondaki virgül; ESLint uyarısız.
- Next.js: bileşenler varsayılan olarak **sunucu bileşeni**; `"use client"` yalnızca form durumu/etkileşim için. Expo: ekranlar `src/app/`, ekran olmayan her şey `src/components/` veya `src/lib/`.

## 9. Bitti tanımı (her iş için)

1. `AdminPanel`: `npx tsc --noEmit` ve `npx eslint src scripts` temiz.
2. `QrScannerApp`: `npm run typecheck` ve `npx eslint src` temiz.
3. Sunucu davranışı değiştiyse uçtan uca test (`npm run test:e2e`, geliştirildiğinde) tamamen geçer; yeni kural için test eklenmiştir. Testler ortak veritabanında kendi verisini oluşturur ve temizler (§7.1).
4. Gizli bilgi yok (`git diff` kontrol edildi), ilgili belge güncellendi.
5. Dosyalar §3 hedef ağacındaki yerinde; diff §8.1'e göre gözden geçirildi (gereksiz/kullanılmayan kod yok, adlar açıklayıcı, fonksiyonlar kısa).
6. Dışa aktarılan adlar, mesaj metinleri, hata kodları ve davranış bu dosyadaki ve alt projelerin AGENTS.md'sindeki tanımlarla aynı; farklar yalnızca görevin/prompt'un açıkça istedikleri (§0.3).

## 10. Hızlı başlangıç

```bash
# 1) AdminPanel — ortak veritabanına bağlanma (Docker gerekmez)
cd AdminPanel && npm install
cp .env.example .env.local                  # DATABASE_URL'yi proje sahibinin verdiği adresle doldurun ve kendi AUTH_SECRET'inizi üretin (§5.1)
npm run db:check                            # "Bağlandı … SSL: açık" ve 12 tablo görmelisiniz (şemaya `feedbacks` eklendi)
npm run dev                                 # http://localhost:3000 → "Ortak veritabanına bağlı"; http://localhost:3000/api/health → JSON

# 2) Mobil (ayrı terminal) — telefon, bilgisayarınızdaki AdminPanel'e bağlanır
cd QrScannerApp && npm install
echo "EXPO_PUBLIC_API_URL=http://$(ipconfig getifaddr en0):3000" > .env
npx expo run:ios --device                   # veya: npx expo run:android --device (BLE için geliştirme derlemesi gerekir)
```
Ayrıntılar: `README.md` (veritabanına bağlanma, sorun giderme), `infra/dev-db/README.md` (sunucu tarafı).
