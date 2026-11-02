<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

# AGENTS.md — AdminPanel (web paneli + mobil API)

Önce kökteki [`../AGENTS.md`](../AGENTS.md) dosyasını okuyun: protokol, API sözleşmesi, iş kuralları ve **referans kaynak kodu (§0)** oradadır ve **bağlayıcıdır**.
Bu dosya AdminPanel'in nasıl kurulduğunu, **her dosyaya ne yazılacağını** ve hangi kurallarla geliştirildiğini anlatır.

**Uyarlamalar:** kök AGENTS.md §0.2'deki farkları uygulayın.
**Prompt önceliği:** görev referansta olmayan bir özellik veya var olan bir özelliğin değişmesini isterse prompt esastır (kök AGENTS.md §0.3). Değişmeyenler: `/api/mobile/*` sözleşmesi (yol, istek/yanıt, `{ error, code }`), JWT rolleri, `protocol.ts`, `scan.ts` doğrulama sırası ve BLE kontrolü, SQL'in yalnızca `repo.ts`'te olması, şema. Yeni sayfa/action/route eklenirse §2 ağacı ve §4–§5 tabloları aynı işte güncellenir.

---

## 1. Teknoloji

| Katman | Seçim |
| --- | --- |
| Çatı | **Next.js 16** (App Router, Turbopack, Server Components, Server Actions), React 19, TypeScript (strict) |
| Stil | Tailwind CSS 4 — tasarım belirteçleri ve yardımcı sınıflar `src/app/globals.css` içinde (`.btn`, `.btn-primary`, `.btn-secondary`, `.btn-danger`, `.input`, `.card`, `.table`, `.badge`, `.page-title`, `.page-sub`) |
| Veritabanı | PostgreSQL 17 — **AWS'deki ortak geliştirme veritabanı** (SSL, port 5433); sürücü `pg` (ORM yok, el yazımı SQL). Yerelde veritabanı kurulmaz |
| Doğrulama | `zod` (her API gövdesi ve her form) |
| Kimlik | `jose` (JWT HS256), `bcryptjs` (şifre özeti, maliyet 10) |
| Canlı | Docker (`output: "standalone"`) + Caddy (otomatik HTTPS) — `deploy/` (son görevler) |

Paket yöneticisi **npm** (`package-lock.json`). Tüm gerekli paketler **kurulu**; yeni paket eklenmez (referans da başka paket kullanmaz). Gerekirse önce proje sahibine sorun.

## 2. Klasör yapısı (hedef ağaç)

Geliştirme bittiğinde AdminPanel tam olarak budur. ✅ = depoda hazır (değiştirmeyin) · ⏳ = ekip geliştirecek.
Yollar ve dosya adları değiştirilmez; listede olmayan dosya eklenmeden önce proje sahibine sorulur.

```
AdminPanel/
├── AGENTS.md, CLAUDE.md ✅ · README.md ⏳ · .env.example ✅ · .gitignore ✅
├── package.json ✅ (uçtan uca test eklenirken "test:e2e" betiği eklenir) · package-lock.json ✅
├── next.config.ts ✅ · tsconfig.json ✅ · eslint.config.mjs ✅ · postcss.config.mjs ✅
├── Dockerfile ✅ · .dockerignore ✅
├── public/.gitkeep ✅ · public/logo.png (uygulama logosu, panel menüsü)
├── db/schema.sql ✅                          tek şema dosyası (yalnızca proje sahibi uygular)
├── deploy/ ✅                         docker-compose.yml (app + caddy; veritabanı infra/dev-db) · Caddyfile · .env.example · sunucu-hazirla.sh · yedek-al.sh
├── scripts/
│   ├── db-check.mjs ✅                        ortak veritabanı bağlantı kontrolü (bu projeye özgü)
│   ├── db-setup.mjs ✅                        şemayı uygular (DATABASE_OWNER_URL — yalnızca proje sahibi)
│   ├── create-admin.mjs ✅                    ADMIN/KIOSK hesabı açar (şifre gizli sorulur)
│   └── e2e-test.mjs ⏳                uçtan uca API testi (kendi verisini oluşturur ve siler)
└── src/
    ├── lib/
    │   ├── db/pg.ts ✅ · db/types.ts ✅
    │   ├── db/repo.ts ✅ (+ kiosk silme ve kiosk tablet hesapları için 4 fonksiyon eklendi (BE-5) — §3.2)
    │   ├── config.ts ⏳               ortam değişkenleri (tek yer)
    │   ├── auth.ts ⏳                 signToken, verifyToken, bearerClaims
    │   ├── api.ts ⏳                  ApiError, ok, handler, body, requireStudent, requireKioskAdmin, rateLimit, clientIp
    │   ├── whatsapp.ts ⏳     formatTime, formatDateTime, buildWhatsAppMessage, dispatchWhatsApp (mock | insider)
    │   ├── csv.ts                    parseCsv — öğrenci toplu ekleme (";"/"," ayraç, tırnak, BOM)
    │   ├── session.ts ⏳              createWebSession, destroyWebSession, getWebAdmin, requireWebAdmin
    │   ├── protocol.ts ✅             QR/BLE/imza — mobildeki ile birebir aynı kurallar
    │   ├── phone.ts ✅                normalizePhone, phoneField — öğrenci giriş kimliği (+905XXXXXXXXX)
    │   ├── dummy.ts GEÇİCİ           henüz verisi olmayan listelerin örnek satırları (kök AGENTS.md §0.4); tüm BE görevleri bitince SİLİNİR
    │   └── scan.ts ✅         REJECT_MESSAGES, ScanInput, ScanResult, processScan, manualEvent
    ├── components/
    │   ├── AutoRefresh.tsx ⏳          sayfayı N sn'de bir yeniler
    │   ├── Pagination.tsx            PAGE_SIZE (50), pageFrom(?sayfa), Pagination — uzun listeler
    │   ├── Icon.tsx                   çizgi ikonlar (Lucide yolları; paket yok) — panel düğmeleri ve menü
    │   └── Badges.tsx ⏳               DirectionBadge, SourceBadge
    └── app/
        ├── layout.tsx ✅ · globals.css ✅ (panel tasarımı yapılırken referans tasarımına güncellenir) · icon.png + apple-icon.png (sekme ikonu = uygulama ikonu)
        ├── page.tsx ✅ GEÇİCİ — panel ana ekranı yazılınca SİLİNİR ("/" artık (panel)/page.tsx)
        ├── api/health/route.ts ✅              ortak veritabanı sağlık kontrolü (kalır)
        ├── login/
        │   ├── page.tsx ⏳ · LoginForm.tsx ⏳
        │   └── actions.ts ⏳           loginAction, logoutAction
        ├── (public)/ ✅                herkese açık, giriş gerektirmez (mağaza Support / Privacy Policy URL'leri)
        │   ├── layout.tsx · school.ts  ortak çerçeve + okul/destek iletişim bilgileri
        │   └── destek/page.tsx · gizlilik/page.tsx
        ├── (panel)/
        │   ├── layout.tsx ⏳           requireWebAdmin + sol menü + kullanıcı kutusu + Çıkış yap
        │   ├── NavLinks.tsx ⏳         menü bağlantıları (aktif bağlantı vurgulu, ikonlu)
        │   ├── actions.ts ⏳ TÜM panel Server Action'ları (§4.2)
        │   ├── page.tsx ⏳             Canlı durum
        │   ├── ogrenciler/page.tsx ⏳ · ogrenciler/StudentForms.tsx ⏳ · ogrenciler/DeleteStudentButton.tsx
        │   ├── kiosklar/page.tsx ⏳ · kiosklar/DeleteKioskButton.tsx ⏳ · kiosklar/AccountForms.tsx ⏳
        │   ├── hareketler/page.tsx ⏳
        │   ├── denemeler/page.tsx ⏳ · denetim/page.tsx ⏳
        │   ├── whatsapp/page.tsx ⏳
        │   └── geri-bildirimler/page.tsx   öğrenci geri bildirimleri (Yapılacak / Tamamlandı)
        └── api/mobile/
            ├── config/route.ts ⏳
            ├── student/check-phone/route.ts ✅ · student/set-password/route.ts ⏳
            ├── student/login/route.ts ⏳ · student/me/route.ts ⏳
            ├── scan/route.ts ✅
            ├── feedback/route.ts         öğrenci geri bildirimi (POST)
            ├── admin/login/route.ts ⏳
            └── kiosks/route.ts ✅ · kiosks/[id]/start/route.ts ✅ · kiosks/[id]/feed/route.ts ✅
```

