# DOQR — Okul Giriş-Çıkış Sistemi / School Attendance System

**[Türkçe](#türkçe)** · **[English](#english)**

---

<a id="türkçe"></a>

# Türkçe

Öğrencinin okula giriş ve çıkışını kaydeden açık kaynak sistem. Okul kapısındaki tablet (kiosk) **5 saniyede bir değişen imzalı bir QR kod** gösterir ve aynı anda **Bluetooth (BLE) ile kısa ömürlü bir jeton** yayınlar. Öğrenci kendi telefonundaki uygulamayla QR'ı okutur. Sunucu okutmayı doğrular, giriş/çıkış kaydını açar ve veliye WhatsApp bildirimi gönderir. Okul yönetimi her şeyi web panelinden izler.

```
 Android tablet (KIOSK)          Öğrenci telefonu (iOS/Android)        Tarayıcı (yönetici)
 dönen QR + BLE yayını           QR okut + BLE jetonunu dinle          web paneli
          └───────────── HTTPS ─────────┴─────────── HTTPS ───────────────┘
                                         ▼
                      AdminPanel (Next.js) — web paneli + /api/mobile/*
                                         ▼
                              PostgreSQL (yalnızca sunucu erişir)
```

| Klasör | İçerik |
| --- | --- |
| [`AdminPanel/`](AdminPanel) | Next.js 16: yönetici web paneli + mobil uygulamanın API'si (`/api/mobile/*`) + `Dockerfile` + `deploy/` (canlı sunucu dosyaları) |
| [`QrScannerApp/`](QrScannerApp) | Expo SDK 57 (React Native): tek uygulama, iki mod — **öğrenci** ve **kiosk** |
| [`infra/dev-db/`](infra/dev-db) | PostgreSQL 17 (Docker, SSL zorunlu, iki kullanıcı: şema sahibi + yalnızca veri okuyup yazan uygulama kullanıcısı) |

Protokol, API sözleşmesi ve iş kuralları: [`AGENTS.md`](AGENTS.md).

---

## 1. Gereksinimler

**Sunucu (zorunlu)**
- Linux sunucu (Ubuntu 24.04 ile denendi; AWS Lightsail, Hetzner, DigitalOcean vb.). En az **1 GB RAM**: 0,5 GB Next.js derlemesine yetmez.
- Docker + Docker Compose eklentisi.
- Açık portlar: **80** ve **443** (HTTPS). **5433** yalnızca sunucu dışından veritabanına bağlanmak isterseniz (yerel geliştirme, TablePlus vb.).
- Alan adı (ör. `giris.okulunuz.com`). Yoksa ücretsiz `sslip.io` kullanılabilir: IP `3.121.45.67` → `3-121-45-67.sslip.io`. HTTPS sertifikasını Caddy otomatik alır.

**Kiosk (zorunlu)**
- **Android tablet** (Android 8+, BLE yayını destekli). iPhone/iPad kiosk olamaz: iOS Bluetooth "service data" yayınlayamaz.

**Mobil uygulama derlemesi**
- [Expo](https://expo.dev) hesabı ve EAS CLI (`npx eas-cli@latest`). Derleme bulutta yapılır, Xcode/Android Studio gerekmez.
- Mağazaya yüklemek için: Apple Developer (99 $/yıl) ve/veya Google Play Console (25 $, tek sefer) hesabı.
- Expo Go yalnızca arayüzü denemeye yeter; Bluetooth için geliştirme veya mağaza derlemesi gerekir.

**İsteğe bağlı**
- Veli bildirimleri için [Insider](https://useinsider.com) WhatsApp API anahtarı ve onaylı bir şablon. Yoksa `WHATSAPP_PROVIDER=mock` kalır: mesajlar yalnızca veritabanına yazılır, panelde görünür, gönderilmez.
- Yerel geliştirme için Node.js 22+.

---

## 2. Fork'unuzda değiştirmeniz gerekenler

Depodaki bazı değerler bu projeyi geliştiren okula aittir. Kendi kurulumunuzda değiştirin:

| Ne | Nerede |
| --- | --- |
| Sunucu adresi (`https://doqrapp.com`) | `QrScannerApp/eas.json` → `preview` ve `production` profillerindeki `EXPO_PUBLIC_API_URL` |
| EAS proje kimliği | `QrScannerApp/app.json` → `expo.extra.eas.projectId`. Silin; `npx eas-cli@latest init` kendi kimliğinizi yazar |
| Uygulama adı, paket kimliği | `QrScannerApp/app.json` → `name`, `slug`, `scheme`, `ios.bundleIdentifier`, `android.package` (ör. `com.okulunuz.giris`). Mağazada başkasının paket kimliği kullanılamaz |
| Okul adı ve slogan | `AdminPanel/src/app/layout.tsx` (sayfa başlığı), `AdminPanel/src/app/(panel)/layout.tsx`, `AdminPanel/src/app/login/page.tsx`, `QrScannerApp/src/app/welcome.tsx`, `QrScannerApp/src/components/BrandHeader.tsx` |
| Destek ve gizlilik sayfasındaki iletişim bilgileri | `AdminPanel/src/app/(public)/school.ts` (`/destek` ve `/gizlilik` sayfaları; mağazalar bu adresleri ister) |
| Logolar | `QrScannerApp/assets/` (`logo.png`, `ifl-logo.png` okul amblemi, `icon.png`, `android-icon-*.png`, `splash-icon.png`) ve `AdminPanel/public/logo.png` |
| Renkler | `QrScannerApp/src/lib/config.ts` → `colors`; panel: `AdminPanel/src/app/globals.css` |

**Değiştirmeyin:** QR biçimi (`FB2`), `BLE_SERVICE_UUID` ve imza kuralları. Sunucu ile uygulama birbirini bunlarla tanır; değiştirirseniz iki tarafı birlikte güncellemeniz gerekir ([`AGENTS.md`](AGENTS.md) §4).

---

## 3. Sunucuya kurulum (adım adım)

Aşağıdaki komutlar sunucuda (SSH ile bağlanarak) çalıştırılır.

### 3.1 Docker'ı kurun
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit            # çıkıp tekrar bağlanın (docker grubunun etkinleşmesi için)
```

### 3.2 Kodu indirin
```bash
git clone https://github.com/yasirozcn/DOQR.git doqr
cd doqr
```

### 3.3 Veritabanını kurun
```bash
cd ~/doqr/infra/dev-db
bash kur.sh
```
`kur.sh` şunları yapar:
1. `.env` yoksa iki güçlü şifre üretir: `OWNER_PASSWORD` (şema sahibi) ve `APP_PASSWORD` (uygulama kullanıcısı).
2. PostgreSQL'i SSL zorunlu olarak başlatır.
3. `AdminPanel/db/schema.sql` şemasını uygular.

Tekrar çalıştırmak güvenlidir, veri silmez.

`infra/dev-db/.env` dosyasını yedekleyin ve kimseyle paylaşmayın.

### 3.4 Sunucu ayarları (swap, saat dilimi, gece yedeği)
```bash
cd ~/doqr/AdminPanel/deploy
bash sunucu-hazirla.sh
```
Bu betik 2 GB swap açar, saat dilimini `Europe/Istanbul` yapar ve her gece 03:30'da veritabanı yedeği alacak şekilde ayarlar (yedekler 14 gün saklanır).

### 3.5 Ortam değişkenleri
```bash
cd ~/doqr/AdminPanel/deploy
cp .env.example .env && chmod 600 .env
openssl rand -hex 32                          # çıktıyı AUTH_SECRET'e yazacaksınız
grep APP_PASSWORD ~/doqr/infra/dev-db/.env    # DATABASE_URL'deki <APP_PASSWORD> yerine yazacaksınız
nano .env
```

| Değişken | Açıklama |
| --- | --- |
| `DOMAIN` | Alan adınız (ör. `giris.okulunuz.com` veya `3-121-45-67.sslip.io`). DNS'te `A` kaydı sunucu IP'sini göstermeli |
| `AUTH_SECRET` | `openssl rand -hex 32` çıktısı. Tüm oturumları imzalar; değişirse herkes yeniden giriş yapar. **Zorunlu** |
| `DATABASE_URL` | `postgres://izmirfen_app:<APP_PASSWORD>@db:5432/izmirfen`. `db` konteyner adıdır, değiştirmeyin |
| `BLE_REQUIRED` | `true`: okutmada kapının Bluetooth jetonu zorunlu (önerilen; evden QR fotoğrafıyla okutmayı engeller). İlk denemelerde `false` bırakabilirsiniz |
| `WHATSAPP_PROVIDER` | `mock` (gönderim yok) veya `insider` (gerçek gönderim) |
| `INSIDER_API_KEY`, `WHATSAPP_TEMPLATE` | Yalnızca `insider` için: API anahtarı ve onaylı şablon adı |

### 3.6 Paneli başlatın
```bash
cd ~/doqr/AdminPanel/deploy
docker compose up -d --build        # ilk derleme 5–15 dk sürebilir
docker compose logs -f app          # "Ready" görünce Ctrl+C
```

### 3.7 İlk yönetici hesabı
Şifre gizli sorulur, en az 10 karakter:
```bash
docker compose exec -it app node scripts/create-admin.mjs --email yonetici@okulunuz.com --name "Ad Soyad" --role ADMIN
```
Aynı komut, aynı e-postayla tekrar çalıştırılırsa o hesabın şifresini değiştirir.

### 3.8 Kontrol
1. `https://<DOMAIN>/api/health` adresi `"db":"ok"` döndürmeli.
2. `https://<DOMAIN>/login` adresinde yönetici hesabıyla giriş yapabilmelisiniz.
3. `https://<DOMAIN>/api/mobile/config` adresi `{"bleRequired":…}` döndürmeli.
4. Sertifika hatası alırsanız: 80/443 açık mı, DNS kaydı doğru IP'yi mi gösteriyor? (`docker compose logs caddy`)

---

## 4. Paneli kullanıma hazırlama

Panelde (`https://<DOMAIN>`):
1. **Kiosklar → Kiosk ekle:** her kapı için bir kiosk (ör. "Ana Kapı"). Gizli anahtarı sunucu üretir.
2. **Kiosklar → Kiosk tablet hesabı:** tabletin giriş yapacağı hesap (e-posta + en az 10 karakter şifre). Bu hesap web paneline giremez, yalnızca tablette kullanılır.
3. **Öğrenciler → Öğrenci ekle** veya **CSV ile içe aktar:** ad, soyad, sınıf, cep telefonu (`05XX XXX XX XX`) ve en fazla 2 veli telefonu. Kayıt ekranı yoktur; yalnızca panelde tanımlı öğrenciler giriş yapabilir.

---

## 5. Mobil uygulama

### 5.1 Hazırlık
```bash
cd QrScannerApp
npm install
npx eas-cli@latest login
npx eas-cli@latest init          # Expo hesabınızda proje oluşturur, app.json'a projectId yazar
```
`eas.json` içindeki `preview` ve `production` profillerinde `EXPO_PUBLIC_API_URL` değerini `https://<DOMAIN>` yapın. Sunucu adresi derlemeye gömülür; değişirse yeniden derleme gerekir.

### 5.2 Kiosk tableti (Android APK)
```bash
npx eas-cli@latest build --platform android --profile preview
```
Bittiğinde EAS'ın verdiği bağlantıdan APK'yı tablete indirip kurun. Ardından tablette:
1. **Yönetici olarak devam et** → kiosk tablet hesabıyla giriş yapın.
2. Kapıyı seçin → **QR ekranını başlat**.

Tablet şarja takılı ve Bluetooth açık kalmalı. Ekran kendiliğinden kapanmaz.

### 5.3 Öğrenci uygulaması (mağazalar)
```bash
npx eas-cli@latest build --platform ios --profile production
npx eas-cli@latest build --platform android --profile production
npx eas-cli@latest submit --platform ios        # App Store Connect / TestFlight
npx eas-cli@latest submit --platform android    # Google Play
```
Mağaza formlarında gizlilik ve destek adresi olarak `https://<DOMAIN>/gizlilik` ve `https://<DOMAIN>/destek` kullanılabilir.

Öğrenci ilk girişte telefon numarasını yazar ve şifre oluşturur. Bu anda telefon hesaba bağlanır. Başka telefondan girmek için panelden **Cihazı sıfırla** gerekir.

---

## 6. Günlük işletme

```bash
cd ~/doqr && git pull
cd infra/dev-db && bash kur.sh                       # şema değiştiyse (veri silmez)
cd ~/doqr/AdminPanel/deploy && docker compose up -d --build   # yeni sürümü yayınla
docker compose logs --tail 100 app                   # sunucu logları
bash yedek-al.sh && ls -lh yedek/                    # elle yedek
```
Yedekleri ara sıra sunucu dışına da indirin; sunucu kaybolursa üzerindeki yedekler de gider.

**Veritabanı şifresini değiştirmek** (ör. biri ekipten ayrıldığında):
```bash
NEW=$(openssl rand -hex 24)
cd ~/doqr/infra/dev-db && \
docker compose exec -T db psql -U izmirfen_owner -d izmirfen -c "ALTER ROLE izmirfen_app PASSWORD '$NEW'" && \
sed -i "s/^APP_PASSWORD=.*/APP_PASSWORD=$NEW/" .env
cd ~/doqr/AdminPanel/deploy && \
sed -i "s#^DATABASE_URL=postgres://izmirfen_app:[^@]*@#DATABASE_URL=postgres://izmirfen_app:$NEW@#" .env && \
docker compose up -d --force-recreate app
unset NEW
```

---

## 7. Güvenlik kontrol listesi (gerçek öğrenci verisinden önce)

- [ ] `AUTH_SECRET` kendi ürettiğiniz rastgele değer; hiçbir yere kopyalanmadı.
- [ ] Port **5433** kapalı veya yalnızca bilinen IP'lere açık. Panel veritabanına Docker ağı üzerinden bağlandığı için kapatmak paneli bozmaz.
- [ ] `BLE_REQUIRED=true`.
- [ ] `deploy/.env` ve `infra/dev-db/.env` yalnızca sunucuda, `chmod 600`; depoya, sohbete veya ekran görüntüsüne girmedi.
- [ ] Gece yedeği çalışıyor (`ls ~/doqr/AdminPanel/deploy/yedek/`).
- [ ] Geliştirme ve canlı için ayrı veritabanı kullanıyorsunuz. Canlı veritabanına test verisi girilmez.

---

## 8. Yerel geliştirme

```bash
# 1) Panel + API
cd AdminPanel && npm install
cp .env.example .env.local      # DATABASE_URL (kendi veritabanınız, 5433 portu) ve AUTH_SECRET (openssl rand -hex 32)
npm run db:check                # "Bağlandı … SSL: açık" ve tablo listesi
npm run dev                     # http://localhost:3000

# 2) Mobil (ayrı terminal) — telefon ve bilgisayar aynı Wi-Fi'da
cd QrScannerApp && npm install
echo "EXPO_PUBLIC_API_URL=http://$(ipconfig getifaddr en0):3000" > .env
npx expo start --go             # Expo Go: hızlı arayüz denemesi (Bluetooth yok)
npx expo run:android --device   # geliştirme derlemesi (Bluetooth var)
```
Her değişiklikten sonra:
- `AdminPanel` içinde: `npm run typecheck && npx eslint src scripts`
- `QrScannerApp` içinde: `npm run typecheck && npx eslint src`

| `npm run db:check` hatası | Neden |
| --- | --- |
| `timeout` / `ECONNREFUSED` | Adres/port yanlış veya 5433 güvenlik duvarında kapalı |
| `no pg_hba.conf entry … no encryption` | `.env.local`'de `DATABASE_SSL=true` yok |
| `password authentication failed` | Şifre yanlış veya değiştirilmiş |
| `self-signed certificate` | `DATABASE_SSL_REJECT_UNAUTHORIZED=false` yok |

Katkı kuralları, katmanlar ve kod stili: [`AGENTS.md`](AGENTS.md), [`AdminPanel/AGENTS.md`](AdminPanel/AGENTS.md), [`QrScannerApp/AGENTS.md`](QrScannerApp/AGENTS.md).

---

## Lisans

[MIT](LICENSE) — kodu serbestçe kullanabilir, değiştirebilir ve dağıtabilirsiniz; telif satırı korunmalıdır.

---

<a id="english"></a>

# English

An open-source system that records students entering and leaving school. A tablet at the school gate (the kiosk) shows a **signed QR code that changes every 5 seconds** and at the same time broadcasts a **short-lived Bluetooth (BLE) token**. Students scan the QR with the app on their own phone. The server verifies the scan, records the entry/exit and notifies parents via WhatsApp. School staff monitor everything from a web panel.

```
 Android tablet (KIOSK)          Student phone (iOS/Android)           Browser (admin)
 rotating QR + BLE broadcast     scan QR + listen for BLE token        web panel
          └───────────── HTTPS ─────────┴─────────── HTTPS ───────────────┘
                                         ▼
                      AdminPanel (Next.js) — web panel + /api/mobile/*
                                         ▼
                              PostgreSQL (server access only)
```

| Folder | Contents |
| --- | --- |
| [`AdminPanel/`](AdminPanel) | Next.js 16: admin web panel + the mobile app's API (`/api/mobile/*`) + `Dockerfile` + `deploy/` (production server files) |
| [`QrScannerApp/`](QrScannerApp) | Expo SDK 57 (React Native): one app, two modes — **student** and **kiosk** |
| [`infra/dev-db/`](infra/dev-db) | PostgreSQL 17 (Docker, SSL required, two users: schema owner + an app user that can only read/write data) |

Protocol, API contract and business rules: [`AGENTS.md`](AGENTS.md) (in Turkish). The app and panel UI are in Turkish.

---

## 1. Requirements

**Server (required)**
- A Linux server (tested on Ubuntu 24.04; AWS Lightsail, Hetzner, DigitalOcean, etc.) with at least **1 GB RAM**. 0.5 GB is not enough to build Next.js.
- Docker + the Docker Compose plugin.
- Open ports: **80** and **443** (HTTPS). **5433** only if you want to reach the database from outside the server (local development, TablePlus, etc.).
- A domain name (e.g. `attendance.yourschool.com`). Without one you can use free `sslip.io`: IP `3.121.45.67` → `3-121-45-67.sslip.io`. Caddy obtains the HTTPS certificate automatically.

**Kiosk (required)**
- An **Android tablet** (Android 8+, with BLE advertising support). iPhone/iPad cannot be a kiosk: iOS cannot broadcast Bluetooth "service data".

**Building the mobile app**
- An [Expo](https://expo.dev) account and the EAS CLI (`npx eas-cli@latest`). Builds run in the cloud; no Xcode/Android Studio needed.
- To publish to stores: an Apple Developer account ($99/year) and/or a Google Play Console account ($25 one-time).
- Expo Go is only enough to try the UI; Bluetooth requires a development or store build.

**Optional**
- An [Insider](https://useinsider.com) WhatsApp API key and an approved template for parent notifications. Otherwise keep `WHATSAPP_PROVIDER=mock`: messages are only written to the database and shown in the panel, not sent.
- Node.js 22+ for local development.

---

## 2. What to change in your fork

Some values in the repository belong to the school that built this project. Change them for your own deployment:

| What | Where |
| --- | --- |
| Server URL (`https://doqrapp.com`) | `QrScannerApp/eas.json` → `EXPO_PUBLIC_API_URL` in the `preview` and `production` profiles |
| EAS project ID | `QrScannerApp/app.json` → `expo.extra.eas.projectId`. Delete it; `npx eas-cli@latest init` writes your own |
| App name, package ID | `QrScannerApp/app.json` → `name`, `slug`, `scheme`, `ios.bundleIdentifier`, `android.package` (e.g. `com.yourschool.attendance`). Stores do not accept someone else's package ID |
| School name and slogan | `AdminPanel/src/app/layout.tsx` (page title), `AdminPanel/src/app/(panel)/layout.tsx`, `AdminPanel/src/app/login/page.tsx`, `QrScannerApp/src/app/welcome.tsx`, `QrScannerApp/src/components/BrandHeader.tsx` |
| Contact info on the support and privacy pages | `AdminPanel/src/app/(public)/school.ts` (the `/destek` and `/gizlilik` pages; app stores require these URLs) |
| Logos | `QrScannerApp/assets/` (`logo.png`, `ifl-logo.png` school crest, `icon.png`, `android-icon-*.png`, `splash-icon.png`) and `AdminPanel/public/logo.png` |
| Colors | `QrScannerApp/src/lib/config.ts` → `colors`; panel: `AdminPanel/src/app/globals.css` |

**Do not change:** the QR format (`FB2`), `BLE_SERVICE_UUID` or the signing rules. Server and app recognize each other through these; if you change them, update both sides together ([`AGENTS.md`](AGENTS.md) §4).

---

## 3. Server installation (step by step)

Run the commands below on the server (over SSH).

### 3.1 Install Docker
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit            # log out and back in so the docker group takes effect
```

### 3.2 Get the code
```bash
git clone https://github.com/yasirozcn/DOQR.git doqr
cd doqr
```

### 3.3 Set up the database
```bash
cd ~/doqr/infra/dev-db
bash kur.sh
```
`kur.sh` does the following:
1. If `.env` is missing, it generates two strong passwords: `OWNER_PASSWORD` (schema owner) and `APP_PASSWORD` (app user).
2. Starts PostgreSQL with SSL required.
3. Applies the `AdminPanel/db/schema.sql` schema.

It is safe to run again; it does not delete data.

Back up `infra/dev-db/.env` and do not share it.

### 3.4 Server setup (swap, time zone, nightly backup)
```bash
cd ~/doqr/AdminPanel/deploy
bash sunucu-hazirla.sh
```
This script adds 2 GB of swap, sets the time zone to `Europe/Istanbul` and schedules a database backup every night at 03:30 (backups are kept for 14 days).

### 3.5 Environment variables
```bash
cd ~/doqr/AdminPanel/deploy
cp .env.example .env && chmod 600 .env
openssl rand -hex 32                          # put the output into AUTH_SECRET
grep APP_PASSWORD ~/doqr/infra/dev-db/.env    # put this into DATABASE_URL in place of <APP_PASSWORD>
nano .env
```

| Variable | Description |
| --- | --- |
| `DOMAIN` | Your domain (e.g. `attendance.yourschool.com` or `3-121-45-67.sslip.io`). The DNS `A` record must point to the server IP |
| `AUTH_SECRET` | Output of `openssl rand -hex 32`. Signs all sessions; if it changes, everyone has to log in again. **Required** |
| `DATABASE_URL` | `postgres://izmirfen_app:<APP_PASSWORD>@db:5432/izmirfen`. `db` is the container name; do not change it |
| `BLE_REQUIRED` | `true`: the gate's Bluetooth token is required to scan (recommended; prevents scanning a photo of the QR from home). You can leave it `false` during first tests |
| `WHATSAPP_PROVIDER` | `mock` (nothing sent) or `insider` (real delivery) |
| `INSIDER_API_KEY`, `WHATSAPP_TEMPLATE` | Only for `insider`: API key and approved template name |

### 3.6 Start the panel
```bash
cd ~/doqr/AdminPanel/deploy
docker compose up -d --build        # the first build can take 5–15 min
docker compose logs -f app          # press Ctrl+C once you see "Ready"
```

### 3.7 First admin account
The password is asked for hidden, at least 10 characters:
```bash
docker compose exec -it app node scripts/create-admin.mjs --email admin@yourschool.com --name "Full Name" --role ADMIN
```
Running the same command again with the same email changes that account's password.

### 3.8 Check
1. `https://<DOMAIN>/api/health` should return `"db":"ok"`.
2. You should be able to sign in with the admin account at `https://<DOMAIN>/login`.
3. `https://<DOMAIN>/api/mobile/config` should return `{"bleRequired":…}`.
4. Certificate error? Check that ports 80/443 are open and the DNS record points to the right IP (`docker compose logs caddy`).

---

## 4. Preparing the panel

In the panel (`https://<DOMAIN>`):
1. **Kiosklar → add kiosk:** one kiosk per gate (e.g. "Main Gate"). The server generates its secret key.
2. **Kiosklar → kiosk tablet account:** the account the tablet signs in with (email + password of at least 10 characters). This account cannot use the web panel; it is only for the tablet.
3. **Öğrenciler → add student** or **import CSV:** first name, last name, class, mobile number (`05XX XXX XX XX`) and up to 2 parent phone numbers. There is no sign-up screen; only students defined in the panel can sign in.

---

## 5. Mobile app

### 5.1 Preparation
```bash
cd QrScannerApp
npm install
npx eas-cli@latest login
npx eas-cli@latest init          # creates the project in your Expo account, writes projectId into app.json
```
In `eas.json`, set `EXPO_PUBLIC_API_URL` in the `preview` and `production` profiles to `https://<DOMAIN>`. The server URL is baked into the build; if it changes, rebuild.

### 5.2 Kiosk tablet (Android APK)
```bash
npx eas-cli@latest build --platform android --profile preview
```
When it finishes, download the APK to the tablet from the link EAS gives you and install it. Then on the tablet:
1. **Yönetici olarak devam et** (continue as admin) → sign in with the kiosk tablet account.
2. Choose the gate → **QR ekranını başlat** (start QR screen).

Keep the tablet plugged in with Bluetooth on. The screen does not turn off by itself.

### 5.3 Student app (stores)
```bash
npx eas-cli@latest build --platform ios --profile production
npx eas-cli@latest build --platform android --profile production
npx eas-cli@latest submit --platform ios        # App Store Connect / TestFlight
npx eas-cli@latest submit --platform android    # Google Play
```
In store forms you can use `https://<DOMAIN>/gizlilik` (privacy) and `https://<DOMAIN>/destek` (support).

On first sign-in the student enters their phone number and creates a password. At that moment the phone is bound to the account. Signing in from another phone requires **Cihazı sıfırla** (reset device) in the panel.

---

## 6. Day-to-day operation

```bash
cd ~/doqr && git pull
cd infra/dev-db && bash kur.sh                       # if the schema changed (does not delete data)
cd ~/doqr/AdminPanel/deploy && docker compose up -d --build   # deploy the new version
docker compose logs --tail 100 app                   # server logs
bash yedek-al.sh && ls -lh yedek/                    # manual backup
```
Copy backups off the server from time to time; if the server is lost, the backups on it go too.

**Changing the database password** (e.g. when someone leaves the team):
```bash
NEW=$(openssl rand -hex 24)
cd ~/doqr/infra/dev-db && \
docker compose exec -T db psql -U izmirfen_owner -d izmirfen -c "ALTER ROLE izmirfen_app PASSWORD '$NEW'" && \
sed -i "s/^APP_PASSWORD=.*/APP_PASSWORD=$NEW/" .env
cd ~/doqr/AdminPanel/deploy && \
sed -i "s#^DATABASE_URL=postgres://izmirfen_app:[^@]*@#DATABASE_URL=postgres://izmirfen_app:$NEW@#" .env && \
docker compose up -d --force-recreate app
unset NEW
```

---

## 7. Security checklist (before real student data)

- [ ] `AUTH_SECRET` is a random value you generated; it was not copied anywhere.
- [ ] Port **5433** is closed or open only to known IPs. The panel reaches the database over the Docker network, so closing it does not break the panel.
- [ ] `BLE_REQUIRED=true`.
- [ ] `deploy/.env` and `infra/dev-db/.env` exist only on the server with `chmod 600`; they never went into the repo, a chat or a screenshot.
- [ ] The nightly backup runs (`ls ~/doqr/AdminPanel/deploy/yedek/`).
- [ ] You use separate databases for development and production. No test data in the production database.

---

## 8. Local development

```bash
# 1) Panel + API
cd AdminPanel && npm install
cp .env.example .env.local      # DATABASE_URL (your database, port 5433) and AUTH_SECRET (openssl rand -hex 32)
npm run db:check                # "Bağlandı … SSL: açık" (connected, SSL on) and the table list
npm run dev                     # http://localhost:3000

# 2) Mobile (separate terminal) — phone and computer on the same Wi-Fi
cd QrScannerApp && npm install
echo "EXPO_PUBLIC_API_URL=http://$(ipconfig getifaddr en0):3000" > .env
npx expo start --go             # Expo Go: quick UI testing (no Bluetooth)
npx expo run:android --device   # development build (with Bluetooth)
```
After every change:
- in `AdminPanel`: `npm run typecheck && npx eslint src scripts`
- in `QrScannerApp`: `npm run typecheck && npx eslint src`

| `npm run db:check` error | Cause |
| --- | --- |
| `timeout` / `ECONNREFUSED` | Wrong address/port, or 5433 closed in the firewall |
| `no pg_hba.conf entry … no encryption` | `DATABASE_SSL=true` missing from `.env.local` |
| `password authentication failed` | Wrong or changed password |
| `self-signed certificate` | `DATABASE_SSL_REJECT_UNAUTHORIZED=false` missing |

Contribution rules, layers and code style: [`AGENTS.md`](AGENTS.md), [`AdminPanel/AGENTS.md`](AdminPanel/AGENTS.md), [`QrScannerApp/AGENTS.md`](QrScannerApp/AGENTS.md) (in Turkish).

---

## License

[MIT](LICENSE) — you may freely use, modify and distribute the code; the copyright notice must be kept.
