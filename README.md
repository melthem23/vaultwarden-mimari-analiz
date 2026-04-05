
# Vaultwarden Güvenlik ve Sistem Mimarisi Analizi
---

## 🛠️ Adım 1: Kurulum ve install.sh Analizi
**Hedef:** Kurulum esnasındaki güvenlik önlemleri ve dış paket çekim mekanizması.

### 🔍 Kritik Soru Cevabı: Paket Güvenliği
Vaultwarden projesinde doğrudan terminale kopyalanıp çalıştırılan güvensiz `curl | bash` mekanizmaları (install.sh) yerine modern paket yöneticileri tercih edilmiştir. 
* Proje ana dizinindeki `Cargo.lock` dosyası, kullanılan tüm dış kütüphanelerin **SHA-256 hash imzalarını** kilit altında tutar. 
* Yazılım derlenirken bu imzalar otomatik doğrulanır. Dolayısıyla körü körüne bir paket çekimi söz konusu değildir; kaynak bütünlüğü sıkı sıkıya korunmaktadır.

### 📁 Dizinler ve Yetkiler
* **Dizin:** Yazılım çalıştırıldığında tüm verileri `/data` (veya Docker içinde `/vw-data/`) klasöründe depolar.
* **Yetki:** Sistemde `root` (en yetkili) erişimini zorunlu kılmaz. Bu durum, olası bir siber saldırıda hacker'ın tüm ana makineyi ele geçirmesini engeller.

---

## 🧹 Adım 2: İzolasyon ve İz Bırakmadan Temizlik
**Hedef:** Kurulan aracın sistemden hiçbir kalıntı kalmayacak şekilde silinmesi ve bunun ispatı.

Vaultwarden sanal ortamda (VM) bir Docker konteyneri olarak ayağa kaldırılmıştır. İz bırakmadan temizlik şu adımlarla gerçekleştirilmiştir:

1. **Konteyner ve İmaj Silme:** `docker stop vaultwarden` ve `docker rm vaultwarden` komutlarıyla çalışan canlı sistem durdurulmuş ve silinmiştir.
2. **Kalıntı Kontrolü (İspat):**
   * **Port Kontrolü:** `netstat -tulnp | grep 80` komutuyla uygulamanın arkada bir dinleme noktası bırakıp bırakmadığı kontrol edilmiştir.
   * **Dosya Kontrolü:** `find / -name "*vaultwarden*"` komutuyla diskte başıboş log veya kalıntı dosyası olmadığı ispatlanmıştır.

---

## 🚀 Adım 3:## 🚀 Adım 3: İş Akışları ve CI/CD Pipeline Analizi
**Hedef:** Repoda bulunan CI/CD paketlerinden birinin seçilip derinlemesine incelenmesi ve Webhook kavramının açıklanması.

Vaultwarden reposunda yer alan `.github/workflows/build.yml` dosyası analiz edilmiştir. Bu dosya, projede Sürekli Entegrasyon (CI) ve Sürekli Dağıtım (CD) süreçlerini yöneten otomasyon şablonudur.

### ⚙️ 1. CI/CD Adım Adım Ne Yapıyor?
Geliştiriciler ana koda her `push` yaptığında veya yeni bir `Pull Request` açıldığında GitHub Actions sunucuları devreye girer ve şu adımları sırasıyla koşturur:
1. **Ortam Hazırlığı:** Ubuntu tabanlı sanal bir makine ayağa kaldırılır ve Rust derleyicisi (`rustup`) kurulur.
2. **Statik Analiz (Linter):** `cargo clippy` komutu çalıştırılarak kodda güvenlik zayıflığı veya yazım hatası olup olmadığı taranır.
3. **Derleme ve Paketleme:** Kod hatasızsa derlenir ve Docker Hub platformuna yüklenmek üzere yeni bir Docker imajı haline getirilir.

### ❓ 2. Kritik Soru: Webhook Nedir ve Ne İşe Yarar?
**Webhook**, bir sistemde belirli bir olay (event) gerçekleştiğinde, başka bir sisteme gerçek zamanlı veri göndermek için kullanılan HTTP POST bildirimleridir. Klasik API'lerin aksine, sürekli "Yeni bir şey var mı?" diye sormak (polling) yerine, olay gerçekleştiği an karşı tarafı "dürterek" haber verir.

**Vaultwarden Özelinde İşlevi:**
Geliştirici GitHub'a yeni kod yüklediğinde, GitHub bunu bir "Push Olayı" olarak algılar ve tanımlı olan Webhook'u tetikler. Bu tetikleme sinyali, CI/CD sunucularına veya Docker Hub'a *"Hey, yeni kod onaylandı, hemen yeni Docker imajını inşa et ve sunucuyu güncelle"* emrini otomatik olarak iletir. Bu sayede insan müdahalesine gerek kalmadan sıfır kesintiyle sistem güncellenmiş olur.

---

## 🐳 Adım 4: Docker Mimarisi ve Konteyner Güvenliği
**Hedef:** Docker mimarisi ve güvenli yapılandırma.

Vaultwarden projesinde yazılımın kendisi ve bağımlılıkları Docker katmanları (`layers`) halinde inşa edilir.

### ❓ Kritik Soru Cevapları:
* **Docker İmajı Nedir?:** Bir uygulamanın çalışması için gereken kodların, kütüphanelerin ve ortam değişkenlerinin paketlenmiş, salt-okunur (read-only) halidir.
* **Katmanlar:** Dockerfile'daki her bir komut (`RUN`, `COPY` vb.) yeni bir katman oluşturur. Bu sayede sadece değişen katmanlar güncellenerek hız kazanılır.
* **Güvenlik Nasıl Sağlanır?:** Konteynerin sistemde kısıtlı erişime sahip olması için imajın `root` olmayan bir kullanıcı (`non-root user`) ile çalıştırılması ve sadece gerekli portların dışarıya açılması gerekmektedir.

---

## 🕵️‍♂️ Adım 5: Kaynak Kod ve Akış Analizi
**Hedef:** Başlangıç noktası, şifreleme ve tehdit modellemesi.

### 🚀 Uygulama Giriş Noktası (Entrypoint)
Vaultwarden bir Rust projesidir. Uygulamanın başlangıç noktası `src/main.rs` dosyasıdır. Web istekleri ise Rocket framework'ü üzerinden yönetilir.

### 🔐 Kimlik Doğrulama (Authentication)
Proje, kullanıcı şifrelerini sunucuya asla "düz metin" (cleartext) olarak göndermez. İstemci tarafında (tarayıcıda) PBKDF2 veya Argon2 şifreleme algoritmalarından geçirilerek sunucuya ulaştırılır.

### ❓ Kritik Soru: Hacker Ne Tür Verileri Çalabilir ve Nasıl Saldırır?
Bir siber saldırgan kaynak kodunu incelediğinde, veri tabanına sızsa bile kullanıcı şifrelerinin hash'lenmiş olduğunu görür. Ancak şunları hedefleyebilir:
* **Brute-Force (Kaba Kuvvet Saldırısı):** Giriş yapma fonksiyonunu bularak zayıf şifreli hesapları kırmaya çalışabilir. Buna karşı Vaultwarden yönetim panelinde rate-limiting (istek sınırlama) aktifleştirilmelidir.
* **Sızıntı Taraması:** Hacker kaynak kodunda veya çevre değişkenlerinde (`.env`) unutulmuş API anahtarları veya veritabanı şifreleri (secrets) arayabilir. Bu yüzden kod hiçbir zaman açık kimlik bilgisi içermemelidir.
