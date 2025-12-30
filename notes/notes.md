## remapping with xremap

- `/etc/xremap/config.yml`

```yml
modmap:
  - name: Numpad Minus
    remap:
      KEY_KPMINUS: KEY_MINUS
```

- `/etc/systemd/system/xremap.service`

```toml
[Unit]
Description=xremap

[Service]
Type=oneshot
KillMode=process
ExecStart=/usr/bin/xremap /etc/xremap/config.yml
ExecStop=/usr/bin/killall xremap
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

### Drivers:

```sh
# amd
xf86-video-amdgpu
xf86-video-ati
amd-ucode
amdvlk

# intel
intel-media-driver
vulkan-intel
intel-gmmlib
```

# Some notes and commands

## set thems and icons for flatpak apps

check theme and icon

set theme and icon

```
sudo flatpak override --system --reset
sudo flatpak override --filesystem=$HOME/.themes/
sudo flatpak override --filesystem=$HOME/.icons/
sudo flatpak override --env=GTK_THEME=Orchis-Grey-Dark-Compact
sudo flatpak override --env=ICON_THEME=Papirus
```

flatpak global override configs are in : `cd /var/lib/flatpak/overrides`

---

### Ran into an issue related to systemd `a failed ret 0x0`

```
$ sudo nvim /etc/mkinitcpio.conf
# pass parameter in `modules=()`
$ modules = (amdgpu)
$ sudo mkinitcpio -p linux
```

## always youtube theatre mode

- go to youtube.com
- run the following code in the console

```
document.cookie = 'wide=1; expires='+new Date('3099').toUTCString()+'; path=/';
```