**Referansta olup bu projeye EKLENMEYEN dosyalar:** `AdminPanel/docker-compose.yml` (yerel veritabanı), `POSTGRESQL_KURULUM.md`, `scripts/json-to-postgres.mjs`, `pnpm-lock.yaml`. `package.json`'a `db:up`, `db:down`, `db:reset`, `db:psql`, `db:import` eklenmez.

### Katmanlar — veri nereden nereye akar

```
Tarayıcı ──(form)──▶ Server Action (src/app/(panel)/actions.ts, login/actions.ts)   ─┐
Tarayıcı ──(sayfa)─▶ Server Component (src/app/(panel)/**/page.tsx)                  ├─▶ src/lib/*.ts (iş kuralı) ─▶ src/lib/db/repo.ts ─▶ q()/tx() ─▶ pg havuzu ─SSL─▶ AWS PostgreSQL
Telefon ──(JSON)───▶ Route Handler (src/app/api/mobile/**/route.ts)                  ─┘
```
- Sayfa, Server Action ve route **ince** kalır: girdi doğrula → `lib` / `repo` fonksiyonu çağır → sonucu döndür.
- **SQL yalnızca `src/lib/db/repo.ts` içinde** yazılır. Başka hiçbir dosya `pg`, `q()` veya `tx()` içe aktarmaz.
- Okutma iş kuralları yalnızca `src/lib/scan.ts`'tedir; protokol hesapları yalnızca `src/lib/protocol.ts`'tedir.

```ts
// Örnek: src/app/api/mobile/kiosks/route.ts (referanstaki dosyanın aynısı)
import { handler, ok, requireKioskAdmin } from "@/lib/api";
import { listKiosks } from "@/lib/db/repo";

export const GET = handler(async (req: Request) => {
  await requireKioskAdmin(req);
  const kiosks = await listKiosks();
  return ok({ kiosks: kiosks.filter((k) => k.status === "ACTIVE").map((k) => ({ id: k.id, name: k.name })) });
});
```

## 3. Veri katmanı

### 3.1 Kurallar
- Her zaman **parametreli sorgu** (`$1, $2`); kullanıcı girdisi SQL metnine eklenmez.
- Birden çok tabloya yazan işlemler `tx()` içinde: `createStudent`, `setFirstPasswordAndBindDevice` (öğrenci satırı `FOR UPDATE` ile kilitlenir), `resetStudentDevice`, `createEventWithWhatsApp`.
- Satırlar `camel()` ile camelCase'e, `Date` değerleri ISO metne çevrilir. `bigint` (`time_slot`) sayı döner.
- `insertScanAttempt`: kabul edilmiş aynı (cihaz, kiosk, dilim) ikinci kez yazılırsa tekil indeks hatası (`23505`) yakalanır ve kayıt `REJECTED/REPLAY` olarak yazılır.
- `dashboardStats`: "bugün" Türkiye saatine göre.
- Kimlikler `newId("stu")` → `stu_` + 16 hex.

### 3.2 Repo fonksiyonları (`src/lib/db/repo.ts`)

