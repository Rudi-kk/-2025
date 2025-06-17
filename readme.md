

## 1. Базовая настройка системы

### Изменение имени хоста
```bash
hostnamectl hostname "имя_машины"
exec bash
# НЕ ЗАБУДЬТЕ ОТКЛЮЧИТЬ  firewall  командой systemctl disable firewalld --now  (если не отключить gre и ospf могут не работать) вроде на  hq-r

```

## 2. Создание пользователя net_admin (HQ-RTR и BR-RTR)
```bash
# Создание пользователя
adduser net_admin

# Установка пароля 
passwd net_admin

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
net_admin ALL=(ALL:ALL)NOPASSWD:ALL

```

## 3. Создание пользователя sshuser (на HQ-SRV и BR-SRV)

```bash
# Создание пользователя
useradd -m -u 1010 sshuser

# Установка пароля 
passwd sshuser

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL

```

## 4. Настройка SSH (на HQ-SRV и BR-SRV)

### Создание баннера
```bash
nano /etc/mybanner
```
Содержимое файла:
```
Authorized access only
```

### Настройка SSH daemon
```bash
nano /etc/openssh/sshd_config
```
Добавить/изменить параметры:
```
Port 2025
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser
```

### Перезапуск службы
```bash
systemctl restart sshd.service
```


## 5. Настройка маршрутизации

### На ISP - включение IP forwarding
```bash
nano /etc/net/sysctl.conf
```
```
net.ipv4.ip_forward = 1

```
## после этого нужнно перезагрузить ISP
```
reboot
```
## 6. Настройка GRE туннелей
### На BR-RTR (Branch Router)
**Сетевые настройки:**
- Profile name: tun1
- Device: tun1
- Mode: GRE
- Parent: ens34
- Local IP: 172.16.5.2
- Remote IP: 172.16.4.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.2/24
  - Gateway: 192.168.0.1

### На HQ-RTR (Headquarters Router)
**Сетевые настройки:**
- Profile name: tun1
- Device: tun1
- Mode: GRE
- Parent: ens34
- Local IP: 172.16.4.2
- Remote IP: 172.16.5.2
- IPv4 Configuration: Manual
  - IP: 192.168.0.1/24
  - Gateway: 192.168.0.2

**Если GRE тунель не работает пропинговать с HQ-R BR-R по его айпи, если пинги не проходят то перезаагрузить машину ISP, также если это не помогло то можно попробовать временно отключить другие сетевые интерфейсы (hqin и brin)**




## 7. Настройка OSPF маршрутизации

### На HQ-R и BR-R - активация FRR
```bash
nano /etc/frr/daemons
```
Изменить строку:
```
ospfd=yes
```

```bash
systemctl enable --now frr
```

### На HQ-RTR - настройка OSPF
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/26 area 0 # указать HQ-IN подсеть
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

# Настройка TTL для туннеля
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr
```


### На BR-RTR - аналогичная настройка
```bash
vtysh
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.6.0/27 area 0  # указать BR-IN подсеть
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do wr
exit
exit

# Настройка TTL для туннеля
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr

#ТЕПЕРЬ ПЕРЕЗАГРУЗИТЕ HQ-R и BR-R
```

### Проверка OSPF
```bash
# Вход в FRR консоль
vtysh

# Проверка соседей OSPF, если ничего не вывело то забиваем и переходим к следующему заданию
show ip ospf neighbor

# Выход из консоли
exit
```
## 8. Настройка DHCP сервера (на HQ-RTR)

### Указание интерфейса для DHCP
```bash
nano /etc/sysconfig/dhcpd
```
```
DHCPARGS=ens35
```

### Создание конфигурационного файла
```bash
# в файле /etc/dhcp/dhcpd.conf.example показан пример dhcp

nano /etc/dhcp/dhcpd.conf
```

Содержимое файла:
```
option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;
authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
  range 172.16.0.3 172.16.0.8;
  option routers 172.16.0.1;
}
```

### Запуск и автозагрузка службы
```bash
systemctl enable --now dhcpd
```
### RAID 5 на HQ-SRV:
```bash
#команда ниже должна вывести 4 диска sda, sdb, sdc, sdd 
lsblk

