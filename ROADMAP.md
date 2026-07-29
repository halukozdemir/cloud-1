# Cloud-1 Yol Haritası

> Inception'ın otomatik deployu — **Terraform** (altyapı) + **Ansible** (yapılandırma) + **Docker Compose** (servisler)
> Bulut: **AWS** `eu-north-1` · Repo: `halukozdemir/cloud-1`
> Son güncelleme: 2026-07-25 (subject baştan okunarak yeniden kurgulandı)

---

## Subject gereksinimleri → nerede karşılanıyor

Savunmada "şu şartı nasıl sağladın?" sorusuna hazır cevap tablosu.

| # | Subject şartı | Nerede | Durum |
|---|---|---|---|
| 1 | Deploy tamamen otomatik | Ansible `site.yml` | 🔄 |
| 2 | Reboot sonrası site kendiliğinden kalkar | `restart: always` + docker servisi `enabled` | 🔄 |
| 3 | Reboot sonrası veri korunur (yazılar, görseller, kullanıcılar) | named volume'lar | ⬜ |
| 4 | Birden fazla sunucuya paralel deploy | Terraform `count` + Ansible forks | ⬜ |
| 5 | Hedef varsayımı: temiz Ubuntu 22.04 + SSH + Python | Ansible agentless | ✅ |
| 6 | 1 process = 1 container, birbirleriyle konuşabilir | 4 konteyner + docker network | ⬜ |
| 7 | Dışarıdan erişim kısıtlı; DB'ye internetten bağlanılamaz | Security group + port yayınlamama | ✅ |
| 8 | Servisler: WordPress + MySQL/MariaDB + phpMyAdmin | compose | ⬜ |
| 9 | `docker-compose.yml` zorunlu | `app` rolü template'i | ⬜ |
| 10 | SQL veritabanı hem WordPress hem phpMyAdmin ile çalışmalı | MariaDB + iki istemci | ⬜ |
| 11 | Mümkünse TLS | Let's Encrypt + DuckDNS | ⬜ |
| 12 | İstenen URL'ye göre doğru siteye yönlendirme | nginx location blokları | ⬜ |
| 13 | Dışarıya sadece 80 / 443 / 22 açık | `aws_security_group.web` | ✅ |
| 14 | Ansible kodu rollere ayrılmış | `roles/docker`, `roles/app` | 🔄 |
| 15 | Taşınabilir: taze bir instance'a da kurulabilmeli | değişkenler + dinamik envanter | 🔄 |
| 16 | Resmi Docker image'ları kullanılabilir | wordpress / mariadb / phpmyadmin / nginx | ⬜ |
| 17 | Idempotent: birkaç kez çalışınca aynı sonuç | modül seçimi + `creates`/`changed_when` | 🔄 |
| 18 | Hardcoded secret YOK | Ansible Vault | ⬜ |

✅ tamam · 🔄 kısmen · ⬜ yapılacak

**Bonus yok.** Subject açıkça belirtiyor: "It does not include a bonus part."

---

## Tamamlanan fazlar

### ✅ Faz 0 — Hazırlık (2026-07-09)

- Araçlar: Terraform v1.15, Ansible core 2.21, AWS CLI v2 (Homebrew)
- AWS hesabı (ücretsiz plan, $100 kredi) + `terraform` IAM kullanıcısı (`AmazonEC2FullAccess`)
- Sıfır harcama bütçesi kuruldu ve doğrulandı
- SSH anahtarı `~/.ssh/cloud-1` (ed25519)

### ✅ Faz 1 — Terraform (2026-07-16)

- Kavramlar: provider, resource, data source, state, bağımlılık grafı, argüman vs attribute
- `main.tf`: key pair + security group (22/80/443) + instance + output
- Best-practice sertleştirmeleri (denetimden geçti): `lifecycle ignore_changes = [ami]`,
  IMDSv2 zorunlu, gp3 15 GB şifreli root disk, `cpu_credits = standard`, `default_tags`,
  SSH anahtar yolu değişkeni (`variables.tf`)
- IMDS deneyi canlı doğrulandı (401 + token dansı)

### ✅ Faz 2 — Ansible temelleri (2026-07-22)

- Kavramlar: inventory, playbook, play, task, module, idempotency, `become`
- `inventory.yml` (INI'den YAML'a çevrildi), `ping` ile ilk temas
- `docker.yml` playbook'u: apt + service + user modülleri
- Idempotency canlı görüldü: 1. koşu `changed=2`, 2. koşu `changed=0`
- Alıştırma: `practice.yml` (file, copy modülleri; `state` kavramı derinlemesine)

### ✅ Faz 3 — Roller (2026-07-25)

- `ansible-galaxy role init` ile iskelet, `roles/docker/` yapısı
- `site.yml` ana playbook'u; `changed=0` ile dönüşüm doğrulandı
- Öğrenilen: convention over configuration, `tasks/` `defaults/` `handlers/` `files/` `templates/` ayrımı

---

## Faz 4 — Boru hattını pürüzsüzleştir (kısa ama kritik)

