# 🚀 Linux'ta .NET Uygulaması Yayınlama: Hızlı Rehber

> **WorkerService, API, MVC ve diğer .NET projelerini Linux'a nasıl deploy edersiniz?** Bu rehberde adım adım, komut komut anlatıyoruz.

---

## 📦 1. Adım — Projeyi Publish Et

Önce projeyi Linux için derleyip yayına hazır hale getirmemiz gerekiyor. Proje dizininde aşağıdaki komutu çalıştır:

```bash
dotnet publish -c Release -r linux-x64 --self-contained false -o ./linux-x64
```

**Ne anlama geliyor?**

| Parametre | Açıklama |
|---|---|
| `-c Release` | Release modunda derle (optimizasyon açık) |
| `-r linux-x64` | Hedef platform: 64-bit Linux |
| `--self-contained false` | .NET runtime'ı dahil etme, hedef makinede yüklü olsun |
| `-o ./linux-x64` | Çıktıyı bu klasöre yaz |

> 💡 `--self-contained false` kullandığında hedef Linux makinesinde .NET runtime kurulu olmalı. Eğer bağımlılıkları da paketlemek istiyorsan `true` yap, dosya boyutu büyür ama bağımsız çalışır.

---

## 🔌 2. Adım — Linux Makinede SSH Kurulumu

Dosyaları taşımak için `scp` kullanacağız. Bunun çalışması için hedef Linux makinede **SSH sunucusu** kurulu olmalı.

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

**Komutların açıklaması:**
- `apt update` — Paket listesini güncelle
- `apt install openssh-server` — SSH sunucusunu kur
- `systemctl enable --now ssh` — SSH'ı hem şu an başlat hem de her açılışta otomatik çalıştır

### SSH Durumunu Kontrol Et

```bash
sudo systemctl status ssh
```

Çıktıda `active (running)` görüyorsan SSH hazır demektir. ✅

---

## 🌐 3. Adım — Linux Makinenin IP Adresini Öğren

Dosya göndereceğin makinenin IP adresine ihtiyacın var:

```bash
ip a
# ya da
hostname -I
```

Çıktıdan `192.168.x.x` formatındaki yerel IP adresini not al.

> ⚠️ **Önemli:** Dosya gönderen makine ile hedef makine **aynı ağda ve aynı IP bloğunda** (ör. `192.168.1.x`) olmalı.

---

## 📤 4. Adım — Dosyaları SCP ile Gönder

`scp` (Secure Copy), SSH üzerinden dosya transferi sağlar. Dosyaları göndereceğin makinede (Windows veya başka bir Linux) şu komutu çalıştır:

```bash
scp -r C:\temp\linux-x64 user@192.168.1.45:/target/directory/
```

**Parçalara ayıralım:**

| Parça | Açıklama |
|---|---|
| `-r` | Klasörü alt dosyalarıyla birlikte recursive gönder |
| `C:\temp\inux-x64` | Göndereceğin kaynak klasör veya dosya |
| `user@192.168.1.45` | Hedef makinedeki kullanıcı adı ve IP |
| `:/target/directory/` | Hedef makinedeki hedef dizin |

### ❌ "No such file or directory" Hatası Alıyorsan

Hedef dizine yazma izni yoktur. Şu komutla izni ver:

```bash
sudo chown -R user:user /target/directory
# Örnek:
sudo chown -R myuser:myuser /home/appPath/
```

Ardından `scp` komutunu tekrar çalıştır.

---

## 🔍 5. Adım — Ağ Bağlantısını Doğrula

### Ping Testi

```bash
ping 192.168.1.45
```

Ping gidiyorsa ama hâlâ dosya gönderemediysen sorun büyük ihtimalle **firewall**'dır.

### Firewall Durumunu Kontrol Et

```bash
sudo ufw status
```

Eğer `Status: inactive` diyorsa sorun değil, firewall zaten kapalı demektir.

Eğer aktifse SSH portunu açman gerekiyor:

```bash
sudo ufw allow ssh
# ya da port numarasıyla:
sudo ufw allow 22
```

---

## ▶️ 6. Adım — Uygulamayı Test Amaçlı Çalıştır

Deploy etmeden önce uygulamanın düzgün ayağa kalkıp kalkmadığını test etmek için:

```bash
dotnet MyApp.dll
```

Eğer hata alıyorsan, servis haline getirmeden önce bu noktada düzeltmeni kolaylaştırır. Uygulama başarıyla ayağa kalkıyorsa sonraki adıma geç.

---

## 📁 7. Adım — Uygulama Dizinini Seç

Servis dosyasını oluşturmadan önce uygulamanın hangi dizine yerleştirileceğine karar vermek gerekiyor. Linux'ta bunu yaparken uygulama tipine göre doğru dizini seçmek hem düzeni hem de güvenliği etkiler.

| Dizin | Ne zaman kullanılır? |
|---|---|
| `/opt/myapi` | ✅ **Önerilen.** Paket yöneticisi dışından kurulan üçüncü parti uygulamalar için standart Linux dizinidir. WorkerService, API, MVC — her tür .NET uygulaması için uygundur. |
| `/var/www/myapi` | Nginx veya Apache arkasında çalışan web uygulamaları için kabul edilebilir, özellikle Ubuntu/Debian topluluğunda yaygındır. Ancak özünde web sunucusu içeriği için tasarlanmış bir dizindir. |
| `/srv/myapi` | Servis verisi için tasarlanmış bir dizindir, bazı dağıtımlarda tercih edilir ama görece daha az yaygındır. |