#Создание raid5
mdadm --zero-superblock /dev/sd[b-d]

mkfs -t ext4 /dev/md0

mkdir /etc/mdadm

echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.con

mkdir /mnt/raid5

nano /etc/fstab
#добавить эту строчку в fstab
/dev/md0  /mnt/raid5  ext4  defaults  0  0

mount -a
```

Проверяем монтирование:
```yml
df -h | grep /mnt/raid5
```
> Вывод должен быть:
> ```yml
> /dev/md0  2.0G  24K  1.9G  1%  /mnt/raid5
> ```

#### Настройка NFS SERVER на HQ-SRV
```
mkdir /mnt/raid5/nfs

chmod 766 /mnt/raid5/nfs

nano /etc/exports
#добавляем строчку ниже | ВАЖНО ВМЕСТО 172.16.0.0/28 указывайте подсеть которая будет на демо
/mnt/raid5/nfs 172.16.0.0/28(rw,no_root_squash)

exportfs -arv

systemctl enable --now nfs-server
```
#### Настройка NFS CLIENT (вроде на hq-cli смотрите по заданию кому надо монтировать)
```
mkdir /mnt/nfs

chmod 777 /mnt/nfs

nano /etc/fstab
#добавляем строчку где 172.16.0.2 айпи nfs сервера 
172.16.0.2:/mnt/raid5/nfs  /mnt/nfs  nfs  defaults  0  0 

mount -a
```
Проверяем монтирование:
```yml
df -h | grep /mnt/nfs
```
> Вывод должен быть:
> ```yml
> 172.16.0.2:/mnt/raid5/nfs  2,0G  0  1,9G  0%  /mnt/nfs
> ```

### Настройка chrony на HQ srv

Правим файл командой  **`nano /etc/chrony.conf`** :
```yml
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html
#pool pool.ntp.org iburst