**Hazır olanlar** (yeniden yazmayın, yalnızca çağırın):
`newId, findStudentByPhone, getStudent, listStudents (StudentRow), createStudent, createStudents + findTakenStudentPhones (CSV), deleteStudent, setStudentPassword, getActiveDevice, getActiveDeviceForStudent, BindError, setFirstPasswordAndBindDevice, resetStudentPassword, resetStudentDevice, findAdminByEmail, getAdmin, listKiosks, getKiosk, touchKiosk, createKiosk, setKioskStatus, setKioskWhatsApp, insertScanAttempt, isReplay, lastEventForStudent(studentId, kioskId) (kapı bazında), listDoorPresence (her öğrencinin her kapıdaki son hareketi), createEventWithWhatsApp, updateWhatsAppMessage, getEvent, reviewEvent, listEvents (EventRow), listScanAttempts (AttemptRow), listKioskAttempts, listWhatsAppMessages, countEvents + countWhatsAppMessages (sayfalama), dashboardStats, addAudit, listAudit, createFeedback, listFeedbacks (FeedbackRow), completeFeedback, dbHealth`.

**Eklenen 4 fonksiyon (BE-5)** (referanstaki `repo.ts` ile birebir aynı; ekleme yerleri referanstaki gibi):
```ts
// "yöneticiler" bölümüne, getAdmin'den sonra (kiosk tablet hesapları)
/** Kiosk tableti hesapları (şifre özeti olmadan). */
export function listKioskAccounts(): Promise<Pick<AdminUser, "id" | "fullName" | "email">[]> {
  return q("SELECT id, full_name, email FROM admin_users WHERE role = 'KIOSK' ORDER BY full_name");
}

export async function createKioskAccount(fullName: string, email: string, passwordHash: string): Promise<AdminUser> {
  const account: AdminUser = { id: newId("adm"), fullName: fullName.trim(), email: email.trim(), role: "KIOSK", passwordHash, twoFactorEnabled: false };
  await tx((c) => insertRow(c, "admin_users", account));
  return account;
}

/** Yalnızca KIOSK rolündeki hesabın şifresini değiştirir; hesap yoksa false döner. */
export async function setKioskAccountPassword(id: string, passwordHash: string): Promise<boolean> {
  const rows = await q("UPDATE admin_users SET password_hash = $2 WHERE id = $1 AND role = 'KIOSK' RETURNING id", [id, passwordHash]);
  return rows.length > 0;
}

// "kiosklar" bölümüne, setKioskStatus'tan sonra (kiosk silme)
/** Kiosku siler. Geçmiş hareket ve okutma kayıtları korunur (kiosk_id NULL olur, FK ON DELETE SET NULL). */
export async function deleteKiosk(id: string): Promise<boolean> {
  const rows = await q("DELETE FROM kiosks WHERE id = $1 RETURNING id", [id]);
  return rows.length > 0;
}
```

### 3.3 Tablolar (`db/schema.sql` — hazır, değiştirmeyin)

| Tablo | Amaç / önemli kolonlar | Kısıtlar |
| --- | --- | --- |
| `students` | school_no, first_name, last_name, phone (giriş kimliği, `+905XXXXXXXXX`), email (isteğe bağlı), class_name, password_hash (null = henüz şifre yok), presence_status `IN/OUT` (varsayılan OUT), is_active | `phone` tekil, `lower(email)` tekil (ikisi de boş olabilir; telefonu olmayan öğrenci giriş yapamaz) |
| `guardians` | full_name, phone | |
| `student_guardians` | student_id, guardian_id, relation, notify_entry, notify_exit | PK (student_id, guardian_id); öğrenci silinince CASCADE |
| `devices` | id = uygulamanın ürettiği deviceId, student_id, platform, device_secret, status `ACTIVE/REVOKED`, bound_at, revoked_at, revoked_by | öğrenci başına tek ACTIVE (kısmi tekil indeks); öğrenci silinince CASCADE |
| `kiosks` | name, secret (32 bayt hex), status `ACTIVE/DISABLED`, last_seen_at, whatsapp_enabled (varsayılan true; false ise veliye mesaj yazılmaz) | **yön kolonu yok** |
| `scan_attempts` | device_id, student_id, kiosk_id, time_slot, ble_token, ble_rssi, ble_ok, integrity_ok, result `ACCEPTED/REJECTED`, reject_reason | (device_id, kiosk_id, time_slot) ACCEPTED için tekil; öğrenci/kiosk silinince SET NULL |
| `attendance_events` | student_id, direction `IN/OUT`, occurred_at, source `APP/MANUAL/OFFLINE`, kiosk_id, scan_attempt_id, review_status `UNREVIEWED/OK/SUSPICIOUS`, reviewed_by, note | öğrenci silinince CASCADE; kiosk silinince SET NULL. `review_status`/`reviewed_by` kullanılmaz (hareket inceleme yok, kök AGENTS.md §0.2): her kayıt `UNREVIEWED` kalır |
| `wp_messages` | event_id, guardian_id, phone, body, status `QUEUED/SENT/DELIVERED/FAILED/MOCK_SENT`, provider_message_id, attempt_count, sent_at, delivered_at | (event_id, guardian_id) tekil; olay silinince CASCADE |
| `permissions` | izin kayıtları (kullanılmıyor) | |
| `admin_users` | full_name, email, role `ADMIN/KIOSK`, password_hash, two_factor_enabled | `lower(email)` tekil |
| `feedbacks` | student_id, message, status `TODO/DONE` (varsayılan TODO), created_at, completed_at, completed_by (admin id) | öğrenci silinince CASCADE · id `fbk_…` |
| `audit_logs` | admin_user_id (FK değil), action, entity, entity_id, before_value/after_value (jsonb) | |

**Ekip kullanıcısının şema yetkisi yoktur.** Bu projenin görevleri şema değişikliği gerektirmez.

## 4. Web paneli

