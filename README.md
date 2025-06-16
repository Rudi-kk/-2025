# -2025
Оглавление
1)	Настройте имена устройств согласно топологии. Используйте полное доменное имя	2
2)	На всех устройствах необходимо сконфигурировать IPv4	2
3)	Создание локальных учетных записей	2
4)	Настройка безопасного удаленного доступа на серверах HQ-SRV и BRSRV:	3
5)	Настройка протокола динамической конфигурации хостов.	3
6)	Между офисами HQ и BR необходимо сконфигурировать ip туннель	4
7)	Обеспечьте динамическую маршрутизацию	6
8)	Настройка Samba AD-DC	10

 




1)	Настройте имена устройств согласно топологии. Используйте полное доменное имя
З hostnamectl set-hostname <ИМЯ>; exec bash
2)	На всех устройствах необходимо сконфигурировать IPv4
Для устройств с графическим интерфейсом ПРАВОЙ КЛАВИШЕЙ ПО СЕТИ, настроить ipv4 и НЕ ЗАБЫТЬ СОХРАНИТЬ
для устройств БЕЗ графического интерфейса пользуемся nmtui
@@@@добавить скрин@@@@
адресация 
		            ens160 (b5)		
	 	            isp	 	
ens34 (bf)	 	            		    ens35 (c9)	
172.16.4.1/28		            		  172.16.5.1/28	
172.16.4.2/28		             		 172.16.5.2/28	
ens34 (ba)		              		  ens34 (54)	
hq-rtr		                   		 br-rtr	
ens35 (c4)	 		             	 ens35 (5e)	
172.16.0.1/26	 		       		  172.16.6.1/27	
ens33 (d6)	  ens34 (af)			  ens33 (b2)	  Он винда
172.16.0.2/26	172.16.0.3/28			172.16.6.2/27	172.16.6.3/27
hq-srv	      hq-cli		     		br-srv	      	br-DC

3)	Создание локальных учетных записей на HQ-SRV и BR-SRV
useradd -m -u 1010 sshuser
passwd sshuser
nano /etc/sudoers
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
ctrl+x
y
enter
 
4)	Настройка безопасного удаленного доступа на серверах HQ-SRV и BR-SRV:

на серверах 
nano /etc/mybanner
В этом НОВОМ ПУСТОМ файле пишем Authorized access only
MaxAuthTries 2
AllowUsers sshuser
ctrl+x
y
enter
nano /etc/openssh/sshd_config
находим строчки
 #port 22, раскоменчиваем и пишем port 2024
systemctl restart sshd.service
5)	Настройка протокола динамической конфигурации хостов.
Настройки проводим на HQ-RTR
nano /etc/sysconfig/dhcpd
DHCPARGS=ens35
ctrl+x
y
enter
cp /etc/dhcp/dhcpd.conf{.example,}
nano /etc/dhcp/dhcpd.conf
В этом файле должны быть строки
option domain-name “au-team.irpo”;
option domain-name-servers 172.16.0.2;

default-lease-time 6000;
max-lease-time 72000;

authoritative;
subnet 172.16.0.0 netmask 255.255.255.192 {
	range 172.16.0.3 172.16.0.8;
	option routers 172.16.0.1;
}
ctrl+x
y
enter
systemctl enable --now dhcpd
6)	Между офисами HQ и BR необходимо сконфигурировать ip туннель
Перед настройкой самого тоннеля необходимо убедиться, что на ISP включён forwarding IPv4
на ISP
nano /etc/net/sysctl.conf
и меняем строчку net ipv4 forwarding значение на 1
 
Настраиваем GRE через nmtui
BR-RTR
 
 
HQ-RTR
 
Обеспечьте динамическую маршрутизацию HQ-RTR
Отлючить файр вол systemctl disable firewalld.service --now
7)	
Настраиваем OSPF на BR-R HQ-R
nano /etc/frr/daemons
меняем строчку
ospfd=no на строчку
ospfd=yes

HQ-RTR
systemctl enable --now frr
vtysh
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
exit