server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0
```
> ![image](https://github.com/user-attachments/assets/1a80b134-5e9e-4310-9a1a-1c1eca654200)

Запускаем и добавляем в автозагрузку утилиту **chronyd**:
```yml
systemctl enable --now chronyd
```

Проверяем работоспособность (вывод должен быть как в примере)
```yml
chronyc sources
```
> Вывод:
> ```yml
> MS Name/IP address        Stratum  Poll  Reach  LastRx  Last  sample
> =============================================================================
> ^/ localhost.localdomain  0        8     377    -       +0ns[  +0ns] +/-  0ns
> ```
```yml
chronyc tracking | grep Stratum
```
> Вывод:
> ```yml
> Stratum: 5
> ```

## Примечания
В задание есть тема установить яндекс браузер, ищите это легкие баллы 



- 
https://ru.files.fm/u/f9g3bthbwu



Глава 1: 
hostnamectl set-hostname «имя_машины»; exec bash
имена HQ-SRV и BR-SRV, маршрутизаторы - HQ-RTR и BR-RTR, а рабочие станции назывались HQ-CLI и BR-DC.
Глава 2: Раздача адресов
nmtui:
                 			isp          
ens34 (bf)      					          ens35 (c9)      
172.20.4.0/28   				             172.20.5.0/28   
172.20.4.1/28   				              172.20.5.1/28   
ens34 (ba)      					            ens34 (54)      
hq-rtr         					              br-rtr          
ens35 (c4)      				              	ens35 (5e)      
172.20.0.1/26   				              172.20.6.1/27   
ens33 (d6)      ens34 (af)      			ens33 (b2)      (Он винда)
172.20.0.2/26   172.20.0.3/28   		172.20.6.2/27   172.20.6.3/27
hq-srv          hq-cli          			br-srv          br-DC

Глава 3: Создание верных слуг
создал пользователя sshuser, который мог бы выполнять любые команды 
useradd -m -u 1010 sshuser
passwd sshuser
дающую sshuser неограниченные права:
nano /etc/sudoers
Добавил:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
: Ctrl+X, Y, Enter. 
Глава 4: 
предупреждение для всех, кто попытается войти без разрешения:
nano /etc/mybanner
строгое предупреждение:
Authorized access only
Затем настроил защитные механизмы SSH, изменив конфигурационный файл:
nano /etc/openssh/sshd_config
Установил:
#port 22, раскоменчиваем и пишем port 2024
Banner /etc/mybanner
MaxAuthTries 2
ДОБАВИТЬ строчку - AllowUsers sshuser
После этого перезапустил службу SSH, чтобы изменения вступили в силу:
systemctl restart sshd.service
под надежной защитой.

Глава 5: Автоматическая раздача адресов
администратор настроил DHCP-сервер на HQ-RTR. Сначала он указал, какой интерфейс будет раздавать адреса:		
nano /etc/sysconfig/dhcpd
Добавил строку:
DHCPARGS=ens35
Затем создал конфигурационный файл, взяв за основу пример:
cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
Прописал основные параметры:
Доменное имя "au-team.irpo"
Адреса DNS-серверов
option domain-name-servers 172.16.0.2;
Время аренды адресов
default-lease-time 6000;
max-lease-time 72000;

Диапазон раздаваемых адресов
authoritative;
subnet 172.16.0.0 netmask 255.255.255.192 {
range 172.16.0.3 172.16.0.8;
option routers 172.16.0.1;
}

После этого включил и запустил службу DHCP:
systemctl enable --now dhcpd
Теперь все новые автоматически получали свои адреса.
Глава 6: тоннель между замками
Туннель HQ и удалённой крепостью BR. Но для этого сначала нужно было получить разрешение от — сервера ISP.
ISP:
nano /etc/net/sysctl.conf
Найдя строку net.ipv4.ip_forward, он изменил её значение на 1
net.ipv4.ip_forward = 1
Теперь пакеты могли свободно проходить через ISP. Вдохновлённый, администратор взял волшебный инструмент nmtui и начал настраивать GRE-тоннель между HQ-RTR и BR-RTR. Это было подобно прокладыванию подземного хода — невидимого для посторонних глаз, но надёжно соединяющего два удалённых замка.

Глава 7: Живые дороги OSPF
На HQ-RTR:
nano /etc/frr/daemons
И сменил строку ospfd=no на ospfd=yes
systemctl enable --now frr
Войдя в интерфейс vtysh, администратор начал настраивать маршруты:
conf t
router ospf
passive-interface default
network 192.168.0.0/24 area 0
network 172.16.0.0/26 area 0
exit
interface tun1
no ip ospf network broadcast
no ip ospf passive
exit
do write memory
exit
Не забыл он и про настройку TTL для тоннеля:

bash
nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit

Глава 8: Великая книга имён
 (DNS) на HQ-SRV.
Он начал с изменения основных настроек:
nano /etc/bind/options.conf
Затем создал зоны, скопировав священные образцы:
cd /etc/bind/zone
cp localdomain au.db
cp 127.in-addr.arpa 0.db
Изменив владельцев файлов, чтобы только избранные могли вносить изменения:
chown root:named {au,0}.db
После настройки зонных файлов он перезапустил службу:
systemctl restart bind
Теперь, произнеся заклинание:
host hq-rtr.au-team.irpo

Глава 9: Создание центрального управления
Администратор начал настройку Samba AD-DC на HQ-SRV, но сначала временно отключил все интерфейсы через nmtui

Он очистил старые конфигурации:
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
Создал новые каталоги и начал процесс провижининга:
mkdir -p /var/lib/samba/sysvol
samba-tool domain provision
После настройки включил службы:
systemctl enable --now samba
systemctl enable --now bind
Когда bind отказался запускаться, администратор не растерялся. Он заглянул в конфигурационный файл, внёс необходимые изменения и перезапустил службу:
nano /etc/bind/named.conf
systemctl restart bind
Проверив статус службы, он убедился, что всё работает как надо. Затем настроил аутентификацию Kerberos:
nano /etc/krb5.conf
samba-tool domain info 127.0.0.1
kinit administrator@au-team.irpo
Глава 10: Первые подданные королевства
Пришло время создать первых пользователей. Администратор открыл волшебный инструмент admc и создал пять верных подданных:
user1.hq
user2.hq
user3.hq
user4.hq
user5.hq
Каждый получил свой уникальный пароль и права в королевстве.
