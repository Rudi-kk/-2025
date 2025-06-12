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
ens34 (bf)	 	                ens35 (c9)	
172.16.4.1/28		              172.16.5.1/28	
172.16.4.2/28		              172.16.5.2/28	
ens34 (ba)		                ens34 (54)	
hq-rtr		                    br-rtr	
ens35 (c4)	 		              ens35 (5e)	
172.16.0.1/26	 		            172.16.6.1/27	
ens33 (d6)	  ens34 (af)		  ens33 (b2)	  Он винда
172.16.0.2/26	172.16.0.3/28		172.16.6.2/27	172.16.6.3/27
hq-srv	      hq-cli		      br-srv	      br-DC

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