Oturum: `/login` (Server Action, bcrypt). Yalnızca `role = ADMIN` girer; KIOSK hesabı panele giremez. Oturum 12 saat, `fb_admin` httpOnly çerezi.
Her yönetici işlemi Server Action ile yapılır, **önce** `requireWebAdmin()` çağrılır, işlemden sonra `addAudit(...)` yazılır ve `revalidatePath(...)` çağrılır.
Sayfalar sunucu bileşenidir, veriyi doğrudan repo'dan okur ve `export const dynamic = "force-dynamic";` içerir. Her panel sayfası veriyi okumadan önce `await requireWebAdmin();` çağırır: layout istemci içi gezinmede yeniden çalışmaz, tek başına yetki kontrolü sayılmaz (`node_modules/next/dist/docs/01-app/02-guides/authentication.md` → "Layouts and auth checks"; kök AGENTS.md §0.2). `"use client"` yalnızca form durumu (`useActionState`), `usePathname`, `confirm()` ve otomatik yenileme için.

### 4.1 Sayfalar

| Yol | Dosya | Veri (repo) | İçerik |
| --- | --- | --- | --- |
| `/login` | `login/page.tsx` + `LoginForm.tsx` | `getWebAdmin` | Oturum varsa `/`'e yönlendir. Solda koyu tanıtım paneli (geniş ekranda): logo `İF`, "İzmir Fen", "Giriş-Çıkış Yönetim Paneli", büyük başlık "Kim okulda, kim çıktı — tek bakışta.". Sağda "Giriş-Çıkış Yönetim Paneli", "Yönetici hesabınızla giriş yapın.", form: E-posta, Şifre, "Giriş yap" / "Giriş yapılıyor…" |
| `/destek` | `(public)/destek/page.tsx` | — (statik) | Giriş gerektirmez. Uygulamanın tanımı, sık sorulanlar (giriş, telefon değiştirme/cihaz sıfırlama, kamera ve Bluetooth izni, kiosk için Android tablet), iletişim (destek e-postası, okul telefonu, okul sitesi — `(public)/school.ts`) |
| `/gizlilik` | `(public)/gizlilik/page.tsx` | — (statik) | Giriş gerektirmez. Gizlilik politikası: veri sorumlusu okul; işlenen veriler (telefon, kiosk e-postası, cihaz kimliği, giriş-çıkış kayıtları) mobil privacy manifest ve App Privacy beyanıyla aynı; konum toplanmaz; saklama bir eğitim yılı; KVKK m.11 hakları |
| `/` | `(panel)/page.tsx` | `dashboardStats()`, `listEvents({ limit: 15 })`, `listKiosks()`, `listDoorPresence()` | "Canlı durum" + `<AutoRefresh seconds={10} />`; kartlar: her aktif kiosk için bir kart (kiosk adı, o kapıda içerideki öğrenci sayısı, "içeride · `total` öğrenciden", doluluk çubuğu), **Bugünkü hareket** (`todayEvents`, "bugün"), **Reddedilen okutma** (`rejectedToday`, "bugün"; >0 ise kırmızı kart). "Son hareketler" tablosu + "Tümünü gör →" `/hareketler` |
| `/hareketler` | `(panel)/hareketler/page.tsx` | `countEvents()`, `listEvents({ limit: PAGE_SIZE, offset })` | Sayfa başına 50 kayıt (`?sayfa=N`, `components/Pagination.tsx`: "N kayıttan a–b arası", Önceki/Sonraki + sayfa numaraları). Tablo (inceleme düğmesi/filtresi yok, kök AGENTS.md §0.2); manuel/çevrimdışı kayıtta kaynak rozeti; not varsa "Not: …" |
| `/ogrenciler` | `(panel)/ogrenciler/page.tsx` + `StudentForms.tsx` | `listStudents()`, `listKiosks()`, `listDoorPresence()` | Üstte arama (`?q=`; ad, soyad, okul no, sınıf, e-posta, öğrenci/veli adı ve telefonu — telefon boşluklu/0'lı yazılabilir; "Temizle", eşleşen sayısı). Tablo (Öğrenci · Telefon · Veli · Durum · Cihaz · İşlemler; İşlemler'de **Sil** — onay penceresiyle) + "Yeni öğrenci ekle" formu (en fazla 2 veli; "+ İkinci veli ekle") + "CSV ile toplu ekle" (şablon indirme; sütunlar `okul_no;ad;soyad;sinif;telefon;eposta;veli1_ad;veli1_telefon;veli2_ad;veli2_telefon`). Durum: her aktif kiosk için kiosk adı + "İçeride"/"Dışarıda"/"Kayıt yok" + **Manuel giriş/çıkış** düğmesi (kayıt yoksa ikisi de) |
| `/kiosklar` | `(panel)/kiosklar/page.tsx` + `DeleteKioskButton.tsx` + `AccountForms.tsx` | `listKiosks()`, `listKioskAccounts()` | Kiosk tablosu (Ad · Durum · Son sinyal · WhatsApp "Açık"/"Kapalı" + aç/kapat düğmesi (`toggleKioskWhatsAppAction`) · Devre dışı bırak/Etkinleştir + Sil), "Yeni kiosk" formu, "Kiosk tablet hesapları" tablosu (Ad · E-posta · şifre sıfırlama formu) + "Yeni kiosk hesabı" formu |
| `/denemeler` | `(panel)/denemeler/page.tsx` | `listScanAttempts(true, 300)` | "Reddedilen okutmalar": Zaman · Öğrenci · Kiosk · Neden (kod rozeti + `REJECT_MESSAGES` açıklaması) · BLE ("doğru"/"hatalı" + " · -60 dBm", jeton yoksa "yok") |
| `/whatsapp` | `(panel)/whatsapp/page.tsx` | `countWhatsAppMessages()`, `listWhatsAppMessages(PAGE_SIZE, offset)` | Sayfa başına 50 kayıt (`?sayfa=N`, `Pagination`). Açıklama: `mock` ise "Test modu: WhatsApp mesajları gerçekten gönderilmez, yalnızca burada ve sunucu konsolunda görünür." değilse "Sağlayıcı: …". Tablo: Zaman · Öğrenci · Telefon · Mesaj · Durum (FAILED kırmızı, QUEUED sarı, diğerleri yeşil) |
| `/geri-bildirimler` | `(panel)/geri-bildirimler/page.tsx` | `listFeedbacks()` | "Geri bildirimler": iki bölüm **Yapılacak** ve **Tamamlandı** (sayılarıyla). Her kayıt: öğrenci adı soyadı, sınıf, telefon, gönderim zamanı, mesaj; Yapılacak'ta **Tamamlandı** düğmesi (`completeFeedbackAction`) kaydı alt bölüme taşır, Tamamlandı'da tamamlanma zamanı |
| `/denetim` | `(panel)/denetim/page.tsx` | `listAudit(300)` | Zaman · Kim (`adminName ?? "Öğrenci uygulaması"`) · İşlem · Kayıt (`entity · entityId`) · Ayrıntı (`afterValue` JSON, tek satır kısaltılmış) |