nmcli connection edit tun1
set ip-tunnel.ttl 64
save
quit
systemctl restart frr

BR-RTR
Повторяем со своими адресами


Настройка DNS для офисов HQ и BR.
На HQ-SRV
nano /etc/bind/options.conf
Меняем выделенные строчки
 
systemctl enable bind --now
 
nano /etc/bind/local.conf
 

cd /etc/bind/zone
cp localdomain au.db
cp 127.in-addr.arpa 0.db
chown root:named {au,0}.db
 
nano au.db
 
Nano 0.db
 
Systemctl restart bind 
«Благослови тебя Омниссия»
Проверка host hq-rtr.au-team.irpo
Должен выдать IP

8)	Настройка Samba AD-DC
HQ-SRV
Произведём временное отключение интерфейсов. Обязательно перед началом настройки samba!
nmtui
 
grep -q 'bind-dns' /etc/bind/named.conf || echo 'include "/var/lib/samba/bind-dns/named.conf";' >> /etc/bind/named.conf

nano /etc/bind/options.conf
 

systemctl stop bind
nano /etc/sysconfig/network

 
domainname au-team.irpo
rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba
rm -rf /var/cache/samba
mkdir -p /var/lib/samba/sysvol
samba-tool domain provision
 
systemctl enable --now samba
systemctl enable --now bind #если бинд не запускается то делаем следующие шаги:
1.	nano /etc/bind/named.conf
2.	 
3.	systemctl restart bind
4.	проверяем что все работает командой systemctl status bind
5.	 

nano /etc/krb5.conf
 

samba-tool domain info 127.0.0.1
 
kinit administrator@au-team.irpo
 
Создайте 5 пользователей для офиса HQ:
Прописываем команду admc
В открывшимся окне разворачиваем au-team.irpo
Открываем вкладку users и создаем пользователей
имена пользователей формата user№.hq
HQ-CLI
Указываем DNS сервер домена
  
После этого перезагружаем систему командой reboot и пробуем войти под учетной записью administrator@au-team.irpo







МЕТОДА ИЛЮХИ




Demo2025
1. Базовая настройка системы
Изменение имени хоста

hostnamectl hostname "имя_машины"
exec bash
# НЕ ЗАБУДЬТЕ ОТКЛЮЧИТЬ  firewall  командой systemctl disable firewalld --now  (если не отключить gre и ospf могут не работать) вроде на  hq-r

2. Создание пользователя net_admin (HQ-RTR и BR-RTR)

# Создание пользователя
adduser net_admin

# Установка пароля 
passwd net_admin

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
net_admin ALL=(ALL:ALL)NOPASSWD:ALL

3. Создание пользователя sshuser (на HQ-SRV и BR-SRV)

# Создание пользователя
useradd -m -u 1010 sshuser

# Установка пароля 
passwd sshuser

# Добавление в sudoers без пароля
nano /etc/sudoers
# Добавить строку:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL

4. Настройка SSH (на HQ-SRV и BR-SRV)
Создание баннера

nano /etc/mybanner

Содержимое файла:

Authorized access only

Настройка SSH daemon

nano /etc/openssh/sshd_config

Добавить/изменить параметры:

Port 2025
Banner /etc/mybanner
MaxAuthTries 2
AllowUsers sshuser

Перезапуск службы

systemctl restart sshd.service

5. Настройка маршрутизации
На ISP - включение IP forwarding

nano /etc/net/sysctl.conf

net.ipv4.ip_forward = 1

6. Настройка GRE туннелей
На BR-RTR (Branch Router)

Сетевые настройки:

    Profile name: tun1
    Device: tun1
    Mode: GRE
    Parent: ens34
    Local IP: 172.16.5.2
    Remote IP: 172.16.4.2
    IPv4 Configuration: Manual
        IP: 192.168.0.2/24
        Gateway: 192.168.0.1

На HQ-RTR (Headquarters Router)

Сетевые настройки:

    Profile name: tun1
    Device: tun1
    Mode: GRE
    Parent: ens34
    Local IP: 172.16.4.2
    Remote IP: 172.16.5.2
    IPv4 Configuration: Manual
        IP: 192.168.0.1/24
        Gateway: 192.168.0.2

