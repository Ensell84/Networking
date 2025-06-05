# Adding new adapter

Добавим новый адаптер в виртуалку и настроим его для работы в сети.

## Ip command

`ip a s` —> address show

```sh
bondar@bondar:~$ ip a s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0c:03:e1 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86354sec preferred_lft 86354sec
    inet6 fd00::cc74:9f9:1e38:749c/64 scope global temporary dynamic 
       valid_lft 86357sec preferred_lft 14357sec
    inet6 fd00::a00:27ff:fe0c:3e1/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 86357sec preferred_lft 14357sec
    inet6 fe80::a00:27ff:fe0c:3e1/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ec:53:0f brd ff:ff:ff:ff:ff:ff
    inet6 fe80::8878:9ce8:cde7:2d19/64 scope link tentative noprefixroute 
       valid_lft forever preferred_lft forever
```

`1: lo:` Loopback адаптер - виртуальный интерфейс для общения системы с собой же

- `link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00` - мак адрес и broadcast мак адрес
- `inet 127.0.0.1/8` - ipv4 адрес (scope host -- адрес валиден только на нашей машине)
- `valid_lft forever preferred_lft forever`: valid lifetime и preferred lifetime
  - _forever_ - статически присвоены и никогда не истекут

`2: enp0s3:` Ethernet сетевой адаптер

- `link/ether 08:00:27:d4:82:a3 brd ff:ff:ff:ff:ff:ff` - мак и broadcast мак
- `inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3`
  - 10.0.2.255 -> получателем будут все машины в данной подсети (broadcast адрес)
  - `scope global` - доступен извне
  - `dynamic` - адрес динамичный (dhcp сервер в данный момент - виртуальная машина - VirtualBox NAT)
  - `valid_lft 86325sec preferred_lft 86325sec` -- как долго данный адрес будет считаться валидным (задается DHCP сервером)
- inet6 - адреса ipv6

## Добавляем в виртуалку Internal Network Адаптер

В настройках виртуалки в разделе сети:

- Адаптер 1 --> уже настроенная сеть, не трогаем
![alt text](/pics/image.png)

- Адаптер 2 --> включаем и выбираем Internal Network
![alt text](/pics/image-1.png)

У нас появляется `enp0s8` адаптер в `ip a s` и пропадает интернет:

![alt text](/pics/image3.png)
`ip route show default` --> пусто (у системы нет Default Gateway)

## Troubleshooting

Если ставили систему с xfce(KDE наверное так же) то рулит сетью у нас `NetworkManager`  

```sh
sudo systemctl status NetworkManager.service
```

![alt text](/pics/image4.png)

Для взаимодействия пользуемся `nmcli`.

Первый запуск `nmcli` покажет что у нас теперь есть 2 соединения:
![alt text](/pics/image5.png)
При этом как мы можем видеть NetworkManager пытается подключится к `enp0s8`.  

И как видно из предыдущего скриншота - он пытается получить адрес от DHCP сервера, но так как мы выбрали Internal Network в VirtualBox - DHCP сервера нет и соединение не устанавливается.

```sh
nmcli con show -->  показать соединения о которых NetworkManager знает
nmcli dev --> состояния устройств
nmcli --> общие данные
nmcli dev show enp0s3 --> подробная информация про enp0s3
nmcli con show "Wired connection 1" --> подробная информация о соединении
```

```sh
shrek@deb:~$ nmcli con show
NAME                UUID                                  TYPE      DEVICE 
lo                  a5a37289-6c35-4d6e-936a-05e0a04b3335  loopback  lo     
Wired connection 1  6c3b3ca9-1eea-44c6-9147-7fbba78f7e04  ethernet  --     
```

Поднимем соединение вручную:

```sh
shrek@deb:~$ nmcli con up id "Wired connection 1" ifname enp0s3
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/23)

shrek@deb:~$ ip route show default
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 
```

Теперь у нас есть Default Gateway и интернет работает.

### Решение проблемы с NetworkManager

> [!IMPORTANT] Явно зададим что "Wired connection 1" -- должен подключатся через enp0s3

```sh
sudo nmcli con modify "Wired connection 1" connection.interface-name enp0s3
```

Теперь мы имеем 2 подключения (NetworkManager создал второе сам после **перезагрузки**):

```sh
shrek@deb:~$ nmcli con show
NAME                UUID                                  TYPE      DEVICE 
Wired connection 2  c17a454f-0a8c-3cc1-85d2-40e41844456d  ethernet  enp0s8 
Wired connection 1  07774190-c6f1-4861-a1fa-82417503add7  ethernet  enp0s3 
lo                  822ce0b2-52aa-449f-b546-cf7281dbe5d4  loopback  lo  

shrek@deb:~$ nmcli dev
DEVICE  TYPE      STATE                                  CONNECTION         
enp0s3  ethernet  connected                              Wired connection 1 
lo      loopback  connected (externally)                 lo                 
enp0s8  ethernet  connecting (getting IP configuration)  Wired connection 2 
```

Как видно - Wired connection 2 пытается получить адрес от DHCP сервера, но мы установили `enp0s8` как Internal Network в VirtualBox -> DHCP сервер отсутствует.

Все тоже самое можно сделать через графический интерфейс NetworkManager в DE, но с помощью `nmcli` это можно сделать даже без графической оболочки.

## Default gateway

**Default Gateway** - Router на который наш пк отправляет все network пакеты с итоговым адресом назначения вне нашей локальной сети

```sh
bondar@bondar:~$ ip route show default
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100 
```

- в случае виртуальной машины - это VirtualBox NAT "Router" с ip = 10.0.2.2
- в реальной жизни - скорее всего домашний wifi роутер
- Зачастую назначается DHCP сервером

## Что такое device? connection? interface?
