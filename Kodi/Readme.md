# HTPC на базе Arch + Kodi с поддержкой HDR

**Самый пиздец: на момент написания standalone kodi и HDR не работают на nvidia**

## Железо

+ AMD Ryzen 5 3400G
+ Denon AVR-S760H

## Софт

+ sudo pacman -Syu kodi-gles
+ yay -a kodi-standalone-services

Для корректного вывода не ставить pipewire, чтобы работал прямой вывод звука.  
Иначе даже с прямым выводом в kodi всё равно вывод идёт в pipewire,
который придётся донастраивать, к тому же он не умеет в некоторые виды DTS

## Системные параметры

Добавить target для systemd: `/etc/udev/rules.d/99-kodi-ready.rules`  
```
SUBSYSTEM=="drm", KERNEL=="card1", TAG+="systemd"
```

Добавить в параметры ядра (это ещё было бы неплохо уточнить):
```
amdgpu.dcfeaturemask=0x8
```


## Сервис kodi

`sudo systemctl enable kodi-gles`  
`sudo systemctl edit kodi-gles`  
After, Wants - обязательно нужно дождаться готовности графики, иначе HDR
при запуске c высокой вероятностью не подхватится, будет доступен после перезапуска.  
Restart меняем для того, чтобы сервис перезапускался после выхода из плеера
и пользователя не кидало в TTY.  
ExecStart - сразу указываем, что надо использовать alsa, иначе сначала полезет в pipewire.
```
[Unit]
After=dev-dri-card1.device
Wants=dev-dri-card1.device

[Service]
ExecStart=
ExecStart=/usr/bin/kodi-standalone --audio-backend=alsa
User=valheru
Group=valheru
Restart=always
```

card1 проверять в зависимости от конкретной системы, может быть и card0
См:
```
[valheru@htpc ~]$ ls -l /dev/dri
итого 0
drwxr-xr-x  2 root root         80 янв 22 22:09 by-path
crw-rw----+ 1 root video  226,   1 янв 22 22:09 card1
crw-rw-rw-  1 root render 226, 128 янв 22 22:09 renderD128
[valheru@htpc ~]$ ls -l /sys/class/drm
итого 0
lrwxrwxrwx 1 root root    0 янв 22 22:09 card1 -> ../../devices/pci0000:00/0000:00:08.1/0000:04:00.0/drm/card1
lrwxrwxrwx 1 root root    0 янв 22 22:09 card1-DP-1 -> ../../devices/pci0000:00/0000:00:08.1/0000:04:00.0/drm/card1/card1-DP-1
lrwxrwxrwx 1 root root    0 янв 22 22:09 card1-HDMI-A-1 -> ../../devices/pci0000:00/0000:00:08.1/0000:04:00.0/drm/card1/card1-HDMI-A-1
lrwxrwxrwx 1 root root    0 янв 22 22:09 renderD128 -> ../../devices/pci0000:00/0000:00:08.1/0000:04:00.0/drm/renderD128
-r--r--r-- 1 root root 4096 янв 22 22:26 version
```
Теоретически пользователя следует добавить в группы `video,render,audio,input`, но пока работает с добавлением только в `video` — хз, может и это лишнее

## Упрощённый вариант, если есть nvidia и не требуется поддержка HDR  

+ sudo pacman -Syu kodi hyprland

Конфиг для hyprland: `~/.config/hypr/hyprland.conf`  
```
monitorv2 {
    output = HDMI-A-1
    mode = 3840x2160@60
}

exec-once = dbus-run-session kodi --windowing=wayland --audio-backend=alsa; hyprctl dispatch exit

animations {
    enabled = false
}

```
Сервис: `/etc/systemd/system/kodi-hyprland.service`  
```
[Unit]
Description=Kodi via Hyprland (Wayland)
After=remote-fs.target systemd-user-sessions.service network-online.target polkit.service upower.service
Wants=network-online.target polkit.service upower.service
Conflicts=getty@tty1.service

[Service]
User=valheru
Group=valheru
PAMName=login

ExecStart=/usr/bin/start-hyprland 

Restart=always
RestartSec=3

StandardInput=tty
StandardOutput=journal
TTYPath=/dev/tty1

[Install]
WantedBy=multi-user.target
```