Если GRE тунель не работает пропинговать с HQ-R BR-R по его айпи, если пинги не проходят то перезаагрузить машину ISP, также если это не помогло то можно попробовать временно отключить другие сетевые интерфейсы (hqin и brin)
7. Настройка OSPF маршрутизации
На HQ-R и BR-R - активация FRR

nano /etc/frr/daemons

Изменить строку:

ospfd=yes

systemctl enable --now frr

На HQ-RTR - настройка OSPF

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

На BR-RTR - аналогичная настройка

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

Проверка OSPF

# Вход в FRR консоль
vtysh

# Проверка соседей OSPF, если ничего не вывело то забиваем и переходим к следующему заданию
show ip ospf neighbor

# Выход из консоли
exit

8. Настройка DHCP сервера (на HQ-RTR)
Указание интерфейса для DHCP

nano /etc/sysconfig/dhcpd

DHCPARGS=ens35

Создание конфигурационного файла

# в файле /etc/dhcp/dhcpd.conf.example показан пример dhcp

nano /etc/dhcp/dhcpd.conf

Содержимое файла:

option domain-name "au-team.irpo";
option domain-name-servers 172.16.0.2;
default-lease-time 6000;
max-lease-time 72000;
authoritative;

subnet 172.16.0.0 netmask 255.255.255.192 {
  range 172.16.0.3 172.16.0.8;
  option routers 172.16.0.1;
}

Запуск и автозагрузка службы

systemctl enable --now dhcpd

RAID 5 на HQ-SRV:

#команда ниже должна вывести 4 диска sda, sdb, sdc, sdd 
lsblk

#Создание raid5
sudo mdadm --zero-superblock /dev/sd[b-d]

mkfs -t ext4 /dev/md0

mkdir /etc/mdadm

echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.con

mkdir /mnt/raid5

nano /etc/fstab
#добавить эту строчку в fstab
/dev/md0  /mnt/raid5  ext4  defaults  0  0

mount -a

Проверяем монтирование:

df -h | grep /mnt/raid5

Вывод должен быть:

/dev/md0  2.0G  24K  1.9G  1%  /mnt/raid5

Настройка NFS SERVER на HQ-SRV

mkdir /mnt/raid5/nfs

chmod 766 /mnt/raid5/nfs

nano /etc/exports
#добавляем строчку ниже | ВАЖНО ВМЕСТО 172.16.0.0/28 указывайте подсеть которая будет на демо
/mnt/raid5/nfs 172.16.0.0/28(rw,no_root_squash)

exportfs -arv

systemctl enable --now nfs-server

Настройка NFS CLIENT (вроде на hq-cli смотрите по заданию кому надо монтировать)

mkdir /mnt/nfs

chmod 777 /mnt/nfs

nano /etc/fstab
#добавляем строчку где 172.16.0.2 айпи nfs сервера 
172.16.0.2:/mnt/raid5/nfs  /mnt/nfs  nfs  defaults  0  0 

mount -a

Проверяем монтирование:

df -h | grep /mnt/nfs

Вывод должен быть:

172.16.0.2:/mnt/raid5/nfs  2,0G  0  1,9G  0%  /mnt/nfs

Настройка chrony на HQ srv

