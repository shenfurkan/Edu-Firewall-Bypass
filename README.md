# Okul/Kurum Ağı Kısıtlamalarını Aşma Rehberi
## AWS Üzerinde Self-Hosted Amnezia + Xray Kurulumu

> ⚠️ **Yasal Uyarı:** Bu rehber yalnızca **akademik araştırma, ağ güvenliği analizi ve eğitim** amacıyla hazırlanmıştır. Anlatılan yöntemlerin gerçek ortamlarda uygulanması, bulunduğunuz ülkenin yasalarıyla veya kurumunuzun kullanım politikalarıyla çelişebilir. Doğabilecek her türlü hukuki ve teknik sorumluluk kullanıcıya aittir. Bu depo, sistem açıklarının kötüye kullanılmasını teşvik etmemektedir.

---

## Neden Standart VPN'ler Çalışmaz?

Okul ve kütüphane ağlarındaki sansür iki katmanda çalışır:

| Katman | Nasıl Çalışır? | Standart VPN'in Sorunu |
|---|---|---|
| **1. IP Engelleme** | Bağlanmak istediğin adres kara listede mi diye kontrol eder | Ticari VPN'lerin IP aralıkları zaten listelerde |
| **2. Paket İnceleme (DPI)** | Veri paketinin içine bakarak onu tanımlar | WireGuard, OpenVPN'in kendine özgü imzaları var, kolayca tespitedilir |

### Çalışmayanlar

Piyasadaki popüler çözümlerin neden bu tür ağlarda işe yaramadığını bizzat test ettim:

| Servis / Araç | Neden Çalışmıyor? |
|---|---|
| **Windscribe** | IP aralıkları ve VPN protokol imzaları kurumsal kara listelerde |
| **ProtonVPN** | Aynı sebep; WireGuard/OpenVPN imzaları DPI tarafından anında tespit ediliyor |
| **Mullvad** | IP blokları bilinen ticari VPN havuzlarından geliyor, ayrıca obfuscation varsayılan olarak kapalı |
| **ExpressVPN** | Çok yaygın olduğu için IP aralıkları neredeyse her kurumsal filtre listesinde mevcut |
| **Tor** | Tor düğümlerinin (relay) IP listesi kamuya açık; ağ, bu adresleri doğrudan engelliyor. Bridge kullanımı kısmen çalışsa da okul ağlarında çoğunlukla o da engelli |

**Ortak sorun:** Hepsinin ya bilinen IP blokları vardır ya da trafiğinin kendine özgü bir "imzası" (parmak izi) vardır. Ağ yöneticileri bu imzaları ve IP listelerini satın alarak filtrelerine ekler.

Bu rehberdeki mimari her iki katmanı da geçersiz kılar:

- **IP sorununu** → AWS (Amazon) sunucusu çözer. Kurumlar, meşru internet trafiğinin büyük kısmı AWS'den geçtiği için bu IP bloklarını tamamen engelleyemez.
- **DPI sorununu** → Xray protokolü çözer. Tüm VPN trafiğini sıradan bir HTTPS (web sitesi) trafiği gibi gösterir.
- **Kurulum karmaşıklığını** → Amnezia istemcisi çözer. Tüm teknik süreci otomatikleştirir.

### Neden Amnezia'nın Kendi Sunucuları Değil?

Amnezia, kendi hazır sunucularını da sunar; ancak ücretsiz versiyonun **8 Mbps** hız sınırı vardır ve ücretli seçenekler gerektirir. AWS Free Tier (ücretsiz katman) ile kendi sunucunu kurduğunda:

- Hız sınırı olmaz
- Altyapı üzerinde tam kontrol sağlarsın
- Ek maliyet oluşmaz

> **Not:** Bu sistem seni **tamamen anonim yapmaz.** AWS, sunucuna sabit (statik) bir IP atar. Amaç anonimlik değil; ağ denetimini kriptografik olarak atlatarak kısıtlı içeriklere ve torrent gibi protokollere erişmektir.

---

## Bölüm 1: AWS Sunucusu Kurulumu

### 1.1 — EC2 Sanal Sunucusu Oluşturma

