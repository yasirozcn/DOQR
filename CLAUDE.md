@AGENTS.md

## Claude Code için ek notlar

Bu projede çalışan ekip üyeleri yazılımda deneyimli olmayabilir. Kullanıcı genellikle bir Jira görevini (ör. `WEB-3`, `APP-2`) yapıştırıp "bunu yap" diyecektir.
Sizin işiniz, görevi **referans projeyle aynı yapıda** (görev/prompt referanstan farklı bir şey istiyorsa prompt'a göre — kök AGENTS.md §0.3), küçük ve doğrulanmış adımlarla tamamlamak ve kullanıcıya **ne yaptığınızı sade Türkçeyle** anlatmaktır.

### Her görevde izlenecek akış (sırayla)

1. **Bağlamı okuyun:** kök `AGENTS.md`, ilgili alt projenin `AGENTS.md` + `CLAUDE.md` (AdminPanel veya QrScannerApp). Görev metnindeki "Referans dosyalar" listesini çıkarın.
2. **İsteği sınıflandırın:** (a) referanstaki bir özelliği kurmak, (b) referansta **olmayan** yeni bir özellik, (c) var olan bir özelliği **değiştirmek**. (b) ve (c)'de **prompt esastır** (kök AGENTS.md §0.3); istek Bluetooth yapısına, panel ↔ uygulama bağlantısına (API sözleşmesi), protokole, güvenlik çekirdeğine veya şemaya dokunuyorsa durun, etkisini açıklayın ve onay isteyin.
3. **İlgili dosyaları okuyun:** görevdeki dosyaları ve içe aktardıkları dosyaları okuyun. Yeni özellikte (b) bu depodaki en yakın benzer dosyayı yapı/stil örneği olarak alın. Bu depoda zaten var olan ilgili dosyaları da okuyun (özellikle `src/lib/db/repo.ts`, `types.ts`).
4. **Plan yazın (kısa):** hangi dosyaları oluşturacağınızı/değiştireceğinizi, referanstan hangi uyarlamaları (§0.2) ve prompt nedeniyle hangi bilinçli farkları (§0.3) yapacağınızı 3–8 maddeyle kullanıcıya söyleyin. Görev kapsamı dışına çıkmayın.
5. **Uygulayın:** referanstaki yapıyı, adları, Türkçe metinleri ve hata kodlarını kullanın; §0.2'deki farkları ve prompt'un açıkça istediği değişiklikleri uygulayın. Yeni özellik eklediyseniz ilgili AGENTS.md hedef ağacını/tablolarını da güncelleyin. Hazır dosyaları (`src/lib/db/*`, `db/schema.sql`, `scripts/db-*`, `infra/`) görev açıkça istemedikçe değiştirmeyin.
6. **Doğrulayın:** kök AGENTS.md §9'daki komutları çalıştırın (tip kontrolü + lint; sunucu davranışı değiştiyse uçtan uca test). Görevdeki **her kabul kriterini** nasıl doğruladığınızı tek tek yazın (komut + beklenen/gerçek çıktı). Doğrulayamadığınız kriteri "doğrulanmadı — neden" diye açıkça belirtin; asla "çalışıyor olmalı" demeyin.
7. **Belgelerle karşılaştırın:** yazdığınız dosyayı AGENTS.md'lerdeki tanımlarla (adlar, metinler, hata kodları) karşılaştırın; kalan her farkın prompt'un isteğinde (§0.3) yer aldığını kontrol edin. Açıklanamayan fark varsa düzeltin.
8. **Özetleyin:** değişen dosyalar, nasıl test edildiği, kullanıcının elle yapması gerekenler (ör. telefonda deneme). Commit/push yalnızca kullanıcı isterse; commit mesajı Türkçe.

### Kullanıcıyla iletişim

- Kullanıcının teknik terimleri bilmediğini varsayın: komut önerirken **ne işe yaradığını** bir cümleyle söyleyin; hata çıktısını sade dille açıklayın.
- Kullanıcının çalıştırması gereken komutları tam ve kopyalanabilir verin (hangi klasörde çalıştırılacağı dahil). Etkileşimli komutlar için `! <komut>` biçimini önerin.
- Görev belirsizse (referansla aynı mı kalsın, prompt'a göre mi değişsin?) veya referans ile AGENTS.md çelişiyorsa tahmin etmeyin; durup sorun.
- Bir görev başka bir görevin bitmesini gerektiriyorsa (Jira "Bağımlılık") ve o iş yapılmamışsa, bunu söyleyin; eksik işi kendiliğinden yapmayın.

### Kurallar

- **Varsayılan hedef, referans projeyle aynı yapıdır** (kök AGENTS.md §0). Dosya, klasör, fonksiyon, route ve ekran adlarını hedef ağaçlardan ve referans koddan alın. **Prompt referansta olmayan bir özellik veya var olan bir özelliğin değişmesini isterse prompt baz alınır** (§0.3); yeni dosyalar aynı katman yapısına ve adlandırma kurallarına uyar.
- **Değişmeyen ana yapılar** (prompt istese bile onaysız değiştirilmez, §0.3): Bluetooth yapısı, panel ↔ uygulama bağlantısı (`/api/mobile/*` sözleşmesi, JWT, hata biçimi), QR/imza protokolü, cihaz bağlama ve okutma doğrulama sırası, veri erişim katmanı ve şema.
- **Öğrenci girişi e-posta değil cep telefonu + şifredir** (proje sahibinin kararı, kök AGENTS.md §0.2): `students.phone` (`+905XXXXXXXXX`), `POST /api/mobile/student/check-phone`, gövdelerde `phone`. Referanstaki e-posta akışını (`check-email`, `email` alanı, "Geçerli bir e-posta girin.") öğrenci için geri getirmeyin. Panel ve kiosk tableti girişi e-postayla kalır. Jira görevinde öğrenci için "e-posta" geçiyorsa telefon numarası olarak uygulayın.
- **Referans depodaki belgeler bu proje için geçerli değildir** (yerel Docker, `db:reset` vb.). Referanstan yalnızca kaynak kod alınır; kurallar bu depodaki AGENTS.md/CLAUDE.md'dir.
- **Dosya yapısı** (ayrıntı: kök AGENTS.md §3, alt projelerin AGENTS.md §2):
  ```
  AdminPanel/   db/schema.sql · scripts/ · src/lib/{db/,config,auth,session,api,protocol,scan,whatsapp}.ts · src/components/ · src/app/{login,(panel),api/mobile}/
  QrScannerApp/ src/app/(ekranlar) · src/components/{ui,icons,BluetoothGate}.tsx · src/lib/{config,api,session,device,protocol,ble} · modules/kiosk-beacon/ · tools/
  infra/dev-db/ ortak geliştirme veritabanı (dokunmayın — proje sahibi)
  ```
- **Gereksiz kod yok, okunabilir kod** (kök AGENTS.md §8.1): yalnızca istenen değişikliği yapın; spekülatif soyutlama, kullanılmayan kod, yorum satırına alınmış kod, tek satırlık sarmalayıcı, gereksiz bağımlılık eklemeyin. Var olan yardımcıları (`repo.ts`, `api.ts`, `ui.tsx`) kullanın.
- **`AUTH_SECRET`**: kök AGENTS.md §5.1'e uyun: her geliştirici kendi anahtarını üretir (`openssl rand -hex 32`), anahtar paylaşılmaz/commit'lenmez; üretimde `AUTH_SECRET` yoksa `auth.ts` hata fırlatır. Kullanıcıdan başkasının anahtarını istemeyin veya bir anahtarı koda yazmayın.
- Bir alt projede çalışırken o klasördeki `CLAUDE.md` / `AGENTS.md` da yüklenir; ikisini birlikte uygulayın. Çelişki olursa **alt projedeki** kural geçerlidir, ama protokol ve API sözleşmesi (kök AGENTS.md §4–§6) yalnızca iki taraf birlikte değiştirilerek değişir.
- Kütüphane sürümleri eğitim verinizden yenidir (Next.js 16, Expo SDK 57, React 19). API'yi hafızadan yazmayın: Next.js için `AdminPanel/node_modules/next/dist/docs/`, Expo için https://docs.expo.dev/llms.txt. Referans kod bu sürümlerle çalışır; şüphede referanstaki kullanımı örnek alın.
- Gizli bilgi içeren dosyaları (`.env*`, `infra/dev-db/.env`) okumayın/göstermeyin; commit'e eklemeyin. Şifreyi veya `DATABASE_URL`'yi komut satırına, koda, loga yazan bir komut önermeyin.
- Veritabanı **ortaktır** (AWS, tüm ekip). Veri silen/değiştiren SQL'i (`DELETE`, `UPDATE`, toplu işlemler) çalıştırmadan önce kullanıcıya etkilenecek satırları gösterip onay alın; `WHERE`'siz `DELETE/UPDATE` asla. Testler kendi oluşturduğu kayıtları kullanır ve sonunda siler (kök AGENTS.md §7.1).
- Şema değişikliğini veritabanında doğrudan denemeyin (yetkiniz de yok): `db/schema.sql` + `types.ts` + `repo.ts` değişikliğini PR olarak hazırlayın ve proje sahibinin uygulaması gerektiğini belirtin. **Bu projenin görevleri şema değişikliği gerektirmez**; gerektiğini düşünüyorsanız durup sorun.
- Kullanıcının çalışan sunucusunu (`npm run dev`) durdurmadan/yeniden başlatmadan önce sorun.
- Commit ve push yalnızca istendiğinde.