Uygulamaya geçmeden önce günlük iş akışındaki sürtünmeleri kaldırıyoruz.
Bunlar olmadan her `destroy`/`apply` sonrası elle iş yapmak gerekiyor.

- [ ] **Dinamik envanter** — `terraform output -raw public_ip` çıktısından envanteri üret.
      IP artık hiçbir yere elle yazılmayacak (şu an `inventory.yml`'de sabit duruyor).
- [ ] **`ansible.cfg`** — `-i inventory.yml` yazmaktan kurtul; host key checking, forks ayarı
- [ ] **Swap dosyası** — t3.micro'da 1 GB RAM var, Docker build/pull sırasında yetmez.
      2 GB swap Ansible görevi olarak (`creates:` guard'ıyla idempotent)
      (DuckDNS alan adı Faz 5'in TLS adımına ertelendi — self-signed IP ile çalışıyor)

## Faz 5 — Uygulama yığını

Dört konteyner: nginx (TLS + routing) · wordpress (php-fpm) · mariadb · phpmyadmin

- [ ] **Ansible Vault** — `group_vars/all/vault.yml` şifreli; DB şifreleri, WP admin şifresi,
      DuckDNS token'ı burada. `vault-password.txt` zaten `.gitignore`'da.
- [ ] **`app` rolü** — dizin yapısı, `docker-compose.yml` template'i (Jinja2), `.env` template'i
- [ ] **MariaDB** — named volume, port yayınlanmıyor (sadece iç ağ)
- [ ] **WordPress (fpm)** — dosya volume'ü nginx ile paylaşımlı
- [ ] **phpMyAdmin** — 443 üzerinden URL alt yolundan erişim (kendi portu YOK)
- [ ] **nginx** — `/` → WordPress, `/phpmyadmin` → phpMyAdmin, 80 → 443 yönlendirmesi
- [ ] **TLS — üç aşamalı** (Haluk'un kararı 2026-07-25: self-signed ile başla)
      1. Self-signed sertifika → yığın HTTPS ile ayağa kalksın, alan adı gerekmez
      2. DuckDNS + certbot **staging** → gerçek ACME akışını bedava prova et
      3. Production sertifika → tek değişken çevirisi, en sonda, bir kez
      Gerekçe aşağıdaki "TLS kota stratejisi" bölümünde
- [ ] **Compose çalıştırma** — `community.docker.docker_compose_v2` modülü (shell değil!)

## Faz 6 — Doğrulama ve sağlamlaştırma

Her biri subject'in bir şartının kanıtı. Savunmada bunları canlı göstereceksin.

- [ ] **Idempotency testi** — playbook'u arka arkaya 2 kez çalıştır → ikinci koşu `changed=0`
- [ ] **Reboot testi** — `sudo reboot` → makine kalkınca site kendiliğinden ayakta,
      yazılar/görseller/kullanıcılar duruyor
- [ ] **Taze instance testi** — `terraform destroy` + `apply` + tek komutla tam deploy.
      Subject'in "portable, sadece geliştirdiğin makinede değil" şartı budur.
- [ ] **Port taraması** — dışarıdan yalnızca 22/80/443 açık olduğunu doğrula (`nmap` ya da AWS konsolu)
- [ ] **Secret taraması** — repoda ve git geçmişinde düz şifre olmadığını doğrula
- [ ] **Güvenlik ekstraları (opsiyonel)** — fail2ban, unattended-upgrades

## Faz 7 — Çoklu sunucu

- [ ] Terraform `count` ile 2 instance
- [ ] Envanterde iki host; Ansible paralel deploy (forks)
- [ ] Her sunucu için ayrı DuckDNS subdomain'i (ya da alternatif strateji)
- [ ] İki siteyi de tarayıcıda doğrula

## Faz 8 — Savunma hazırlığı

- [ ] **Uçtan uca prova** — sıfırdan (destroy → apply → deploy → tarayıcı) süreyi ölç
- [ ] **README.md** — kurulum adımları, mimari, tasarım kararları
- [ ] **AWS konsol erişimi** — ⚠️ Subject şartı: savunmada öğrenci bulut hesabına
      kendi e-postasıyla / root hesabıyla giriş yapabilmeli. Aksi halde kopya sayılıp
      savunma durduruluyor. Şifreyi/2FA'yı hazır tut.
- [ ] **Soru-cevap provası** — aşağıdaki listeden

### Savunma soruları (hazır cevabı olması gerekenler)

- Neden Ansible? (agentless — subject'in "sadece SSH + Python" şartıyla birebir)
- Idempotency nedir, nasıl sağladın, nasıl ispatlarsın?
- `shell`/`command` neden idempotent değil, ne zaman kullandın ve nasıl korudun?
- Roller neden var, `defaults` ile `vars` farkı?
- Secret'lar nerede, git'e neden sızmıyor?
- Neden resmi image'lar? Inception'dakileri neden kullanamıyorsun?
- Veritabanına internetten neden erişilemiyor? İki katman göster (SG + port yayınlamama)
- Reboot'ta ne oluyor, hangi mekanizma siteyi ayağa kaldırıyor?
- TLS nasıl çalışıyor, sertifikayı kim imzaladı?
- URL'ye göre yönlendirme nasıl oluyor?
- Terraform state nedir, neden git'te değil?
- IMDSv2 neden zorunlu yaptın? (SSRF, confused deputy)
- Aynı anda 2 sunucuya nasıl deploy ediyorsun?

---

## 🔐 TLS kota stratejisi (araştırma 2026-07-25, kaynak: letsencrypt.org/docs/rate-limits)

**Bağlayıcı limit:** Aynı alan adı için **7 günde 5 sertifika**. Doluysa saatte
1 yenisi değil — her 34 saatte 1 hak geri geliyor. İtiraz/muafiyet yok.
Bizim `destroy`/`apply` döngümüzde her rebuild yeni sertifika demek → bir akşam
hata ayıklarken kota bitebilir ve savunma haftasına denk gelirse HTTPS yok.

**İyi haber:** `duckdns.org` Public Suffix List'te → her subdomain'in kotası ayrı.

**Kurallar:**

1. **İki DuckDNS adı al:** biri geliştirme, biri savunma/gerçek. Geliştirme adını
   yakarsan gerçek olan temiz kalır. (Ücretsiz hesapta 5 subdomain hakkı var.)
2. **Geliştirme boyunca STAGING kullan** — `https://acme-staging-v02.api.letsencrypt.org/directory`.
   Limitleri pratikte sınırsız (haftada 30.000). Tarayıcı uyarı verir, önemli değil.
   Ansible değişkeni: `letsencrypt_staging: true` (varsayılan), sonunda `false`.
3. **Gerçek sertifikayı en sona bırak** — her şey çalışınca, bir kez.
4. **Sertifika alındıktan sonra makineyi yok etme** ya da `/etc/letsencrypt` klasörünü
   yedekle. (Sertifikalar rebuild'de kaybolursa yeniden istemek kotadan düşer;
   ARI muafiyeti burada işe yaramıyor — eldeki sertifikayı gerektiriyor.)
5. **Başarısız doğrulama ayrı limit:** saatte 5 deneme (12 dakikada 1 yenilenir).
   Yani yanlış yapılandırmayla üst üste denemek de seni kilitler. Önce
   `--dry-run` ile prova et.

**Challenge yöntemi:** HTTP-01 (webroot) — 80 portu zaten açık olmak zorunda,
nginx `/.well-known/acme-challenge/` yolunu servis eder. Alternatif DNS-01:
DuckDNS TXT kaydı destekliyor ve A kaydının güncel olmasını beklemez; A kaydı
yayılma sorunu yaşarsak buna geçeriz.

## ⚠️ Idempotency tuzakları (araştırma 2026-07-17, kaynak: Ansible resmi dokümanı)

Modüller idempotent, **playbook'lar değil**. Resmi doküman: "not all playbooks and
not all modules behave this way" (playbooks_intro → Desired state and idempotency).
Bizim projede patlayacak yerler:

- `shell`/`command` ile `docker compose up`, `openssl`, `wp-cli` → her koşuda `changed`
  - Çözüm: `creates:` / `removes:` guard, ya da `register` + `changed_when:`
  - Okuma probu ise: `changed_when: false` + `failed_when: false` + `when:` ile gate
- **En tehlikelisi:** her koşuda yeni şifre üretmek (`openssl rand`, `lookup('password','/dev/null')`)
  → `.env` değişir → konteynerler yeni şifreyle kalkar ama MariaDB volume'ünde ESKİ şifre durur
  → WordPress "database connection error". Çözüm: Vault'ta sabit şifre.
- `lineinfile`/`blockinfile` yanlış kullanımı, `git force`, `unarchive`
- Test: playbook 2 kez → ikinci koşu `changed=0`

## Bilinçli ertelenen/reddedilen kararlar (savunmada cevabı hazır)

| Karar | Gerekçe |
|---|---|
| S3 remote state yok | Tek kişilik proje; local state yeterli, karmaşıklık eklemez |
| Elastic IP yok | Saatlik ücretli; IP'yi hiçbir yere elle yazmadığımız için gerekmiyor |
| Standalone SG rule resource'ları yok | Inline bloklar bu ölçekte yeterli; v6 idiyomu Faz 7 refactor'ında |
| Docker resmi deposu yerine Ubuntu deposu | Sadelik; 3 görev vs 6 görev |
| Redis/FTP/Portainer yok | Subject'in çekirdek şartı değil; kapsamı dar tutuyoruz |

## Maliyet güvenliği

- **Gün sonunda `terraform destroy`** — makine açık kalırsa saatlik kredi yer
- Sıfır harcama bütçesi kurulu; Billing → Credits'ten $100 bakiyeyi ara sıra kontrol et
- t3.micro'dan büyüğüne çıkma; disk 15 GB gp3
- Makine şu an: **çalışıyor** (`i-0148334ee034aea76`) — iş bitince yok et
