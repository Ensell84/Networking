# Установка и настройка виртуалки

После создания виртуальной машины на Debian:

Devices --> Insert Guest Additions CD image

```sh
sudo -i
cd media/cdrom0/
sh VboxLinuxAdditions.run
```

(Если в `cdrom0` ниче нет -> откройте на рабочем столе полупрозрачный диск -> заново зайдите в папку)

```sh
sudo apt install wireshark mc
usermod -a -G wireshark urname
```

**RESTART!**

Заходим в wireshark --> выбираем `enp0s3`  
В терминале `ping mail.ru` --> наблюдаем в wireshark активность