Amazon Web Services (AWS), bir yıllık ücretsiz (Free Tier) sanal sunucu hakkı verir. İlk olarak bu sunucuyu (VPS) kiralayıp yapılandırmak gerekir:

1. **Hesap Oluşturma:** Eğer hesabınız yoksa, öncelikle [AWS](https://aws.amazon.com/tr/free/) üzerinden bir hesap açılır. Kayıt olurken kimlik doğrulaması için bir kredi/banka kartı girmeniz gerekir (ücretsiz sınırlar içinde kaldığınız sürece para çekmez).
2. **AWS Paneline Giriş:** Hesabınızı oluşturduktan veya giriş yaptıktan sonra [AWS Yönetim Paneli (Console)](https://aws.amazon.com/console/)'ne gidilir.
3. **Bölge (Region) Seçimi:** Ekranın sağ üst köşesinde bir şehir ismi (örneğin *Frankfurt*, *Stockholm*, *N. Virginia*) yazar. Buraya tıklanarak size veya bulunduğunuz yere en yakın olan (en düşük gecikme/ping süresi için) bir bölge seçilir (Örn: *Frankfurt (eu-central-1)*).
4. **EC2 Hizmetini Bulma:** Sol üstteki arama çubuğuna tıklanır, **EC2** yazılır ve arama sonuçlarında çıkan **"EC2"** (Virtual Servers in the Cloud) hizmetine tıklanarak açılır.
5. **Sunucu Başlatma:** Açılan EC2 Dashboard (Kontrol Paneli) ekranında turuncu renkli **"Launch instance"** (Sunucu Başlat) butonuna tıklanır.
6. **Sunucuya İsim Verme:** "Name and tags" bölümündeki **Name** boşluğuna sunucunuza vermek istediğiniz bir isim yazılır (Örneğin: `vpn-sunucum`).
7. **İşletim Sistemini Seçme:** "Application and OS Images (Amazon Machine Image)" başlığı altında bir sürü işletim sistemi logosu göreceksiniz. Buradan **Ubuntu** logosuna tıklanır.
    - Altında açılan "Amazon Machine Image (AMI)" listesinden **Ubuntu Server 24.04 LTS** veya **Ubuntu Server 22.04 LTS** seçilir (Yanında *Free tier eligible* - *Ücretsiz kullanıma uygun* yazdığından emin olunur).
8. **Sunucu Gücünü Seçme:** "Instance type" başlığı altındaki menüden **`t2.micro`** veya **`t3.micro`** seçilir. Hangisinin yanında *(Free tier eligible)* yazıyorsa onu seçmelisiniz. Bu donanım, tünelleme şifrelemesi için fazlasıyla yeterli işlem gücünü sunar.

### 1.2 — SSH Anahtarı Oluşturma

Sunucuya **kriptografik anahtar** ile bağlanılır. Zaten bunu sunucuyu oluştururken yaptırdığı için extradan bir şey yapmanıza gerek yok.

1. **"Create new key pair"** seçilir.
2. Anahtar oluşturulur ve bilgisayara `.pem` uzantılı bir dosya olarak indirilir.
3. Bu dosya **silinmemeli ve kaybedilmemelidir** — Amnezia'nın sunucuya bağlanabilmesinin tek yolu budur.

### 1.3 — Güvenlik Duvarı (Security Group) Kuralları

Maskelenmiş VPN trafiğinin sunucuya ulaşabilmesi için doğru portların açık olması gerekir.

1. Kurulum ekranında **"Network settings → Edit"** bölümüne tıklanır.
2. Aşağıdaki **Inbound (gelen trafik) kuralları** listeye eklenir:

| Kural Tipi | Protokol | Port | Kaynak | Amaç |
|---|---|---|---|---|
| SSH | TCP | `22` | Anywhere |Bunu varsayılan bırakın değiştirmeniz gerekmez|
| Custom TCP | TCP | `443` | Anywhere | HTTPS görünümlü tünel trafiği |
| Custom UDP | UDP | `443` | Anywhere | HTTPS görünümlü tünel trafiği |

> **Neden 443?** Bu port, tüm dünyada standart web sitesi (HTTPS) trafiği için kullanılır. Ağ denetleyicileri bu porta gelen veriyi engelleyemez, çünkü engellerlerse normal internet de çalışmaz. Xray, VPN verisini bu porta yönlendirerek görünmez kılar.

3. Ayarlar kaydedilir ve **"Launch Instance"** ile sunucu başlatılır.

### 1.4 — Sunucuyu Güncelleme

Sunucu çalışmaya başladıktan sonra sistemin güncellenmesi sağlıklı olacaktır.

1. AWS panelinde ilgili sunucu seçilir.
2. **"Connect"** butonuna tıklanır → **"EC2 Instance Connect"** sekmesinde tekrar **"Connect"** butonuna basılır.
3. Tarayıcı üzerinden bir terminal açılacaktır. Sırasıyla şu komutlar çalıştırılır:

```bash
sudo apt update
sudo apt upgrade -y
```

Bu kadar. Sunucu artık hazır.

---

## Bölüm 2: Amnezia ile Otomatik Kurulum

### Neden Amnezia?

Xray gibi gelişmiş bir maskeleme protokolünü elle kurmak normalde şunları gerektirir:
- Karmaşık güvenlik duvarı kuralları (`iptables`) yazmak
- SSL/TLS sertifikalarını manuel yönetmek
- Sunucu ve istemci konfigürasyon dosyalarını hatasız eşzamanlamak

**Amnezia tüm bunları tek bir arayüzden otomatik yapar.**

### 2.1 — Amnezia'yı Sunucuya Bağlama

1. [Amnezia](https://amnezia.org/self-hosted)'yı bilgisayara indirilip kurulur.
2. Uygulama açıldığında **"Kendi Sunucumu Kurmak İstiyorum"** seçeneğiyle ilerlenir.
3. Bağlantı bilgileri aşağıdaki gibi doldurulur:

| Alan | Girilecek Değer |
|---|---|
| **Server IP address [:port]** | AWS panelinden kopyaladığın Public IPv4 adresi + `:22` (örnek: `54.123.45.67:22`) |
| **SSH Username** | `ubuntu` |
| **Private key / Password** | `.pem` dosyasının içeriği (aşağıya bak) |

**.pem dosyasının içeriği nasıl aktarılır?**

1. İndirilen `.pem` dosyasına **sağ tıklanır → "Birlikte Aç" → Notepad** (Not Defteri) ile açılır.
2. `-----BEGIN RSA PRIVATE KEY-----` ile başlayan tüm metin seçilir ve kopyalanır.
3. Amnezia'daki ilgili alana yapıştırılır.

### 2.2 — Protokol Seçimi: Xray

Bağlantı kurulduktan sonra Amnezia hangi protokolün kurulacağını soracaktır.

**→ Xray protokolü seçilir.**

#### Xray Nasıl Çalışır?

```
Sen                  Okul Ağı (DPI)           AWS Sunucun       İnternet
 │                        │                        │                │
 │──── Şifrelenmiş ───────►│                        │                │
 │     HTTPS trafiği       │  "Normal web sitesi"   │                │
 │     (aslında VPN)       │   gibi görünür ✓       │                │
 │                         │────────────────────────►│────────────────►│
```

- Bilgisayarından çıkan VPN verisi, sıradan bir HTTPS web sayfasına ait paketler gibi görünür.
- DPI cihazları analiz ettiğinde VPN değil, AWS'deki meşru bir web sitesiyle iletişim görür.
- Tünel kurulur, bağlantı aktif olur.

### 2.3 Kurulum Tamamlandı

Protokol seçimi yapıldıktan sonra Amnezia sunucuya bağlanarak tüm kurulumu otomatik gerçekleştirir. Birkaç dakika içinde bağlantı aktif olur. Geniş çaplı bir test imkânım olmadı, ancak rehberi denemeye yönlendirdiğim herkes başarıyla çalıştırdı. Farklı okul ağlarında da sorunsuz çalıştığına dair geri dönüş aldım.

Eğer bir sorunla karşılaşırsan bir issue açabilirsiniz.