Menü (`NavLinks.tsx`, bu sırayla): Canlı durum `/` · Giriş-çıkışlar `/hareketler` · Öğrenciler `/ogrenciler` · Kiosklar `/kiosklar` · Reddedilen okutmalar `/denemeler` · WhatsApp kayıtları `/whatsapp` · Geri bildirimler `/geri-bildirimler` · Denetim kaydı `/denetim`. Aktif bağlantı: `/` için tam eşleşme, diğerleri `startsWith`. İkon yolları referanstaki `NavLinks.tsx`'ten alınır.

Tarih/saat gösterimi her yerde `formatDateTime(iso)` (`src/lib/whatsapp.ts`).

### 4.2 Server Action'lar (`src/app/(panel)/actions.ts` — dosyanın başında `"use server";`)

| Fonksiyon | Form alanları | Repo | Audit `action` (entity) | `revalidatePath` | Not |
| --- | --- | --- | --- | --- | --- |
| `resetPasswordAction(form)` | `studentId` | `resetStudentPassword` | `PASSWORD_RESET` (student) | `/ogrenciler` | Cihaz bağlı kalır |
| `resetDeviceAction(form)` | `studentId` | `resetStudentDevice(id, admin.id)` | `DEVICE_RESET` (student) | `/ogrenciler` | Cihaz REVOKED + şifre silinir |
| `createStudentAction(prev, form)` | `schoolNo, firstName, lastName, className, phone, email (isteğe bağlı), guardian1Name, guardian1Phone, guardian2Name, guardian2Phone` | `createStudent` (telefon `phoneField` ile çevrilir) | `STUDENT_CREATED` (student) `{ phone }` | `/ogrenciler` | Dönüş `{ error?, ok? }`; zod hatası → "Lütfen zorunlu alanları doğru doldurun."; e-posta/telefon tekrarı → Türkçe mesaj; diğer hatalar → "Kayıt oluşturulamadı." (ayrıntı yalnızca sunucu logunda, `[db]`); adı yazılan velinin telefonu yoksa / iki veli telefonu aynıysa Türkçe mesaj |
| `importStudentsAction(prev, form)` | `file` (CSV, UTF-8) | `findTakenStudentPhones` + `createStudents` (tek transaction) | her öğrenci için `STUDENT_CREATED` `{ phone, source: "csv" }` | `/ogrenciler` | Dönüş `{ error?, ok?, count? }`; önce tüm satırlar doğrulanır (zorunlu alanlar, telefon biçimi, dosya içinde/veritabanında tekrar), tek hata varsa hiçbir öğrenci eklenmez ve satır numaralı hatalar (ilk 10) gösterilir |
| `deleteStudentAction(form)` | `studentId` | `deleteStudent` (cihaz, giriş-çıkış ve WhatsApp kayıtları CASCADE silinir; başka öğrenciye bağlı olmayan veliler de silinir) | `STUDENT_DELETED` (student) before `{ name, schoolNo, className, phone }`, after `null` | `/`, `/ogrenciler`, `/hareketler` | Öğrenci yoksa çık |
| `manualEventAction(form)` | `studentId, kioskId, direction, note` | `getStudent` + `getKiosk` + `lastEventForStudent` + `manualEvent(studentId, kiosk, direction, note)` (scan.ts) | `MANUAL_EVENT` (attendance_event) `{ direction, kioskId, note }` | `/`, `/ogrenciler`, `/hareketler` | Not boşsa "Manuel kayıt"; yön o kapıdaki durumla aynıysa sessizce çık |
| `createKioskAction(form)` | `name` | `createKiosk(name)` | `KIOSK_CREATED` (kiosk) `{ name }` | `/kiosklar` | Ad boşsa çık |
| `toggleKioskAction(form)` | `kioskId, status` | `setKioskStatus` | `KIOSK_STATUS` (kiosk) `{ status }` | `/kiosklar` | |
| `toggleKioskWhatsAppAction(form)` | `kioskId, whatsappEnabled ("true"/"false")` | `setKioskWhatsApp` | `KIOSK_WHATSAPP` (kiosk) `{ whatsappEnabled }` | `/kiosklar` | Kapalıyken giriş-çıkış kaydı tutulur, veliye mesaj yazılmaz |
| `deleteKioskAction(form)` | `kioskId` | `getKiosk` + `deleteKiosk` | `KIOSK_DELETED` (kiosk) before `{ name, status }`, after `null` | `/kiosklar` | Kiosk yoksa çık |
| `completeFeedbackAction(form)` | `feedbackId` | `completeFeedback(id, admin.id)` (yalnızca `TODO` olanı `DONE` yapar) | `FEEDBACK_COMPLETED` (feedback) before `{ status: "TODO" }`, after `{ status: "DONE" }` | `/geri-bildirimler` | Zaten tamamlandıysa çık |
| `createKioskAccountAction(prev, form)` | `fullName, email, password` | `findAdminByEmail` + `createKioskAccount(fullName, email, bcrypt.hash(pw,10))` | `KIOSK_ACCOUNT_CREATED` (admin_user) `{ email }` | `/kiosklar` | zod: "Ad zorunlu.", "Geçerli bir e-posta girin.", "Şifre en az 10 karakter olmalı."; e-posta varsa "Bu e-posta zaten kayıtlı." |
| `resetKioskAccountPasswordAction(prev, form)` | `accountId, password` | `setKioskAccountPassword` | `KIOSK_ACCOUNT_PASSWORD_RESET` (admin_user) | `/kiosklar` | <10 → "Şifre en az 10 karakter olmalı."; bulunamazsa "Kiosk hesabı bulunamadı." |

