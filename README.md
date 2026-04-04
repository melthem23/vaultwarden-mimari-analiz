
# Vaultwarden Güvenlik ve Sistem Mimarisi Analizi
**Öğrenci:** Meltem Eser  
**Öğrenci No:** 2420191036  
**Bölüm:** Bilişim Güvenliği Teknolojisi  

Bu çalışma; açık kaynaklı `Vaultwarden` projesinin kurulumundan kaynak koduna kadar olan süreçlerinin bir "Güvenlik Uzmanı ve Sistem Mimarı" gözlüğüyle incelenmesini içermektedir.

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

## 🚀 Adım 3: İş Akışları ve CI/CD Pipeline Analizi
**Hedef:** `.github/workflows` içindeki otomasyonların analizi.

Repoda yer alan `workflow` dosyalarından biri incelenmiştir. Bu dosyalar, koda yeni bir güncelleme geldiğinde otomatik olarak testlerin koşulmasını sağlar.

### ❓ Kritik Soru: Webhook Nedir?
**Webhook**, bir sistemde bir olay (event) gerçekleştiğinde, başka bir sisteme gerçek zamanlı veri göndermek için kullanılan HTTP "push" bildirimleridir. 
* **Bu Projedeki İşlevi:** Geliştirici koda yeni bir güncelleme gönderdiğinde (Push / Pull Request), GitHub sunucuları otomatik olarak bir Webhook tetikler. Bu tetikleme, CI/CD sunucularına "Hey, yeni kod geldi, hemen testleri başlat ve Docker imajını inşa et" emrini gönderir. Böylece insan müdahalesine gerek kalmadan sürekli entegrasyon sağlanmış olur.

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