Правим файл командой nano /etc/chrony.conf :

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (https://www.pool.ntp.org/join.html
#pool pool.ntp.org iburst

server 127.0.0.1 iburst prefer
hwtimestamp *
local stratum 5
allow 0/0

    image

Запускаем и добавляем в автозагрузку утилиту chronyd:

systemctl enable --now chronyd

Проверяем работоспособность (вывод должен быть как в примере)

chronyc sources

Вывод:

MS Name/IP address        Stratum  Poll  Reach  LastRx  Last  sample
=============================================================================
^/ localhost.localdomain  0        8     377    -       +0ns[  +0ns] +/-  0ns

chronyc tracking | grep Stratum

Вывод:

Stratum: 5
















Глава 1: Нареyчение имен
hostnamectl set-hostname «имя_машины»; exec bash
Серверы гордо носили имена HQ-SRV и BR-SRV, маршрутизаторы - HQ-RTR и BR-RTR, а рабочие станции скромно назывались HQ-CLI и BR-DC.
Глава 2: Раздача волшебных адресов
выбрать настройки IPv4 и аккуратно вписать нужные цифры, не забыв сохранить изменения.
nmtui:
Администратор тщательно распределил адреса между всеми жителями королевства, записав их в священный свиток:
                 			isp          
ens34 (bf)      					ens35 (c9)      
172.16.4.1/28   				172.16.5.1/28   
172.16.4.2/28   				172.16.5.2/28   
ens34 (ba)      					ens34 (54)      
hq-rtr         					br-rtr          
ens35 (c4)      					ens35 (5e)      
172.16.0.1/26   				172.16.6.1/27   
ens33 (d6)      ens34 (af)      			ens33 (b2)      (Он винда)
172.16.0.2/26   172.16.0.3/28   		172.16.6.2/27   172.16.6.3/27
hq-srv          hq-cli          			br-srv          br-DC

Глава 3: Создание верных слуг
создал пользователя sshuser, который мог бы выполнять любые команды 
useradd -m -u 1010 sshuser
passwd sshuser
дающую sshuser неограниченные права:
nano /etc/sudoers
Добавил:
sshuser ALL=(ALL:ALL)NOPASSWD:ALL
: Ctrl+X, Y, Enter. Теперь у него был верный слуга, готовый выполнять любые поручения.
Глава 4: Защитные заклинания
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
 
ДЕПАРТАМЕНТ ОБРАЗОВАНИЯ И НАУКИ ГОРОДА МОСКВЫ
ГОСУДАРСТВЕННОЕ АВТОНОМНОЕ ПРОФЕССИОНАЛЬНОЕ
ОБРАЗОВАТЕЛЬНОЕ УЧРЕЖДЕНИЕ Г. МОСКВЫ
«КОЛЛЕДЖ ПРЕДПРИНИМАТЕЛЬСТВА №11»
ЦЕНТР ИНФОРМАЦИОННО–КОММУНИКАЦИОННЫХ ТЕХНОЛОГИЙ









Отчёт по выполнению задания демонстрационного экзамена 
специальности 09.02.06 «Сетевое и системное администрирование»
КОД 09.02.06-3-2025









Выполнил студент гр. С-41
Печенкин Тимофей Владимирович












Москва 2025
Задания:
1.	Расчет IP-адресации
2.	Выбор и создание туннеля
3.	Выбор технологии динамической маршрутизации и её настройка
4.	Настройка динамической адресации
5.	Создание  и настройка файлового хранилища
6.	Настройка moodle
7.	Установка браузера
8.	Настройка туннеля до уровня обеспечивающего шифрование трафика
9.	Выбор системы мониторинга и настройка этой системы 

1.	 Расчет IP-адресации
В таблице показано, какие адреса закреплены за конкретными устройствами.
Имя устройства	IP-адрес	Шлюз по умолчанию
ISP	172.16.4.1/28 
172.16.5.1/28 
	
HQ-RTR	172.16.4.2/28 
172.16.0.1/26
	172.16.4.1
BR-RTR	172.16.5.2/28
172.16.6.1/27
	172.16.5.1
HQ-SRV	172.16.0.2/26
	172.16.0.1
HQ-CLI	172.16.0.3/28
	
BR-SRV	172.16.6.2/27
	
BR-DC	172.16.6.3/27	

2.	Выбор и создание туннеля
Для организации соединения между BR-RTR и HQ-RTR предпочтение отдали протоколу GRE вместо IP-in-IP благодаря его расширенному функционалу. Основные причины выбора GRE включают:
1.	Поддержка широковещательного трафика — GRE позволяет инкапсулировать широковещательные и multicast-пакеты, что критично для работы некоторых сетевых протоколов.
2.	Универсальная совместимость — оборудование и ОС, не поддерживающие обработку IP-in-IP, как правило, корректно взаимодействуют с GRE-туннелями.
3.	Механизмы защиты — GRE предоставляет опцию аутентификации заголовков туннеля, снижая риски несанкционированного доступа.
Эти особенности делают GRE более гибким и безопасным решением для построения защищенных туннелей в гетерогенных сетевых средах.
Настройка GRE на BR-RTR
 

Настройка GRE на HQ-RTR
 

3.	Выбор технологии динамической маршрутизации и её настройка
Протокол OSPF выбран в качестве основного решения, исходя из ключевых требований:
Высокая скорость конвергенции — быстрое формирование маршрутных таблиц при старте или изменении топологии.
Нативная совместимость с Alt Linux — полная поддержка на уровне ОС, включая инструменты управления и мониторинга.
Адаптивность к изменениям — автоматическая корректировка маршрутов при расширении сети или обновлении оборудования.
Настройка протокола OSPF на BR-RTR
 
Настройка протокола OSPF на HQ-RTR
 
4.	Настройка динамической адресации
Настройка протокола DHCP на HQ-RTR
 
5.	Создание и настройка файлового хранилища
На сервере HQ-SRV реализован массив RAID 5 уровня, объединяющий три накопителя емкостью по 1 ГБ каждый. Выбор данной конфигурации обусловлен оптимальным сочетанием производительности и отказоустойчивости — технология RAID 5 обеспечивает защиту данных за счет распределенной чётности, сохраняя высокую скорость операций чтения. Для упрощения работы с массивом выполнена настройка автоматического подключения в системную директорию /raid5, что гарантирует бесперебойный доступ к хранилищу при перезагрузках.
 
6.	Настройка moodle
На сервере HQ-SRV развернута платформа Moodle для управления образовательным процессом, интегрированная с СУБД MariaDB. Конфигурация включает:
o	База данных: moodledb
o	Учетные записи: Пользователи: moodle (для работы системы), admin (административный доступ)

Аутентификация: пароль P@ssw0rd

Такая связка обеспечивает стабильную работу Moodle с поддержкой транзакций, резервного копирования и управления правами доступа через MariaDB.
7.	Установка браузера
Для установки браузеры был выбран Yandex браузер так как он соответствует требованиям задания
 
 
8.	Настройка туннеля до уровня обеспечивающего шифрование трафика
Для повышения защищенности соединения между серверами HQ-SRV и BR-SRV базовый IP-туннель был усовершенствован. Внедрение протокола IPsec обеспечило сквозное шифрование трафика с использованием алгоритма AES-256, обеспечивающего криптостойкость за счет 256-битных ключей. Аутентификация реализована через Pre-Shared Key (PSK) — метод, упрощающий развертывание, но менее надежный по сравнению с сертификатной аутентификацией.
Ключевые изменения:
Переход от незащищенного туннеля к шифрованному каналу передачи данных;
Оптимальный баланс между безопасностью (AES-256) и простотой конфигурации (PSK);
Совместимость с существующей инфраструктурой без необходимости внедрения PKI.

Данный подход минимизирует риски перехвата данных, сохраняя при этом умеренные требования к ресурсам настройки.
9.	Выбор системы мониторинга и настройка этой системы 
В качестве решения для мониторинга выбран Zabbix, что обусловлено следующими факторами:  

1. Адаптивность и масштабируемость — гибкая настройка под задачи инфраструктуры и возможность расширения функционала по мере роста сети.  
2. Готовые инструменты для мониторинга — предустановленные шаблоны для отслеживания Windows, Linux, сетевого оборудования и IoT-устройств.  
3. Многообразие оповещений — поддержка email, SMS, мессенджеров (Telegram, Slack) и интеграция с системами инцидент-менеджмента.  
4. Открытая лицензия — доступ к исходному коду позволяет кастомизировать систему, проводить аудит безопасности и снижать зависимость от вендоров.  

Данные преимущества делают Zabbix универсальным выбором для комплексного мониторинга гетерогенных сред с требованиями к гибкости и прозрачности решений.

