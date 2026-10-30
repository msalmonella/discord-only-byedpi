# [REHBER] discord-only-byedpi

Tüm internet trafiği yerine sadece Discord'u ByeDPI proxy'si ve DNS yönlendirmesi ile açma.

[hufrea/byedpi](https://github.com/hufrea/byedpi) için hufrea'ya teşekkürler.

> **Test edilen ortam:** Fedora, Flatpak Discord, Sansuroffline.

## 1. ByeDPI'ı derleyin

```bash
sudo dnf install gcc make git
```

Ya da hangi dağıtıma ait paket yöneticisini kullanıyorsanız.

```bash
git clone https://github.com/hufrea/byedpi
cd byedpi && git checkout v0.17.3 && make
mkdir -p ~/.local/bin
cp ciadpi ~/.local/bin/
```

`v0.17.3`, bu rehberin test edildiği sürümdür.

**Derlemek istemiyorsanız**, aynı sürümün hazır dosyasını da kullanabilirsiniz:

```bash
curl -L -o ciadpi.tar.gz "https://github.com/hufrea/byedpi/releases/download/v0.17.3/byedpi-17.3-$(uname -m).tar.gz"
tar -zxf ciadpi.tar.gz
mkdir -p ~/.local/bin
find . -type f -name 'ciadpi-*' -exec cp {} ~/.local/bin/ciadpi \;
chmod +x ~/.local/bin/ciadpi
```

## 2. dnscrypt-proxy'yi kurun

```bash
sudo dnf install dnscrypt-proxy   # kendi dağıtımınıza göre paket farklı olabilir
sudo systemctl enable --now dnscrypt-proxy
dig discord.com @127.0.0.1        # çıktıda NOERROR ve 162.159.x.x adresleri görünmeli
```

`dig` hata verirse ya da cevap gelmezse sunucu listesini sabitleyin:

```bash
sudo nano /etc/dnscrypt-proxy/dnscrypt-proxy.toml
```

Dosyada şu satırı bulup şöyle değiştirin:

```
server_names = ['cloudflare', 'google', 'quad9-dnscrypt-ip4-filter-pri']
```

Sonra:

```bash
sudo systemctl restart dnscrypt-proxy
```

## 3. Discord alan adlarını dnscrypt'e yönlendirin

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/discord.conf <<'EOF'
[Resolve]
DNS=127.0.0.1
Domains=~discord.com ~discordapp.com ~discord.gg ~discordapp.net ~discord.media
EOF
sudo systemctl restart systemd-resolved
getent hosts discord.com
```

`getent` çıktısı `162.159.x.x` ya da `2606:4700:...` ile başlamalı. Sadece Discord alan adları etkilenir, diğer her şey normal DNS'ini kullanır.

## 4. ciadpi servisini oluşturun

```bash
mkdir -p ~/.config/systemd/user
tee ~/.config/systemd/user/ciadpi-discord.service <<'EOF'
[Unit]
Description=ByeDPI proxy for Discord

[Service]
ExecStart=%h/.local/bin/ciadpi --ip 127.0.0.1 --port 1080 --oob=1 --disorder=3+s --split=3 --ttl=3
Restart=on-failure

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable --now ciadpi-discord.service
```

> `ExecStart`'taki argümanlar SansürOffline için çalıştı. Kendi internet servis sağlayıcınız için [ByeDPI README](https://github.com/hufrea/byedpi)'sini okuyun ve deneme yapın.

Çalışmazsa şu alternatif argümanları deneyebilirsiniz ([byedpi-turkey](https://github.com/elrondforwin/byedpi-turkey) profillerinden):

```
--oob=1 --auto=torst,redirect,ssl_err
```

```
--proto=tls --split=1 --disorder=1 --fake=-1 --md5sig --auto=torst,redirect,ssl_err --timeout=2
```

Argümanları değiştirdikten sonra servisi yeniden başlatın:

```bash
systemctl --user restart ciadpi-discord.service
```

## 5. Test edin

```bash
curl -I --socks5-hostname 127.0.0.1:1080 https://discord.com
```

`HTTP/2 200` geliyorsa başarılı.

## 6. Discord'u proxy ile açın

Discord'u argümanlarla açmak gerekiyor.

**Flatpak** ise önce terminale şunu yapıştırarak deneyin:

```bash
flatpak run com.discordapp.Discord --proxy-server="socks5://127.0.0.1:1080"
```

**Native** kurulu ise:

```bash
discord --proxy-server="socks5://127.0.0.1:1080"
```

## 7. Kalıcı yapın (ikona tıklayarak çalıştırmak için)

Herhangi bir sorun yoksa:

```bash
cp /var/lib/flatpak/exports/share/applications/com.discordapp.Discord.desktop ~/.local/share/applications/
nano ~/.local/share/applications/com.discordapp.Discord.desktop
```

`Exec=` kısmında `com.discordapp.Discord`'dan hemen sonra şunu ekleyin:

```
--proxy-server=socks5://127.0.0.1:1080
```

Sonrasında sistem üzerindeki uygulamaları yenileyin:

```bash
update-desktop-database ~/.local/share/applications
```

Flatpak tarafında alınan güncellemeler bu ayarı sıfırlayabiliyor. Uygulamayı güncelledikten sonra `.desktop` dosyasına girip argümanı tekrar eklemeniz ve `update-desktop-database` komutunu yeniden çalıştırmanız gerekebilir.

Bu kadar basit.

## Sorun giderme

- **Discord proxy'siz açılıyor:** Önce tamamen kapatın, arka planda açık kalan süreç bayrağı yok sayar.
  ```bash
  pkill -f -i discord
  ```
- **Terminalden açılıyor, menüden açılmıyor:** `.desktop` dosyasındaki `Exec=` satırında `--proxy-server` bayrağı var mı kontrol edin.
- **Flatpak kullanmıyorsanız:** Kısayol dosyası genelde `/usr/share/applications/discord.desktop` konumundadır, aynı şekilde kopyalayıp `Exec=` satırının sonuna bayrağı ekleyin.
- **`curl` SOCKS5 hatası veriyor:** `getent hosts discord.com` çıktısına bakın, Cloudflare adresi (`162.159.x.x`) değilse 3. adımı kontrol edin.

## Notlar

- Sadece `--proxy-server` ile açılan Discord proxy'den geçer. Bayraksız açılan örnek doğrudan bağlanır ve çalışmaz.
- Sesli sohbet UDP kullanır, ByeDPI'nın SOCKS5 proxy'si ise yalnızca TCP taşır. Sesli kanalların çalışacağı garanti değildir, test edilmemiştir.

## Kaldırma

```bash
systemctl --user disable --now ciadpi-discord.service
rm ~/.config/systemd/user/ciadpi-discord.service
sudo rm /etc/systemd/resolved.conf.d/discord.conf
sudo systemctl restart systemd-resolved
rm ~/.local/share/applications/com.discordapp.Discord.desktop
```

## Teşekkürler

- [hufrea/byedpi](https://github.com/hufrea/byedpi)
- [elrondforwin/byedpi-turkey](https://github.com/elrondforwin/byedpi-turkey) (alternatif profiller ve dnscrypt ayarı için)
- [DNSCrypt/dnscrypt-proxy](https://github.com/DNSCrypt/dnscrypt-proxy)