`src/app/login/actions.ts`: `loginAction(prev, form)` (audit `WEB_LOGIN`, entity `admin_user`) ve `logoutAction()`.

**Tüm audit işlem adları:** `WEB_LOGIN, KIOSK_LOGIN, KIOSK_START, PASSWORD_CREATED (adminUserId null), PASSWORD_RESET, DEVICE_RESET, STUDENT_CREATED, STUDENT_DELETED, MANUAL_EVENT, KIOSK_CREATED, KIOSK_STATUS, KIOSK_WHATSAPP, KIOSK_DELETED, KIOSK_ACCOUNT_CREATED, KIOSK_ACCOUNT_PASSWORD_RESET, FEEDBACK_COMPLETED`. Başka ad uydurulmaz.

### 4.3 Tasarım

`globals.css`: DOQR uygulamasıyla aynı palet (ana renk `#006160`, koyu `#004746`, açık `#DDEFEE`, sıcak nötrler `#F7F6F2`/`#E2DFD6`/`#14211A`, çıkış mavi `#1e4fa8`, uyarı amber, hata `#D04A3B`, manuel mor; tasarım kaynağı `.superdesign/design-system.md`). Bileşen sınıfları `@layer components` içinde: `.btn` (+ `-primary/-secondary/-danger/-sm`), `.btn-icon`, `.btn-icon-danger`, `.input`, `.card`, `.table`, `.badge`, `.page-title`, `.page-sub`, `.section-title`. Her düğmede `Icon` ile 16px ikon; satır işlemleri adı `title`/`aria-label`'da olan ikon düğmeleri. Tablolar masaüstünde (≥1280px) yatay taşmaz. Rozetler (`Badges.tsx`): Giriş "→ Giriş" yeşil, Çıkış "← Çıkış" mavi; kaynak MANUAL "Manuel" / OFFLINE "Çevrimdışı" mor, APP gösterilmez.

## 5. Mobil API

### 5.1 Uygulama kuralları
- Her route `export const GET|POST = handler(async (req, ctx?) => { … })` biçimindedir; gövde `await body(req, zodSchema)`; hata `throw new ApiError(status, "Türkçe mesaj", "KOD")`.
- Dinamik route parametresi Next.js 16'da Promise'tir: `ctx: { params: Promise<{ id: string }> }` → `const { id } = await ctx.params;`.
- Kimlik: `requireStudent(req)` / `requireKioskAdmin(req)`. `GET /student/me` ayrıca cihazın hâlâ ACTIVE ve o öğrenciye bağlı olduğunu kontrol eder (`DEVICE_REVOKED`).
- `rateLimit` (bellek içi, 10 dk pencere): check-phone 30/IP · set-password 8/telefon + 30/IP · login 10/telefon + 50/IP · admin login 10/e-posta + 30/IP. Anahtarlar: `check:<ip>`, `setpw:<phone>`, `setpw-ip:<ip>`, `login:<phone>`, `login-ip:<ip>`, `admin:<email>`, `admin-ip:<ip>` (telefon +905… biçiminde, e-posta küçük harf).
- Okutma kuralları yalnızca `scan.ts`'te; route ince kalır. Her okutma sunucu loguna tek satır: `[scan] KABUL (IN) · öğrenci … · kiosk … · BLE <jeton> RSSI -60 → EŞLEŞTİ`.
- WhatsApp gönderimi yanıtı bekletmez: `void dispatchWhatsApp(messages)`.

### 5.2 Hata mesajları kataloğu (birebir bu metinler)

