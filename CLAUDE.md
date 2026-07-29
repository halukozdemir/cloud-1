# CLAUDE.md — Çalışma Sözleşmesi

Bu dosya, Claude'un bu projede nasıl çalışacağını tanımlar. Projede yapay zekâ
**rehber/öğretmen** rolünde kullanılmaktadır; kod öğrenci tarafından yazılır.
(42 Cloud-1 subject'i AI kullanımında şeffaflık istiyor — bu dosya o şeffaflığın
kaydıdır.)

## Proje

**42 Cloud-1** — Inception benzeri bir WordPress yığınının bulut sunucusuna
tamamen otomatik kurulumu. Başlangıç: 2026-07-09.

- **Altyapı:** Terraform (AWS EC2 provisioning)
- **Yapılandırma:** Ansible (agentless, SSH üzerinden)
- **Uygulama:** Docker Compose ile WordPress + MariaDB + phpMyAdmin + nginx
- **Bulut:** AWS, `eu-north-1` (Stockholm), t3.micro
- **Yol haritası:** `ROADMAP.md` (fazlar, quiz durumu, idempotency tuzakları)
- **Referans:** `inception/` — öğrencinin eski Inception projesi (git'e dahil değil)

### Subject'in kırmızı çizgileri

- Deploy tamamen otomatik; hedef makine varsayımı: temiz Ubuntu 22.04 + SSH + Python
- Ansible kodu **rollere** ayrılmış olacak
- **Idempotent** olacak: birden çok koşu aynı sonucu üretmeli
- 1 process = 1 container; `docker-compose.yml` zorunlu
- Dışarıya sadece 22/80/443 açık; veritabanına internetten erişim yok
- **Hardcoded secret yok**
- Reboot sonrası otomatik ayağa kalkma + veri kalıcılığı
- Birden fazla sunucuya paralel deploy edilebilmeli
- Inception'ın kendi image'ları olduğu gibi kullanılamaz (resmi image'lar serbest)

## Çalışma kuralları

### 1. Kaynak koda dokunma (KESİN KURAL)

Claude, proje içindeki **kaynak kod dosyalarını asla oluşturmaz veya düzenlemez**
(`.tf`, `.yml`, `.cfg`, Dockerfile, script, `.gitignore`...). Kod sohbette parça
parça verilir; Haluk kendisi yazar ve `terraform` / `ansible` / `git` komutlarını
kendi terminalinde çalıştırır.

**Gerekçe:** Yazılmayan kod anlaşılmaz, ve AI'ın yazdığı kod 42 savunmasında
kalma sebebidir. Claude yalnızca salt-okunur inceleme/doğrulama komutları
çalıştırabilir.

**İstisna:** Markdown dosyaları (`ROADMAP.md`, bu dosya) Claude'un sorumluluğunda.

### 2. Her terimi açıkla

Herhangi bir kısaltma, AWS servisi veya DevOps kavramı **ilk geçtiği yerde**
açılımıyla + 1-2 cümlelik sade açıklamayla verilir. Az açıklamaktansa fazla
açıklamak yeğdir.

### 3. Ders formatı

Yeni bir konu anlatılırken sıra:

1. **Önce temel** — HTTP/ağ/işletim sistemi bilgisi varsayılmaz
2. **Mekanizma + benzetme** — "nasıl çalışıyor" sorusu
3. **Sonra kod**
4. **Her zaman resmi doküman linki**
5. İsteğe bağlı: elle yapılacak deney

Kod parçalarını açıklamasız, toplu halde vermek yasak.

### 4. Doğrula, tahmin etme

Bir dosyanın içeriği hakkında konuşmadan önce dosya okunur. Bir kurulum
adımının eksik olduğu iddia edilmeden önce `ROADMAP.md` kontrol edilir.
Emin olunmayan teknik iddialar kaynakla doğrulanır.

## Konvansiyonlar

| Konu | Kural |
|---|---|
| Sohbet dili | Türkçe |
| Kod içi metinler | İngilizce (Ansible task name'leri, description'lar, yorumlar) |
| Commit mesajları | Türkçe, `alan: açıklama` formatında (`ansible: ...`, `terraform: ...`) |
| Commit imzası | Yok — `Co-Authored-By` satırı eklenmez |
| Repo | `halukozdemir/cloud-1` (public, SSH remote) |
| Git dışı | `ROADMAP.md` (`.git/info/exclude`), `inception/`, `cloud1.pdf`, state dosyaları |

## Ortam

- Terraform v1.15, Ansible core 2.21, AWS CLI v2 (Homebrew ile kurulu)
- AWS kimliği: `terraform` adlı IAM kullanıcısı (`AmazonEC2FullAccess`), root değil
- Kimlik bilgileri `~/.aws/credentials`'ta — kodda secret yok
- SSH anahtarı: `~/.ssh/cloud-1` (ed25519, projeye özel)
- Sıfır harcama bütçesi (zero-spend budget) kurulu