**Kısa karar rehberi:** WorkerService veya herhangi bir arka plan servisi çalıştırıyorsan `/opt` her zaman güvenli ve doğru seçimdir. Web'e açık uygulamalarda da `/opt` kullanmak sorun yaratmaz.

Bu rehberde `/opt/myapi` dizinini kullanacağız. Dizini oluşturup dosyaları taşı:

```bash
sudo mkdir -p /opt/myapp
sudo cp -r /home/appPath/myapp/* /opt/myapp/
sudo chown -R www-data:www-data /opt/myapp
```

Veya dosyaları 4. adımda doğrudan ilgili klasöre de aktarabilirsin. 

---

## ⚙️ 8. Adım — Systemd Servis Dosyası Oluştur

Linux'ta uygulamaları arka planda, sistem başlangıcında otomatik olarak ve çöktüğünde yeniden başlayacak şekilde çalıştırmak için **systemd service** kullanıyoruz.

```bash
sudo nano /etc/systemd/system/myapi.service
```

Açılan editöre aşağıdaki içeriği yapıştır:

```ini
[Unit]
Description=My .NET 10 API Service
After=network.target

[Service]
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/dotnet /opt/myapi/MyApp.dll
Restart=always
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=myapp
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false
Environment=ASPNETCORE_URLS=http://0.0.0.0:5000

[Install]
WantedBy=multi-user.target
```

**Alanların açıklaması:**

| Alan | Açıklama |
|---|---|
| `Description` | Servisin açıklaması |
| `After=network.target` | Servis ağ bağlantısı hazır olduktan sonra başlasın |
| `WorkingDirectory` | Uygulamanın çalışacağı dizin |
| `ExecStart` | Servisi başlatacak komut |
| `Restart=always` | Uygulama çökerse otomatik yeniden başlat |
| `RestartSec=10` | Yeniden başlatmadan önce 10 saniye bekle |
| `KillSignal=SIGINT` | Servis durdurulurken gönderilecek sinyal (graceful shutdown) |
| `SyslogIdentifier` | Log kayıtlarında görünecek isim |
| `User=www-data` | Servisi bu kullanıcı olarak çalıştır |
| `ASPNETCORE_ENVIRONMENT` | Ortam değişkeni: Production modunda çalışsın |
| `ASPNETCORE_URLS` | Uygulamanın dinleyeceği adres ve port |

> 🔴 **WorkerService kullanıcıları dikkat:** `ASPNETCORE_URLS` satırını **kaldır**. WorkerService, HTTP portu dinlemez. Bu satır yalnızca **API ve MVC** gibi web tabanlı uygulamalar içindir.

> 🔐 **`www-data` kullanıcısı hakkında:** Bu, web sunucularının (Nginx, Apache) kullandığı standart düşük yetkili kullanıcıdır. Uygulamayı root yerine bu kullanıcıyla çalıştırmak güvenlik açısından önerilen yaklaşımdır. Eğer uygulaman belirli dosya veya dizinlere erişmesi gerekiyorsa, o dizinlerin sahibini `www-data` yapman gerekebilir: `sudo chown -R www-data:www-data /opt/myapp`

Dosyayı kaydetmek için: **Ctrl + X → Y → Enter**

---

## 🟢 9. Adım — Servisi Etkinleştir ve Başlat

```bash
# Systemd'yi yeni servis dosyasından haberdar et
sudo systemctl daemon-reload

# Servisi oluştur (sistem açılışında otomatik başlasın)
sudo systemctl enable myapp.service

# Servisi şimdi başlat
sudo systemctl start myapp.service

# Servisin durumunu kontrol et
sudo systemctl status myapp.service
```

Çıktıda `active (running)` görüyorsan tebrikler, uygulamanı Linux'a başarıyla deploy ettin! 🎉

---

## 🧰 Bonus: Yararlı Servis Komutları

```bash
# Servisi durdur
sudo systemctl stop myapp.service

# Servisi yeniden başlat
sudo systemctl restart myapp.service

# Canlı logları takip et
sudo journalctl -u myapp.service -f
```

---

## 📋 Özet: Adım Adım Checklist

- [ ] `dotnet publish` ile uygulamayı derle
- [ ] Hedef Linux makinede SSH'ı kur ve başlat
- [ ] Hedef makinenin IP adresini öğren
- [ ] Aynı ağda olduğunu doğrula (ping testi)
- [ ] `scp` ile dosyaları gönder
- [ ] `dotnet app.dll` ile test çalıştırması yap
- [ ] Uygulama dizinini seç ve dosyaları taşı (`/opt/myapp` önerilir)
- [ ] `/etc/systemd/system/` altına `.service` dosyasını oluştur
- [ ] `daemon-reload → enable → start → status` sırasını takip et

---

> ✍️ *Bu rehberde anlatılanlar .NET 8/9/10 ile test edilmiştir. Farklı bir .NET sürümü kullanıyorsan komutlar büyük ölçüde aynı çalışır.*