| Yer | Durum · `code` | Mesaj |
| --- | --- | --- |
| `body()` | 400 `BAD_JSON` | İstek gövdesi JSON olmalı. |
| `handler()` zod | 400 `VALIDATION` | Geçersiz istek. (+ `issues`) |
| `handler()` diğer | 500 `ERROR` | Beklenmeyen hata. (iç hata mesajı istemciye dönmez, `[api]` ile loglanır) |
| `requireStudent` | 401 `UNAUTHORIZED` | Oturum süresi doldu. Tekrar giriş yapın. |
| `requireKioskAdmin` | 401 `UNAUTHORIZED` | Yönetici oturumu gerekli. |
| `rateLimit` | 429 `RATE_LIMIT` | Çok fazla deneme yapıldı. Birkaç dakika sonra tekrar deneyin. |
| check-phone / set-password / login (öğrenci) | 400 `VALIDATION` (zod) | Geçerli bir cep telefonu numarası girin. (issues içinde) |
| check-phone / set-password | 404 `NOT_FOUND` | Bu telefon numarası okul kayıtlarında bulunamadı. |
| check-phone | 409 `DEVICE_TAKEN` | Bu telefon başka bir öğrenci hesabına bağlı. Bir telefonda yalnızca bir öğrenci hesabı kullanılabilir. |
| check-phone | 403 `WRONG_DEVICE` | Hesabınız başka bir telefona bağlı. Telefon değiştirdiyseniz okul yönetiminden cihaz sıfırlaması isteyin. |
| set-password | 400 `VALIDATION` (zod) | Şifre en az 8 karakter olmalı. (issues içinde) |
| set-password | `BindError` kodu → 404/409/409/403 | `BindError` mesajı (repo.ts'te hazır) |
| login (öğrenci) | 401 `BAD_CREDENTIALS` | Telefon numarası veya şifre hatalı. |
| admin login | 401 `BAD_CREDENTIALS` | E-posta veya şifre hatalı. |
| login (öğrenci) | 403 `WRONG_DEVICE` | Hesabınız başka bir cihaza bağlı. Telefon değiştirdiyseniz okul yönetiminden cihaz sıfırlaması isteyin. |
| me | 401 `DEVICE_REVOKED` | Bu cihazın bağlantısı kaldırılmış. Tekrar kayıt olmanız gerekiyor. |
| me | 404 `NOT_FOUND` | Öğrenci bulunamadı. |
| kiosks/[id]/start | 404 `NOT_FOUND` | Kiosk bulunamadı. |
| scan | 422 `<RejectReason>` | `REJECT_MESSAGES[reason]` (aşağıda) |
| panel girişi (Server Action) | — | E-posta veya şifre hatalı. · Bu hesap yalnızca kiosk (QR) ekranı içindir; panele giriş yetkisi yok. |

`REJECT_MESSAGES` (`scan.ts`; **WRONG_STATE yok**, kök AGENTS.md §0.2):
`INVALID_QR` Bu QR kod okul kioskuna ait değil. · `EXPIRED_QR` QR kodun süresi doldu. Ekrandaki güncel kodu okutun. · `UNKNOWN_KIOSK` Kiosk tanınmadı veya devre dışı. · `BAD_SIGNATURE` İstek doğrulanamadı. Uygulamayı güncelleyin veya tekrar giriş yapın. · `DEVICE_NOT_BOUND` Bu cihaz hesabınıza bağlı değil. · `BLE_MISSING` Kiosk Bluetooth sinyali algılanamadı. Bluetooth'u açıp kioska yaklaşın. · `BLE_MISMATCH` Kiosk Bluetooth sinyali doğrulanamadı. Kioska yaklaşıp tekrar deneyin. · `REPLAY` Bu kod bu cihazdan zaten okutuldu. Bir sonraki kodu bekleyin. · `DIRECTION_REQUIRED` Bu kapıdaki ilk okutmanız: giriş mi çıkış mı yaptığınızı seçin.
Öğrenci kaydı aktif değilse `DEVICE_NOT_BOUND` koduyla "Öğrenci kaydı aktif değil." döner.

### 5.3 `processScan` ayrıntıları (referansla aynı — atlanmaması gerekenler)
- BLE: jeton geldiyse `bleOk` = QR diliminin, bir önceki veya bir sonraki dilimin jetonuyla `safeEqual`. `bleOk` false ise **yalnızca** `config.bleRequired` iken `BLE_MISMATCH` reddi; değilse okutma kabul edilir ve `bleVerified: false` döner.
- Çift okutma: son olaydan `duplicateWindowSeconds` geçmediyse `ACCEPTED` deneme yazılır, son olay `duplicate: true` ile döner (yön değişmez, WhatsApp mesajı yok). Bu kontrol yön kontrolünden **önce** yapılır.
- `DIRECTION_REQUIRED` hiçbir tabloya yazılmaz.
- Kabulde: `insertScanAttempt(ACCEPTED)` → dönen kayıt `REJECTED` ise (eş zamanlı tekrar) `REPLAY` döndür → `touchKiosk` → `occurredAt = new Date().toISOString()` → `createEventWithWhatsApp(...)` → `void dispatchWhatsApp(messages)`.

## 6. Ortam değişkenleri (`.env.example` → `.env.local`)

| Değişken | Açıklama |
| --- | --- |
| `DATABASE_URL` | Ortak veritabanı: `postgres://izmirfen_app:<şifre>@<sunucu>:5433/izmirfen` — **proje sahibinden özel kanaldan alınır** |
| `DATABASE_SSL` | `true` (sunucu SSL'siz bağlantıyı reddeder) |
| `DATABASE_SSL_REJECT_UNAUTHORIZED` | `false` (sertifika kendinden imzalı; bağlantı yine şifreli) |
| `DATABASE_POOL_MAX` | `3` |
| `DATABASE_OWNER_URL` | **Yalnızca proje sahibi** — ekipte boş |
| `AUTH_SECRET` | JWT anahtarı. Boşsa geliştirmede sabit anahtar kullanılır. Herkes kendisi üretir (`openssl rand -hex 32`), paylaşılmaz; canlıda zorunlu (kök AGENTS.md §5.1) |
| `BLE_REQUIRED` | `true`: BLE jetonu zorunlu (varsayılan `false`). Geliştirmede `false`, **canlıda `true`** (kök AGENTS.md §0.2, hareket inceleme yok) |
| `DUPLICATE_WINDOW_SECONDS` | Çift okutma penceresi (varsayılan 120) |
| `WHATSAPP_PROVIDER` | `mock` (geliştirmede hep `mock`; veritabanı ortak) · `insider` (gerçek gönderim) |
| `INSIDER_API_KEY` | Insider WhatsApp API anahtarı — **gizli**; yalnızca sunucudaki `.env.local` / `deploy/.env`, `NEXT_PUBLIC_` öneki yok, paylaşılmaz/commit'lenmez/loglanmaz |
| `WHATSAPP_TEMPLATE` | Insider'da onaylı şablonun adı (ör. `utility_template_deneme`) |
| `COOKIE_SECURE` | Yerelde (http) `false`, canlıda `true` |

`.env.local` değiştirildiğinde `npm run dev` yeniden başlatılmalıdır.

## 7. Komutlar

```bash
npm install
npm run db:check      # ortak veritabanına bağlantı: SSL, kullanıcı, yetkiler, tablolar, satır sayıları
npm run dev           # http://localhost:3000
npm run typecheck     # tsc --noEmit  — her işten sonra
npm run lint          # eslint        — her işten sonra (kök AGENTS.md §9: npx eslint src scripts)
npm run test:e2e      # uçtan uca test yazıldıktan sonra; npm run dev açıkken ayrı terminalde
npm run admin:create -- --email ad@okul.test --name "Ad Soyad" --role ADMIN   # şifre gizli sorulur
```
Bu projede **`db:reset` yoktur** ve olmayacaktır (ortak veritabanı).

**API'yi elle denemek (curl):**
```bash
curl -s localhost:3000/api/mobile/config
curl -s -X POST localhost:3000/api/mobile/student/check-phone -H 'Content-Type: application/json' \
  -d '{"phone":"0500 000 00 01","deviceId":"test-cihaz-adiniz-1"}'
```
Şifre içeren istekleri curl ile **komut satırına yazmayın** (kabuk geçmişine girer); bunları `npm run test:e2e` veya mobil uygulamayla deneyin.

**Örnek veri yoktur.** Panel hesabı `npm run admin:create` ile açılır; öğrenciler panelden (Öğrenciler sayfası yazılınca) veya proje sahibinin eklediği test kayıtlarıyla gelir. Başkasının kayıtlarının şifresini/cihazını değiştirmeyin; testte kendi test kayıtlarınızı oluşturun (kök AGENTS.md §7.1).

## 8. Uçtan uca test (`scripts/e2e-test.mjs`)

Referanstaki testin **aynı kontrollerini** yapar; farkı ortak veritabanında **kendi verisini oluşturup silmesidir**:
- Başta `pg` ile (aynı `DATABASE_URL`, `DATABASE_SSL*` ayarları): benzersiz son ekli (`randomBytes(4).toString("hex")`) test öğrencisi `E2E Test <ek>` (+ velisi), ikinci test öğrencisi (DEVICE_TAKEN kontrolü için), test kiosku `TEST e2e-<ek>`, `role=KIOSK` test hesabı (rastgele şifre, `bcryptjs` ile özet).
- Kiosk listesinden **kendi test kioskunu** id ile seçer (ilk kioskun değil).
- Test öğrencilerinin telefonları `+905000XXXXXX` (0500 000 XX XX) aralığından rastgele seçilir; ilk istek yerel yazımla (`0500 …`) gönderilerek biçim çevirme de denenir.
- Sonda `try/finally` ile **yalnızca kendi oluşturduklarını** siler, şu sırayla: `scan_attempts WHERE device_id = ANY($1)` → `students WHERE id = ANY($1)` (devices, attendance_events, wp_messages, student_guardians CASCADE) → `guardians WHERE id = ANY($1)` → `kiosks WHERE id = $1` → `admin_users WHERE id = $1`. `audit_logs` silinmez.
- `package.json`: `"test:e2e": "node --env-file=.env.local scripts/e2e-test.mjs"`; adres `API_URL` (varsayılan `http://localhost:3000`).

## 9. Yapılmaması gerekenler

- `repo.ts` dışında SQL; ORM eklemek; şemayı değiştirmeye çalışmak.
- Ortak veritabanında `WHERE`'siz `DELETE/UPDATE`, başkasının kayıtlarını silmek/değiştirmek, veritabanı adresini/şifresini koda veya loga yazmak.
- İstemciden gelen `direction`'a, öğrenci kimliğine veya zaman damgasına güvenmek (öğrenci kimliği JWT'den, yön sunucudan, tazelik sunucu saatinden).
- Hata yanıtında stack/SQL ayrıntısı döndürmek; şifre, anahtar, `deviceSecret`, kiosk `secret` veya JWT loglamak.
- `protocol.ts`'i mobil tarafı güncellemeden değiştirmek.
- Referanstaki yerel veritabanı dosyalarını (§2 sonu) veya referansın `AGENTS.md`/belgelerini bu projeye kopyalamak.
- Hazır dosyaları (`src/lib/db/*`, `db/schema.sql`, `scripts/db-*.mjs`, `scripts/create-admin.mjs`, `api/health`) referanstakiyle değiştirmek (repo.ts'e yalnızca §3.2'deki 4 fonksiyon eklenir).

## 10. Sık karşılaşılan hatalar

| Belirti | Neden / çözüm |
| --- | --- |
| `DATABASE_URL tanımlı değil` | `.env.local` yok veya `npm run dev` `.env.local`'i görmüyor: `cp .env.example .env.local`, adresi yazın, sunucuyu yeniden başlatın |
| `db:check` → `timeout` / `ECONNREFUSED` | Ağınız 5433'ü engelliyor olabilir (okul/şirket ağı); başka ağdan deneyin, proje sahibine haber verin |
| Tüm token'lar 401 | `AUTH_SECRET` değişti veya telefon başka geliştiricinin sunucusuna bağlandı — yeniden giriş yapın |
| Panelde girişten sonra tekrar `/login` | Yerelde `COOKIE_SECURE=false` olmalı; `fb_admin` çerezi yazılmıyor |
| `params` hatası (`params.id` undefined) | Next.js 16'da `params` Promise: `const { id } = await ctx.params` |
| `Type '"WRONG_STATE"' …` tip hatası | `REJECT_MESSAGES`'a `WRONG_STATE` eklenmiş; silin (kök AGENTS.md §0.2) |
| `permission denied for table …` | Ekip kullanıcısının yapamadığı bir şey (şema değişikliği) deneniyor — durun, proje sahibine sorun |
| `too many connections` | Açık kalmış `npm run dev` süreçlerini kapatın |
