# discord-only-byedpi

Tüm internet trafiği yerine sadece Discord'u DNS adresinden geçirme.

[hufrea/byedpi](https://github.com/hufrea/byedpi) için hufrea'ya teşekkürler.

## 1. ByeDPI'ı derleyin

```bash
sudo dnf install gcc make git
```

Ya da hangi dağıtıma ait paket yöneticisini kullanıyorsanız.

```bash
git clone https://github.com/hufrea/byedpi
cd byedpi && make
mkdir -p ~/.local/bin
cp ciadpi ~/.local/bin/
```

## 2. dnscrypt-proxy'yi kurun

```bash
sudo dnf install dnscrypt-proxy   # kendi dağıtımınıza göre paket farklı olabilir
sudo systemctl enable --now dnscrypt-proxy
dig discord.com @127.0.0.1        # çıktıda NOERROR ve 162.159.x.x adresleri görünmeli
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

> `ExecStart`'taki argümanlar Superonline için çalıştı. Kendi internet servis sağlayıcınız için [ByeDPI README](https://github.com/hufrea/byedpi)'sini okuyun ve deneme yapın.

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

Bu kadar basit.

## Kaldırma

```bash
systemctl --user disable --now ciadpi-discord.service
rm ~/.config/systemd/user/ciadpi-discord.service
sudo rm /etc/systemd/resolved.conf.d/discord.conf
sudo systemctl restart systemd-resolved
rm ~/.local/share/applications/com.discordapp.Discord.desktop
```
