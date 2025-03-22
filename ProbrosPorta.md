# Проброс порта

> by [babadzackich](https://github.com/babadzakich)

Наша задача настроить на **второй виртуалке** проброс порта с **8080** на **22(ssh)**, чтобы можно было с первой вирталки общаться с третьей. Для удобства обозначим:

```text
1. Первая машина - роутуер
2. Вторая машина - клиент-роутер
3. Третья машина - клиент  
```

## Начальное состояние машины

### Адаптеры

```text
1. Роутер
  enp0s3 - 10.0.2.15   NAT
  enp0s8 - 192.168.15.1 intNet
2. Клиент-роутер
  enp0s3 - 192.168.15.5 intNet
  enp0s8 - 192.168.20.1 intNet2
3. Клиент
  enp0s3 - 192.168.20.2 intNet2
```

Роутер делает форвардинг интернета с 3 на 8 адаптер, Клиент-роутер уже с 3 на 8 для клиента и клиент может юзать интернет.  

## Установка зависимостей

Проверяем, прослушивает ли машина Клиента порт ssh.

``` bash
sudo ss -tuln | grep 22
```

Должно появиться это.

``` bash
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*          
tcp   LISTEN 0      128             [::]:22            [::]:*  
```

Если у вас вывело что-то похожее, то это значит что всё хорошо и порт прослушивается.

В случае если этого нет, то это значит что или ссш не включен или его нет(*скорее всего это*)
Проверим что ссш есть, но не включен

``` bash
sudo systemctl start ssh
sudo systemctl enable ssh
```

Если вы видете следующее

``` bash
Failed to start ssh.service: Unit ssh.service not found.
Failed to enable unit: Unit file ssh.service does not exist.
```

То у вас нет ссш и вы должны его установить и повторить команды ранее

``` bash
sudo apt update
sudo apt -y install openssh-server
sudo systemctl start ssh
sudo systemctl enable ssh
```

Теперь вы должны увидеть

``` bash
Synchronizing state of ssh.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install enable ssh
```

Вы включили ссш и теперь он прослушивается на порту 22.

Проверяем ранней командой

``` bash
sudo ss -tuln | grep 22
```

Теперь вы уже должны видеть следующее

``` bash
tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*          
tcp   LISTEN 0      128             [::]:22            [::]:*  
```

## Настройка проброса порта

Сейчас будем работать с Клиентом-роутером

``` bash
sudo iptables -t nat -A PREROUTING -p tcp -i enp0s3 --dport <порт для подключения с Роутера> -j DNAT --to-destination <IP Клиента куда мы пробрасываем с портом(для ссш - 22)>
sudo iptables -t nat -A POSTROUTING -p tcp -d <IP Клиента куда мы пробрасываем> --dport <порт на который пробрасываем (для ссш - 22)> -j 
```

### Пример для моей машины

``` bash
sudo iptables -t nat -A PREROUTING -p tcp -i enp0s3 --dport 8080 -j DNAT --to-destination 192.168.20.2:22
sudo iptables -t nat -A POSTROUTING -p tcp -d 192.168.20.2 --dport 22 -j MASQUERADE
```

```text
1. -t nat - Мы помещаем правило в таблицу NAT
2. -A/-D - append/delete, добавляем или удаляем правило
3. -p tcp - протокол tcp
4. --dport в зависимости от цели указываем необходимый порт
```

Проверяем наличие наших указаний

``` bash
$ sudo iptables -t nat -L -n -v
... Нам Важны только два наших правила
Chain PREROUTING (policy ACCEPT 11 packets, 740 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  enp0s3 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:192.168.20.2:22
...       
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    8   547 MASQUERADE  all  --  *      enp0s3  0.0.0.0/0            0.0.0.0/0           
    0     0 MASQUERADE  tcp  --  *      *       0.0.0.0/0            192.168.20.2         tcp dpt:22
...
```

Видим здесь наши правила, значит всё хорошо, устанавливаем их следующими командочками

``` bash
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

Видим что-то такое:

``` bash
run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables start
run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables start
```

Отлично, мы всё настроили

## Настройка Роутера

Теперь надо сказать роутеру, как нам дойти до Клиента, воспользуемся командочкой ***route***

``` bash
sudo ip route add <Адрес вашей группы>/24 via <Адрес Клиента> dev <адаптер внутренней сети>
```

Адрес группы это первые 3 октета адреса адаптера внутренней сети Роутера у Клиента-роутера и последний октет = 0

### Пример для моей машины

``` bash
sudo ip route add 192.168.20.0/24 via 192.168.15.5 dev enp0s8
```

Проверяем его наличие

``` bash
$ ip route
default via 10.0.2.2 dev enp0s3 
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 
169.254.0.0/16 dev enp0s3 scope link metric 1000 
192.168.15.0/24 dev enp0s8 proto kernel scope link src 192.168.15.1 
192.168.20.0/24 via 192.168.15.5 dev enp0s8 (Интересующая строчка)
```

## Проверка работы

Теперь нам остаётся проверить подключение с помощью ***netcat***

``` bash
nc -zv <Адрес Клиента> <Порт откуда пробрасываем>
```

### Пример для моей машины

``` bash
nc -zv 192.168.15.5 8080
```

В случае удачи вам должно написать

``` bash
Connection to 192.168.15.5 8080 port [tcp/http-alt] succeeded!
```

Дальше можно и по ссш подключиться

``` bash
ssh user@192.168.15.5 -p 8080
```

Но там уже понадобится создавать ключи шифрования для подключения, а это уже другая история

## В случае если в конце произошёл пиздец

``` bash
1. Проверьте на роутере ip route, чтобы там был путь из нашей сети на адрес Клиента.
2. Проверьте на Клиенте-роутере sudo iptables -t nat -L -n -v чтобы там были наши правила.
3. Возможно вам компастирует мозга фаервол, тогда надо коечто предпринять
    3.1 sudo apt install ufw -y
    3.2 sudo ufw allow 22/tcp
    3.3 sudo ss -tuln | grep 22
    Если есть tcp   LISTEN 0      128          0.0.0.0:22         0.0.0.0:*          
              tcp   LISTEN 0      128             [::]:22            [::]:*   
    То кайф, пробуем еще раз проверить соединение
```

# Спасибо за внимание
