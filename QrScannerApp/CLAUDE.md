@AGENTS.md

## QrScannerApp'te Claude Code için çalışma notları

QrScannerApp'te çalışırken kökteki `../AGENTS.md` (özellikle §0 referans kod, §0.2 farklar, §4 protokol, §5 API) ve `../CLAUDE.md` (görev akışı) de geçerlidir.

### Yazmadan önce
- Bir ekranı yazmadan önce kullandığı `@/components/*`, `@/lib/*` dosyalarını okuyun. Sunucu tarafını anlamak gerekirse `../AdminPanel/src/app/api/mobile/**` dosyalarını okuyun.

### Her görevin sonunda çalıştırın ve çıktıyı raporlayın
```bash
npm run typecheck    # hatasız olmalı
npx eslint src       # uyarısız olmalı
```
Telefonda denenmesi gereken adımları kullanıcıya **numaralı ve sade** yazın (hangi derleme: Expo Go mu, `npx expo run:… --device` mı; AdminPanel'de `npm run dev` açık olmalı; hangi hesapla girileceği). Telefonda denemeyi siz yapamazsınız: kabul kriterlerinden telefonda doğrulanması gerekenleri "kullanıcı doğrulamalı" diye ayrıca listeleyin.

### Bu klasöre özel kurallar
- Görev referansta olmayan bir özellik veya var olan bir özelliğin değişmesini istiyorsa prompt esastır; yalnızca AGENTS.md başındaki "değişmeyenler"e dokunan isteklerde durup onay isteyin (kök AGENTS.md §0.3).
- Expo / React Native API'lerini hafızadan yazmayın: bu dosyanın başındaki Expo kurallarına uyun; referans kod Expo SDK 57 ile çalışır, şüphede referanstaki kullanımı örnek alın.
- Paket eklemeyin (`npx expo install` dahil) — gereken her şey kurulu. Eksik görünüyorsa durup sorun.
- Sunucuyla konuşma yalnızca `src/lib/api.ts` → `api<T>()` ile; `fetch` doğrudan ekranda kullanılmaz. Hata mesajı olarak sunucunun `e.message`'ı gösterilir.
- Öğrenci girişi **cep telefonu + şifre**dir (`student-login.tsx`: `check-phone` → şifre oluştur | şifre; gövdelerde `phone`, biçimi sunucu çevirir). Öğrenci için e-posta alanı/metni eklemeyin; "Yönetici girişi (kiosk)" e-postayla kalır.
- Gizli değerler (`deviceSecret`, token'lar, kiosk `secret`) loglanmaz, ekranda gösterilmez, AsyncStorage'a yazılmaz.
- `protocol.ts` sunucudakiyle aynı kuralları uygular; değişiklik gerekiyorsa durup sorun (iki taraf birlikte değişir).
- `react-native-ble-plx` yalnızca `src/lib/ble.ts` içinde, koşullu `require` ile yüklenir; `KioskBeacon` yalnızca `@modules/kiosk-beacon` üzerinden kullanılır.
- `ios/` ve `android/` klasörlerini oluşturmayın/düzenlemeyin (üretilir, git'e girmez). `.env` dosyasını okumayın; kullanıcıdan adresi `.env.example`'a bakarak yazmasını isteyin.
- Sunucu davranışını etkileyen bir değişiklik yaptıysanız (yapmamalısınız) AdminPanel uçtan uca testini de çalıştırın.
