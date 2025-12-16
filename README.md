# Wpisy Blogowe

Eksport z 50 wpisów

---

## Aktualizacja WordPress bez FTP

**ID:** 47 | **Typ:** post

1. Dodanie na koniec pliku wp-config.php jednej linii kodu:
`define('FS_METHOD','direct');`

2. Zmiana właściciela katalogu wp-content:
`chown -R www-data:www-data wp-content`

---

## Zmiana klawiszy klawiatury w KDE

**ID:** 117 | **Typ:** post

Nie ma to jak rozwalony tabulator w Linuksie... Oto plaster zanim wymienimy klawiaturę.

1. Wygenerowanie oryginalnego pliku z znakami klawiatury:
`setxkbmap -print | xkbcomp -xkb -o original.xkb -`

2. Dla bezpieczeństwa będziemy edytować innym plik:
`cp original.xkb original-right-ctrl-is-tab.xkb`

3. Zmiany w pliku original-right-ctrl-is-tab.xkb:
- z ` = 23;` na ` = 105;`
- z ` = 105;` na ` = 23;`
4. Stworzenie skryptu change-keys.sh o treści:
` #!/bin/bash
xkbcomp original-right-ctrl-is-tab.xkb $DISPLAY > /dev/null 2>&1`

5. Nadanie mu uprawnień do wykonania:
`chmod 775 change-keys.sh`

6. Wgrywanie przy starcie KDE dla użytkownika test
`ln -s change-keys.sh /home/test/.config/autostart-scripts/.`

Wgrywanie oryginalnych klawiszy:
`xkbcomp original.xkb $DISPLAY`

---

## Let's Encrypt (certbot)

**ID:** 143 | **Typ:** post

Darmowy SSL (https): (https://letsencrypt.org/)

Instalacja na Debianie 8(jessie). Dla innej wersji instalacja może wyglądać inaczej. Zaleca się korzystanie z manuala z (https://letsencrypt.org/):

1. Dodanie repozytorium
`sudo sh -c "echo 'deb http://ftp.debian.org/debian jessie-backports main' >> /etc/apt/sources.list"`

2. Aktualizacja
`sudo apt-get update`

3. Instalacja certbota
`sudo apt-get install python-certbot-apache -t jessie-backports`

4. Utworzenie certyfikatu
`sudo certbot certonly --manual`
Normalnie można iść zgodnie z manualem ze (https://letsencrypt.org/), ale ja miałem problem z certyfikatem ze strony dostawcy domeny. Dlatego wygenerowałem tylko certyfikat oraz tak skonfigurowałem apache żeby pokazał na chwilę stronę o którą prosił skrypt.

5. Instalacja certyfikatu w apache
`sudo certbot --apache`
Jak kto woli, ale ja ustawiłem apache żeby korzystał z https na każdej stronie.

6. Odnawianie certyfikatu
~~`certbot -q renew`
Dodałem opcje "-q" żeby certbot nie spamił mailami.~~
Oczywiście musiało coś pójść nie tak. Podczas próby odnowienia certyfikatu wywala błąd `"The manual plugin is not working"` spowodowany tym że wcześniej certyfikat był utworzony przez `--manual`. Poczytałem trochę `--helpa` i udało się wyczarować coś takiego:
Plik /scripts/renew_certbot.sh:
`#!/bin/bash
service apache2 stop
certbot certonly -d www.bujakiewiczpawel.pl --standalone -n -q
service apache2 start`

Oraz zmodyfikowałem wcześniejszy wpis w cronie na `00 20 * * * root /scripts/renew_certbot.sh`

---

## Resetowanie hasła root

**ID:** 152 | **Typ:** post

1. restart

2. Rozpoczęcie edycji pierwszego wpisu GRUB przez wciśnięcie przycisku "e" (na dole są podpowiedzi jeśli to nie "e").

3. Linia do której będziemy coś dopisywać zaczyna się `kernel` i kończy `ro quiet` może trochę się różnić.

4. Dopisujemy do tej linii na koniec `rw init=/bin/bash`. Zmiany po restarcie nie będą zapisane.

5. Wciskamy ctrl + x (na dole są podpowiedzi jeśli to nie ctrl + x) żeby uruchomić system z naszym wpisem.

6. Po zalogowaniu systemu, resetujemy hasło komenda `passwd`

7. Wywołanie restart komenda `/sbin/reboot -f` lub `echo 1 > /proc/sys/kernel/sysrq && echo b > /proc/sysrq-trigger`

---

## Konfiguracja VPN (OpenVPN)

**ID:** 171 | **Typ:** post

Źródła z których korzystałem:
- (https://sekurak.pl/praktyczna-implementacja-sieci-vpn-na-przykladzie-openvpn/)
- (https://forums.openvpn.net/viewtopic.php?t=11010)
# **1. Własne centrum certyfikacji**

I. Instalacja i organizacja CA:
`sudo aptitude install openvpn
sudo cp -a /usr/share/easy-rsa/ /etc/CA`

II. Edycja pliku **/etc/CA/vars**:
`# These are the default values for fields
# which will be placed in the certificate.
# Don't leave any of these fields blank.
export KEY_COUNTRY="PL"
export KEY_PROVINCE="Pomorskie"
export KEY_CITY="Gdansk"
export KEY_ORG="bujakiewiczpawel@gmail.com"
export KEY_EMAIL="bujakiewiczpawel@gmail.com"
export KEY_OU="Centrum Certyfikacji"`

III. Tworzenie CA:
`cd /etc/CA
sudo source ./vars
sudo ./build-ca
sudo ./build-dh
sudo openvpn --genkey --secret ta.key`
Jeśli skryptom brakuję -x do wykonanie, trzeba dodać.

Zawartość katalogu **/etc/CA/keys/**:
**ca.crt** – certyfikat naszego CA (chmod 644)
**ca.key** – plik z kluczem prywatnym naszego CA (chmod 600)
**dh2048.pem** – klucz algorytmu Diffie-Hellman (chmod 600)

W katalogu głównym **/etc/CA/**:
**ta.key** - dzięki temu komputer, który nie posiada pliku ta.key nie jest w stanie nawet spróbować nawiązać połączenia (chmod 600)

IV. Utworzenie certyfikatu i klucza serwera OpenVPN:
`sudo ./ build-key-server vpn.bujakiewiczpawel.pl`
Hasło tutaj pozostawiamy puste.
Wynikiem polecenia będą pliki **vpn.bujakiewiczpawel.pl.key** i **vpn.bujakiewiczpawel.pl.crt**

V. Utworzenie certyfikatu i klucza klienta OpenVPN:
`sudo ./build-key-pass siwy`
Tutaj wpisujemy hasło.

# **2. Konfiguracja klienta**

 I. Przygotowanie paczki dla klienta (Windows)
Potrzebne pliki wcześnie wygenerowane:
- ca.crt
- siwy.crt
- siwy.key
- ta.key
- vpn.bujakiewiczpawel.pl.crt
- oraz nowy plik openvpn.ovpn, który zawiera:
` # Nazwa naszego klucza i certyfikatu
cert siwy.crt
key siwy.key
remote vpn.bujakiewiczpawel.pl 1194 # Adres IP i port docelowy
client
fragment 1000
dev tun
proto udp
resolv-retry infinite
nobind
user nobody
group nogroup
persist-key
persist-tun
ca ca.crt
ns-cert-type server # Upewniamy sie ze laczymy sie z serwerem
tls-auth ta.key 1
tls-version-min 1.2
cipher AES-256-CBC
verb 3`

 II. Sprawdzanie połączenia z serwerem
Testowanie pingu w CMD:
`ping -t vpn.bujakiewiczpawel.pl`
Jeśli brakuję połączenia trzeba dokonać zmian w DNS lub zmienić konfiguracje żeby działała po IP serwera.

# **4. Dostęp do sieci lokalnej z przeniesieniem całego ruchu przez VPN**

 I Konfiguracja serwera
Na serwerze tworzymy plik /etc/openvpn/openvpn.conf o treści:
`port 1194 # Port na którym nasluchujemy
proto udp
dev tun # Rodzaj tunelu
mssfix 1000 # Wartosci dobrane eksperymentalnie 
fragment 1000 # Wartosci dobrane eksperymentalnie
keepalive 10 120 # Czestotliwosc pakietow keepalive
ca /etc/CA/keys/ca.crt
cert /etc/CA/keys/vpn.bujakiewiczpawel.pl.crt
key /etc/CA/keys/vpn.bujakiewiczpawel.pl.key
dh /etc/CA/keys/dh2048.pem
tls-auth /etc/CA/ta.key 0
tls-version-min 1.2
cipher AES-256-CBC
server 192.168.1.0 255.255.255.0 # Adresy IP klientow vpn
ifconfig-pool-persist ipp.txt
push "route 0.0.0.0 0.0.0.0" # przekierowanie całego ruchu klienta
push "redirect-gateway" # przekierowanie całego ruchu klienta
client-config-dir /etc/openvpn/konta_vpn # Katalog ustawien klientow
ccd-exclusive # Dopuszczamy tylko ZNANYCH klientow
`

 II. Testowanie konfiguracji serwera:
`sudo openvpn /etc/openvpn/openvpn.conf`
Jeśli coś jest nie tak, dokonywać zmian zgodnie z podpowiedziami wyniku komendy.
Serwer openvpn nie musi być włączony żeby połączyć się z nim.

 III. Dodanie NATu
`sudo iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -j SNAT --to-source 192.168.0.3`
192.168.1.0/24 - IP przydzielone dla klientów VPN
192.168.0.3 - IP serwera VPN
Backup iptables i wgrywanie polecam tak jak w tym artykule (https://major.io/2009/11/16/automatically-loading-iptables-on-debianubuntu/).

# **5. Instalacja i konfiguracja klienta VPN (Windows)**

 I. Pobranie klienta: (https://openvpn.net/index.php/open-source/downloads.html)

 II. Konfiguracja klienta:
Po zainstalowaniu aplikacji w katalogu **C:\Program Files\OpenVPN\** umieszczamy wszystkie pliku wymienione w punkcie **2.1**.
W trayu klikamy PPM na nową ikonę i wybieramy połącz.
Podajemy hasło i połączenie gotowe.
Można sprawdzić jakie teraz mamy IP zewnętrzne.

---

## Naprawa konfiguracji startowej (Boot Repair Disk)

**ID:** 197 | **Typ:** post

Źródła z których korzystałem:
- (http://linuxmagazine.pl/index.php/issues/162)
- (https://sourceforge.net/p/boot-repair-cd/home/Home/)
# **Boot Repair Disk**

Narzędzie ma ogromne możliwości. Osobiście skorzystałem z 2, mianowicie usunięcie systemu operacyjnego oraz zastąpienie GRUBa na MBR. 

Obraz płyty można ściągnąć z (https://sourceforge.net/projects/boot-repair-cd/files/).

Zalecane jest stworzenie live-USB za pomocą (http://rufus.akeo.ie/?locale=en) lub (http://unetbootin.sourceforge.net/). U mnie zadziałał jedynie Rufus.

Warto poczytać o możliwościach jakie daję i zapamiętać nazwę bo nie raz na pewno wam uratuję sytuacje jak coś pójdzie nie tak.

---

## Lokalne repozytorium ISO (local ISO image storage repository)

**ID:** 216 | **Typ:** post

Source:
- (https://linuxconfig.org/how-to-add-iso-image-storage-repository-on-xenserver-7-linux)

I. Create a store directory:
`mkdir -p /var/opt/ISO_IMAGES
xe sr-create name-label=ISO_IMAGES_LOCAL type=iso device-config:location=/var/opt/ISO_IMAGES device-config:legacy_mode=true content-type=iso`
The output of last command is UUID you need. If you dont copy UUID, you can call command `xe sr-list` to find UUID. UUID looks like **c395d188-3c1e-c334-748b-dd699420a3b1**.

II. Auto refresh store:
`cd /root/
touch refresh-ISO_IMAGES.sh
chmod 770 refresh-ISO_IMAGES.sh`

Edit file **refresh-ISO_IMAGES.sh** and enter this code:
`#!/bin/bash
xe sr-scan uuid=**c395d188-3c1e-c334-748b-dd699420a3b1**`

 Add to Cron:
Edit file **/etc/crontab** and add this line on end of file:
`*/5 * * * * root /root/refresh-ISO_IMAGES.sh`
This make refresh every 5 minutes.

III. Send ISO files to storage:
Use a regular FTP/SFTP client like **FileZilla** or **WinSCP**.

---

## nmap - port scanning

**ID:** 229 | **Typ:** post

`nmap -sV --script ssl-enum-ciphers www.bujakiewiczpawel.pl`

---

## netstat - listening ports

**ID:** 231 | **Typ:** post

`netstat -tlp
netstat -tlpn`

---

## CRON - example

**ID:** 233 | **Typ:** post

minute 0-59
hour 0-23
day of month 1-31
month 1-12 (or names)
day of week 0-7 (0 or 7 is Sunday or use names)

Every 5 min
`*/5 * * * * root /p/a/t/h/aaaa.sh`

Every hour, from 9:00 to 18:00
`00 09-18 * * * root /p/a/t/h/aaaa.sh`

Every day, at 9:00 and 18:00
`00 09,18 * * * root /p/a/t/h/aaaa.sh`

Every day, at 17:30
`30 17 * * * root /p/a/t/h/aaaa.sh`

Every weekday, at 17:30
`30 17 * * Mon,Tue,Wed,Thu,Fri root /p/a/t/h/aaaa.sh`

---

## Own DNS (PowerDNS + PowerAdmin)

**ID:** 238 | **Typ:** post

Source:
- (https://zaufanatrzeciastrona.pl/post/wlasny-serwer-dns-to-nie-takie-straszne-ani-trudne-jak-sie-wydaje/)
- (http://blog.crazypi.com/setup-dns-server-powerdns-raspberry-pi/)

I. Install Apache2:
`sudo aptitude install apache2`

II. Install MySQL, PHP and secure:
`sudo aptitude install mysql-server
sudo mysql_install_db
sudo mysql_secure_installation
sudo aptitude install mysql-client
sudo aptitude install libapache2-mod-php5 php5 php5-common php5-curl php5-dev php5-gd php-pear php5-imap php5-mcrypt php5-mhash php5-ming php5-mysql php5-xmlrpc gettext
sudo /etc/init.d/mysql restart
sudo /etc/init.d/apache2 restart`

III. Install PowerDNS:
`sudo aptitude install pdns-server pdns-backend-mysql`

IV. Install PowerAdmin:
`cd /var/www/html
sudo wget https://github.com/poweradmin/poweradmin/archive/master.zip
sudo unzip master.zip
sudo mv poweradmin-master/* .
sudo rm master.zip
sudo rm -f poweradmin-master/`

V. Configure PowerAdmin:
Enter IP to browser:
- Choose language.
- Go next.
- Enter credentials to MySQL and password to admin user PowerAdmin.
- Enter user credentials. This user is the communication between WWW and MySQL.
- Do this what is in description.
- Do this what is in description.

VI. Cleaning:
`sudo rm -r /var/www/html/PowerAdmin/install/`

VII. Configuring a recursor:
`sudo sed -i ‘s/# recursor=/recursor=**8.8.8.8**/g’ /etc/powerdns/pdns.conf
sudo sed -i ‘s/allow-recursion=127.0.0.1/allow-recursion=127.0.0.1,**192.168.0.0\/24**/g’ /etc/powerdns/pdns.conf`

---

## AOMEI partition assistant (partitioning)

**ID:** 247 | **Typ:** post

Source:
- (https://www.aomeitech.com/pa/standard.html)

Very useful program to partitioning and cloning OS to other HDDs. Program is designet for **Windows**.

---

## Refreshing commands (watch)

**ID:** 268 | **Typ:** post

`watch -n 5 cat /proc/test`

---

## Install old version GitLab on Debian.

**ID:** 271 | **Typ:** post

Source:
- (https://about.gitlab.com/installation/#debian)
- (https://packages.gitlab.com/gitlab)
 

I. Creating a backup
`sudo gitlab-rake gitlab:backup:create`
 

II. Install and restore backup:
` sudo aptitude install -y curl openssh-server ca-certificates
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
sudo aptitude install gitlab-ee=**10.7.3-ee.0**
sudo mv **1526756133_2018_05_19_10.7.3-ee_gitlab_backup.tar** /var/opt/gitlab/backups/.
sudo gitlab-ctl stop unicorn
sudo gitlab-ctl stop sidekiq
sudo gitlab-rake gitlab:backup:restore BACKUP=**1526756133_2018_05_19_10.7.3-ee**
sudo gitlab-ctl restart
sudo aptitude upgrade`

---

## Test GitLab commands

**ID:** 275 | **Typ:** post

`sudo gitlab-rake gitlab:check --trace
sudo gitlab-rake db:migrate:status --trace
gitlab-ctl tail`

---

## Change hostname permanent

**ID:** 277 | **Typ:** post

`sudo hostnamectl set-hostname new.name.1`
and change old hostname to new in /etc/hosts.

---

## Migration VM from XenServer to ProxMox (*.xva to *.raw)

**ID:** 280 | **Typ:** post

Source:

- (https://techblog.jeppson.org)
- XenServer 7.1
- xcp-ng 8.2
- ProxMox 5.2
- ProxMox 6.3

 

I. Export VM to *.XVA in XenServer.

II. Create new VM in ProxMox with parameters:

- CPU - like VM on XenServer
- RAM - like VM on XenServer
- HDD-1 - size 120GB. Temp system (20GB) and size of VM from XenServer (100GB). This drive will be removed.
- HDD-2 - size 100GB. Size of VM from XenServer

 

III. Install temp system and cmake3 on HDD-1. 

 

IV. Send *.xva file to new VM on HDD-1. I use scp (linux) or WinSCP (Windows).

 

V. Install xva-img from github:

```
wget https://github.com/eriklax/xva-img/archive/master.zip
```

If you have some errors with installation like:

- ... noexcept(false);..., use cmake like this:

cmake -DCMAKE_CXX_STANDARD=11 -DCMAKE_CXX_STANDARD_REQUIRED=ON .

- CMake Error: your CXX compiler: "/usr/bin/c++" was not found. Please set **CMAKE_CXX_COMPILER** to a valid compiler path or name.
- xva-img-master/src/sha1.cpp:20:25: fatal error: openssl/sha.h: No such file or directory #include Install packages:
```
aptitude install build-essential aptitude install libssl-devaptitude install libxxhash-dev
```

 

VI. Extract *.xva (change **vm.xva** to your file name *.xva):

```
mkdir my-virtual-machine tar --warning=no-timestamp -xf vm.xva -C my-virtual-machine chmod -R 755 my-virtual-machine
```

Extract *.xva can take a while.

 

V. Convert to *.raw:

**Note:** When you extract your VM tar creates subfolders for each hard disk attached to the VM. You will have to run this command for each Ref folder that was generated as part of the image extraction process.

```
xva-img -p disk-export my-virtual-machine/Ref\:1/ disk-1.raw
```

The *.raw file size is the same like HDD VM.

 

VI. Transfer files to HDD-2: Name of HDD-2 you can check with **fdisk or lsblk**. In this case is **/dev/sdb**. You don't need make file system or partition. The dd do everything.

```
sudo dd if=disk-1.raw of=/dev/sdb bs=64M
```

 

VII. Turn On VM with HDD-2: In ProxMox:

- Power Off VM.
- Detach HDD-1.
- I deleted HDD-1 with temporary system.
- In settings change boot order to start from HDD-2.
- Turn On the VM.

 

VIII. XenServer tool: If you have tool from XenServer, you can remove package with this command:

```
dpkg -P xe-guest-utilities
```

 

IX. Sometimes you need to update grub.

---

## Debian IP configuration

**ID:** 328 | **Typ:** post

Ip configuration example:

```
allow-hotplug eth0 iface eth0 inet dhcp auto eth1 iface eth1 inet static address 192.168.0.4 netmask 255.255.255.0 gateway 192.168.0.1 dns-nameservers 192.168.0.4
```

---

## Create template for virtualization

**ID:** 330 | **Typ:** post

I recommended changes before make template:

- comment out CD from /etc/apt/sources.list
- install aptitude
- install mc
- install sudo and add user to group
- set static and DHCP IP - comment out static IP
- install Guest Agent

---

## TCP and UDP (protocols)

**ID:** 339 | **Typ:** post

Source:
- (https://www.diffen.com/difference/TCP_vs_UDP)
TCP:
- HTTP - 80
- HTTPs - 443
- FTP - 20 and 21
- SMTP - 25
- Telnet - 23
UDP:
- DNS - 53
- DHCP - 67 and 68
- TFTP - 69
- SNMP - 161
- RIP - 520
- VOIP

---

## Wordpress blog syntax

**ID:** 393 | **Typ:** post

`
Source:
- (https://www.youtube.com/watch?v=jSnvgBvlweY)
`
 

Syntax:
- Enter - ` `
- List:
- text
`List:
- text`
 
 Chars:
- - `>`
- / - `/`
- ] - `]`
- (https://www.google.com): (https://www.google.com)

---

## Initialize Docker Swarm in CentOS (Error response from daemon: rpc error: code = Unavailable desc = grpc: the connection is unavailable)

**ID:** 496 | **Typ:** post

Source:
- training.play-with-docker.com
- forums.docker.com

 

1. Initializing Docker Swarm on **1** node:
```
sudo docker swarm init --advertise-addr 192.168.0.1
```

 

2. Try add the worker on **2** node:
```
sudo docker swarm join --token SWMTKN-1-3b33jjwsqpkcy2c8og73aorjf2ao9sjm4crvbwg3xpd1ome459-ckfdcxqqahb9gy9s2t9n5mi78 192.168.0.1:2377
```

I have error:`Error response from daemon: rpc error: code = Unavailable desc = grpc: the connection is unavailable`

 

3. Solution:
Open ports in firewalld in all nodes:
```
sudo firewall-cmd --add-port=2376/tcp --permanent sudo firewall-cmd --add-port=2377/tcp --permanent sudo firewall-cmd --add-port=7946/tcp --permanent sudo firewall-cmd --add-port=7946/udp --permanent sudo firewall-cmd --add-port=4789/udp --permanent
```
In my case I reset nodes, but maybe reset or reload firewalld do this the same.

---

## How to fix slow LAN upload speed in Windows 10

**ID:** 505 | **Typ:** post

Source:

- (https://www.youtube.com/watch?v=jSnvgBvlweY)

 

In advanced options the ethernet adapter, disabled options:

- Large Send Offload (IPv4)
- Large Send Offload v2 (IPv4)
- Large Send Offload v2 (IPv6)

---

## Manual change of URL in WordPress

**ID:** 512 | **Typ:** post

First login to database.

 

Check what is the current URL:
```
select * from wp_options where option_name = 'home' OR option_name = 'siteurl';
```

 

SQL to change URL:
```
UPDATE wp_options SET option_value = replace(option_value, '**https**://www.old.pl', '**http**://new.home.pl') WHERE option_name = 'home' OR option_name = 'siteurl'; UPDATE wp_posts SET guid = replace(guid, '**https**://www.old.pl','**http**://new.home.pl'); UPDATE wp_posts SET post_content = replace(post_content, '**https**://www.old.pl', '**http**://new.home.pl'); UPDATE wp_postmeta SET meta_value = replace(meta_value,'**https**://www.old.pl','**http**://new.home.pl');
```

 

In this case, I change https url to http. Remember to use correct protocol.

---

## knife bootstrap - Missing Cookbooks: roles

**ID:** 517 | **Typ:** post

Error when calling the knife bootstrap:

`Missing Cookbooks:
------------------
The following cookbooks are required by the client but don't exist on the server:
* roles`

 

Solution:

#### Call knife bootstrap from roles folder.

---

## Encrypted zip with hidden files list without compression

**ID:** 526 | **Typ:** post

### 7z a -p -t7z -mx0 -mhe=on file.7z /path/dir

 

**a** - Add files to archive
**-p** - Set Password
**-t7z** - Set type of archive
**-mx0** - Set compression Method
- **0** - NON
- **1** - Fastest
- **3** - Fast
- **5** - Normal
- **7** - Maximum
- **9** - Ultra
**-mhe=on** - Hide files list

---

## Installation Bacula 9 and Baculum on CentOS 7

**ID:** 536 | **Typ:** post

Source:
- (http://www.bacula.pl/artykul/39/podstawowa-instalacja-i-konfiguracja-oprogramowania-bacula/)
- (https://www.bacula.org/9.4.x-manuals/en/main/main.pdf)
- (http://bacula.us/compilation/)
- (http://bacula.us/baculum/)
 
## This tutorial install DIR and SD on one machine.
 

1. Download bacula tar from (https://sourceforge.net/projects/bacula/):
```
bacula-9.4.1.tar.gz
```
and sent tar to server
 

2. Install PostgreSQL:
```
sudo yum install postgresql-server postgresql-contrib postgresql-devel sudo postgresql-setup initdb sudo systemctl start postgresql sudo systemctl enable postgresql
```
 

3. Configuration firewalld:
```
sudo firewall-cmd --permanent --zone=public --add-port=9101-9103/tcp sudo firewall-cmd --reload
```
 

4. Disable SELinux:
```
sudo nano -c /etc/selinux/config
```
change to "SELINUX=disabled" and restart the server
```
sudo reboot
```
 

5. Install bacula (DIR, SD and FD):
```
tar -xvzf bacula-9.4.1.tar.gz cd bacula-9.4.1 sudo yum install gcc gcc-c++ libacl-devel lzo-devel mt-st mtx openssl-devel readline-devel zlib-devel sudo ./configure --prefix=/usr/local/bacula --with-scriptdir=/usr/local/bacula/scripts --with-postgresql=/usr sudo make sudo make install sudo make install-autostart sudo chmod o+rx /usr/local/bacula/scripts/create_postgresql_database /usr/local/bacula/scripts/make_postgresql_tables /usr/local/bacula/scripts/grant_postgresql_privileges sudo -u postgres /usr/local/bacula/scripts/create_postgresql_database sudo -u postgres /usr/local/bacula/scripts/make_postgresql_tables sudo -u postgres /usr/local/bacula/scripts/grant_postgresql_privileges sudo chmod o-rx /usr/local/bacula/scripts/create_postgresql_database /usr/local/bacula/scripts/make_postgresql_tables /usr/local/bacula/scripts/grant_postgresql_privileges sudo su postgres psql -d bacula alter user bacula with password 'password';
```
and
```
sudo nano -c /var/lib/pgsql/data/pg_hba.conf
```
change all "peer" and "ident" to "trust" (This option is very low security.) and
```
sudo systemctl restart postgresql.service
```
 

6. Configuration DIR, SD and FD:
- (http://www.bacula.pl/artykul/39/podstawowa-instalacja-i-konfiguracja-oprogramowania-bacula/)
- (https://www.bacula.org/9.4.x-manuals/en/main/main.pdf)
You must do the rest yourself. Bacula configuration is not easy.
 

7. Install baculum:
(http://bacula.us/baculum/)
 
```
rpm --import http://bacula.org/downloads/baculum/baculum.pub echo " name=Baculum CentOS repository baseurl=http://bacula.org/downloads/baculum/stable/centos gpgcheck=1 enabled=1 name=Baculum Fedora repository baseurl=http://bacula.org/downloads/baculum/stable/fedora gpgcheck=1 enabled=1" > /etc/yum.repos.d/baculum.repo yum install -y baculum-common baculum-api baculum-api-httpd baculum-web baculum-web-httpd echo "Defaults:apache "'!'"requiretty apache ALL=NOPASSWD: /usr/sbin/bconsole apache ALL=NOPASSWD: /usr/sbin/bdirjson apache ALL=NOPASSWD: /usr/sbin/bsdjson apache ALL=NOPASSWD: /usr/sbin/bfdjson apache ALL=NOPASSWD: /usr/sbin/bbconsjson" > /etc/sudoers.d/baculum chown -R apache /opt/bacula/etc firewall-cmd --permanent --zone=public --add-port=9095-9096/tcp firewall-cmd --reload systemctl enable httpd.service systemctl restart httpd.service # Use the Internet browser to access and configure Baculum API http://localhost:9096/ and then Baculum http://localhost:9095/ # (replace localhost for the actual IP address if necessary)
```
 

First configure the API through the http://localhost:9096/ URL (admin, admin). You can test each of the settings you have made. An API access credential (user and password or oauth) will be defined.
 

Then, access the Baculum interface (http://localhost:9095/ - admin, admin) and also configure the language, access to the Bacula database, the Baculum API and Baculum Interface credential.

---

## Bacula-client installation (CentOS 7 and Debian 9)

**ID:** 559 | **Typ:** post

1. Installation dependence:
CentOS 7
```
sudo yum install gcc gcc-c++ libacl-devel lzo-devel mt-st mtx openssl-devel readline-devel zlib-devel
```
`Debian 9`
```
sudo apt install build-essential
```
 

2. Installation bacula-client:
Download bacula tar from (https://sourceforge.net/projects/bacula/):
```
bacula-9.4.1.tar.gz
```
and sent tar to server
 
CentOS 7 an`d Debian 9`
```
tar -xvzf bacula-gui-9.4.1.tar.gz cd bacula-9.4.1/ sudo ./configure --prefix=/usr/local/bacula \ --disable-build-dird \ --disable-build-stored \ --enable-client-only \ --with-scriptdir=/usr/local/bacula/scripts sudo make sudo make install
```
 

3. Configuration bacula-client and system:
CentOS 7
```
sudo firewall-cmd --permanent --zone=public --add-port=9102/tcp sudo firewall-cmd --reload sudo make install-autostart
```
`Debian 9`
```
sudo iptables -A INPUT -p tcp --dport 9102 --jump ACCEPT sudo apt-get install iptables-persistent sudo make install-autostart
```
 

4. Add your domein to search in nmtui:
CentOS 7
```
sudo nmtui sudo systemctl restart network.service
```
`Debian 9`
```
sudo nano -c /etc/resolv.conf sudo service networking restart
```
 

5.Edit config file:
CentOS 7 an`d Debian 9`
```
sudo nano -c /usr/local/bacula/etc/bacula-fd.conf
```
```
# bacula serwer Director { Name = vm-l-bacula-1-dir Password = "dir-password" } # bacula serwer Director { Name = vm-l-bacula-1-mon Password = "mon-password" Monitor = yes } # client serwer FileDaemon { Name = vm-l-server-1-fd FDport = 9102 WorkingDirectory = /opt/bacula/working Pid Directory = /var/run Maximum Concurrent Jobs = 2 Plugin Directory = /usr/local/bacula/lib } # bacula serwer # Send all messages except skipped files back to Director Messages { Name = Standard director = vm-l-bacula-1-dir = all, !skipped, !restored }
```
 

6. Start bacula-client:
CentOS 7 an`d Debian 9`
```
sudo systemctl start bacula-fd.service
```
 

7. Add new client in Bacula Web Interface:
(http://127.0.0.1:9095/)

---

## Remove volume from Bacula

**ID:** 580 | **Typ:** post

Source:
- dan.langille.org
 
1. In bconsole call:
```
delete yes volume=VOLUME-NAME
```
 
Removing the Volumes from disk.
2. In shell call:
```
rm VOLUME-NAME
```

---

## Use bpipe with Bacula

**ID:** 583 | **Typ:** post

Source:
- (http://bacula.us/proxmox-vm-backup-with-bacula-bpipe-plugin/)
- (http://bacula.us/using-bpipe-to-stream-dumps-vm-clones-and-another-data-to-your-backup/)
 
### I use bpipe to backup ProxMox VM. The *.vma files is transfer to Bacula and save disk space on hypervisor.
 
The plugin bpipe is installed in default bacula instalation.
 
 
1. Make a script in the bacula machine:
```
#!/bin/bash echo "bpipe:/var/100-vm.vma:/usr/bin/vzdump 100 --quiet --stdout --mode suspend:/usr/sbin/qmrestore - 100 --force"
```
Bpipe script have unlimited possibilities, see (http://bacula.us/using-bpipe-to-stream-dumps-vm-clones-and-another-data-to-your-backup/)
 
2. Make a Fileset in bacula-dir.conf file:
```
Fileset { Name = "100-vm-Fileset" Include { Plugin = "\\|/path/to/script/100-vm.sh" } }
```
 
3. Make Job with new Fileset and run it in bconsole:
```
run job=100-vm-job
```
 
4. Restore VM whit use bconsole:
```
restore jobid=282
```
Restore job make reconstruction VM, no recover *.vma file!

---

## Required device isn't connected or cannot be accessed - Windows 8.1

**ID:** 591 | **Typ:** post

Source:
- (https://www.easeus.com/backup-utility/a-required-device-is-not-connected-or-cannot-be-accessed.html)
 
### A month ago i migrate Windows 8.1 from old HDD to SSD. Everything is ok, and bang. OS wont start.
 
That is a list of what I was doing and how resolved problem.
 
 
1. Perform an automatic startup repair
```
1) Boot from the Windows 10, 8 or 7 USB or DVD bootable media. 2) Click Repair your computer. 3) Click Advanced Options. 4) Click Troubleshoot. 5) Click Startup repair. 6) Follow the onscreen instruction.
```
I tried with Windows 8.1 USB, Windows 10 USB, boot-repair-disk-64bit USB, Windows 8.1 CD, Windows 10 CD and boot-repair-disk-64bit CD. Nothing worked.
 
2. Perform a startup repair using command:
```
1) Boot from the Windows 10, 8 or 7 USB or DVD bootable media. 2) Click Repair your computer. 3) Click Advanced Options. 4) Click Troubleshoot and then Command Prompt. 5) Call these commands: bootrec /fixmbr bootrec /fixboot bootrec /rebuildbcd 6) The command fails? Try another suggested commands: bootrec /scanos bootrec /rebuildbcd bootrec /fixmbr bootrec /fixboot
```
I tried with Windows 8.1 USB, Windows 10 USB, Windows 8.1 CD and Windows 10 CD. Nothing worked.
 
3. I lost hope. I use SATA adapter to copy all files to another PC(Windows 10) from SSD before install OS. And then Windows 10 show prompt "Scan and Fix".

I think why not. I press "Scan and Fix". I put SSD back to PC. And what? OS is started normaly.
 
It turned out that Windows 10 also has a good power side.

---

## Add old volume to Bacula

**ID:** 606 | **Typ:** post

Source:
- (https://www.bacula.org/5.1.x-manuals/de/utility/utility/Volume_Utility_Tools.html#SECTION00273000000000000000)
- (http://www.bacula.pl/artykul/50/plik-bootstrap-w-teorii-i-w-praktyce/)
 
### Adding an archive volume:
```
sudo /usr/local/bacula/sbin/bscan -m -s /path/to/file/volume-name
```

---

## ProxMox - GPU Passthrough - Seabios PCI EXPRESS

**ID:** 670 | **Typ:** post

Source:
- (https://pve.proxmox.com/wiki/Pci_passthrough)
- (https://techblog.jeppson.org/2018/03/windows-vm-gtx-1070-gpu-passthrough-proxmox-5/)
- (https://forum.proxmox.com/threads/gpu-passthrough-tutorial-reference.34303/)
- (https://blog.quindorian.org/2018/03/building-a-2u-amd-ryzen-server-proxmox-gpu-passthrough.html/)

## 1. You **must** know what you want to do!

Read line by line: (https://pve.proxmox.com/wiki/Pci_passthrough)

## 2. Check what configuration is the best for you

#### Checking if card is UEFI (ovmf) or BIOS (Seabios) compatible:

**A**. Plugin your GPU thich you want passthrough and turn on the hypervisor

I have two GPU, the one I want passthrough is in the second PCI Express

**B**. Get and compile the software "rom-parser"

```
git clone https://github.com/awilliam/rom-parser cd rom-parser make
```

**C**. Find your GPU

Fast and no info:

```
lspci | grep -i --color 'vga\|3d\|2d'
```

Find yourself and all info:

```
lspci -v
```

Example output:
```
lspci | grep -i --color 'vga\|3d\|2d' 03:00.0 VGA compatible controller: NVIDIA Corporation G98 (rev a1) 04:00.0 VGA compatible controller: NVIDIA Corporation GT218 (rev a2)
```
GPU for passthrough is GeForce 210 and have ID **04:00.0**

**D**. Then dump the rom of you GPU

```
cd /sys/bus/pci/devices/0000:04:00.0/ echo 1 > rom cat rom > /tmp/image.rom echo 0 > rom
```

and test it with

```
./rom-parser /tmp/image.rom Valid ROM signature found @0h, PCIR offset 190h PCIR: **type 0**, vendor: 10de, device: 1280, class: 030000 PCIR: revision 0, vendor revision: 1 Valid ROM signature found @f400h, PCIR offset 1ch PCIR: **type 3**, vendor: 10de, device: 1280, class: 030000 PCIR: revision 3, vendor revision: 0 EFI: Signature Valid Last image
```

To be UEFI compatible, you need a "type 3" in the result.

If you have "type 3" go to (https://blog.quindorian.org/2018/03/building-a-2u-amd-ryzen-server-proxmox-gpu-passthrough.html/). This tutorial is for only "type 0". This mean that i use BIOS (Seabios), no UEFI (ovmf).

## 3. The real GPU Passthrough - Seabios PCI EXPRESS

**A**. Ensure VT-d is supported and enabled in the BIOS

**B**. Enable IOMMU on the host

- (https://pve.proxmox.com/wiki/Pci_passthrough#AMD_CPU)
- (https://pve.proxmox.com/wiki/Pci_passthrough#Intel_CPU)

I have Intel Xeon E5-1620. Edit **/etc/default/grub** file and replace

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
```

to

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on video=efifb:off"
```

Save your changes by running

```
update-grub
```

**C**. Blacklist kernel modules so they don’t get loaded at boot

In all tutorials which I found, always the NVIDIA and Nouveau modules is turn off. I have two NVIDIA GPU and I had lot of problems with first GPU. From "no refresh screen" to "stopping the hypervisor ".
I dont turn off this modules and this work for me.

**D**. Required modules

add to **/etc/modules**

```
vfio vfio_iommu_type1 vfio_pci vfio_virqfd
```

**E**. Assign your GPU to vfio driver

Example output:
```
lspci | grep -i --color 'vga\|3d\|2d' 03:00.0 VGA compatible controller: NVIDIA Corporation G98 (rev a1) 04:00.0 VGA compatible controller: NVIDIA Corporation GT218 (rev a2)
```
GPU for passthrough is GeForce 210 and have ID **04:00.0**

Run lspci -n -s 

 to obtain vendor ID:

```
lspci -n -s 04:00 04:00.0 0300: 10de:0a65 (rev a2) 04:00.1 0403: 10de:0be3 (rev a1)
```

Assign your GPU to vfio driver using the IDs obtained above:

```
echo "options vfio-pci ids=10de:0a65,10de:0be3" > /etc/modprobe.d/vfio.conf
```

**F**. Reboot the host(hypervisor)

**G**. Create Windows VM

Create your Windows VM using the default seabios bios but do not start it yet. Modify **/etc/pve/qemu-server/.conf** and ensure the file looks like:

```
agent: 1 bootdisk: ide0 cores: 2 ide0: local-zfs:vm-101-disk-0,size=100G **machine: q35** memory: 4096 name: vm-w-desktop-1 net0: e1000=4A:70:AE:3A:7D:71,bridge=vmbr0 **numa: 0** scsihw: virtio-scsi-pci smbios1: uuid=82b1f20f-b5bd-462a-8026-c342b1c37449 sockets: 2 vmgenid: 01cfb0fc-595e-4167-82a3-68a9a26a977a
```

This is normal configuration with "machine: q35"

Once installing and updating Windows there is one thing you need to make sure of, that you can remotely login to the VM. The reason why is because if you boot the VM with GPU Passhtrough, it disables to standard VGA adapter after which the built-in VNC/Spice from Proxmox will no longer work.

This means you need to enable RDP and set a static IP or install any other form of remote control (VNC, Teamviewer, etc.).

**H**.Passthrough the GPU

add to **/etc/pve/qemu-server/.conf**

```
hostpci0: **04:00**,x-vga=1,pcie=1
```

Save the file and see if the VM starts. If it does, great, you are probably done! Windows should see the GPU and start installing a driver from Windows Update. Let it do that for a few minutes and if you want you can then replace the driver with a freshly downloaded one.

I use NVIDIA 335.23 drivers.

---

## ProxMox – HDD Passthrough

**ID:** 745 | **Typ:** post

Source:
- (http://blog.imnotacyb.org/disk-passthrough-in-proxmox/)

## 1. Find disk with you want passthrough

I want passthrough all disk, no partition.

```
ls -l /dev/disk/by-id/
```

In my case is:

```
ls -l /dev/disk/by-id/ lrwxrwxrwx 1 root root 9 kwi 12 20:34 ata-ST9250315AS_5VC36VNP -> ../../sdd
```

## 2. Passthrough HDD to VM

Add HDD to VM config file. Add to **/etc/pve/qemu-server/.conf**

```
scsi1: /dev/disk/by-id/ata-ST9250315AS_5VC36VNP
```

## 3. Accept changes

You’ll want to shutdown then boot the VM (not just a regular restart) for the changes to take effect, after which the disk should be accessible in your VM.

---

## ProxMox - net::ERR_CONTENT_LENGTH_MISMATCH

**ID:** 750 | **Typ:** post

## Error repair
```
curl --insecure https://**IP**:8006/pve2/ext6/ext-all-debug.js
```

---

## Bacula-client (bacula-fd) debug mode

**ID:** 777 | **Typ:** post

1. Stop bacula-fd on client

```
systemctl stop bacula-fd.service
```

2. Start debug mode

```
/usr/local/bacula/sbin/bacula-fd -d 200 -c /usr/local/bacula/etc/bacula-fd.conf
```

3. Stop debug mode

```
pgrep -l bacula kill
```

 1. Start bacula-fd on client

```
systemctl start bacula-fd.service
```

---

## Proxmox disable power off button

**ID:** 785 | **Typ:** post

Source: (https://forum.proxmox.com/threads/disable-acpi-power-button-shutdown-host.64637/)

Edit file /etc/systemd/logind.conf. Uncomment **HandlePowerKey **and set value to **ignore**.
To save changes:

```
systemctl daemon-reload systemctl restart systemd-logind.service
```

---

## PrestaShop 1.6 - Change copyright year in footer

**ID:** 794 | **Typ:** post

Edit file **themes/default-bootstrap/footer.tpl** and replace year with:

```
{'Y'|date}
```

This change will automatically update year in footer.

---

## PrestaShop - Free module - sticky banner

**ID:** 800 | **Typ:** post

How To Create a Fixed Header on Scroll.

 

## Installation

1. Download ZIP: (https://github.com/siwy-github/ps_sticky_information)
2. Remove "-main" from zip file and first dir in zip
3. Add new module in Back Office

---

## K8s - Remove all pods in project:

**ID:** 804 | **Typ:** post

```
kubectl delete pods $(kubectl get pods -o=jsonpath='{.items.metadata.name}' -n project) -n project
```

---

## Delete open files. When df and du show different results.

**ID:** 832 | **Typ:** post

In Linux systems, files that have been deleted but are still open by some process can sometimes take up valuable disk space. Fortunately, there's a way to handle this issue.

Here's a command that allows you to "clean" these files:

```
find /proc/*/fd -ls | grep '(deleted)' | awk '{print $11}' | xargs -I % sh -c 'echo /dev/null > %'
```

Understanding what this command does requires breaking it down into parts. Let's start from the beginning.

The `find /proc/*/fd -ls` command searches all files in the `/proc/*/fd` directories. The `/proc/*/fd` directory contains files representing open file descriptors for every process. The `-ls` option makes `find` display detailed information about each found file.

Then `grep '(deleted)'` filters the results from the previous command, looking for lines that contain the phrase `(deleted)`. This phrase appears in `ls` results for files that have been deleted but are still open by some process.

Next, `awk '{print $11}'` prints the 11th field (understood as strings of characters separated by spaces) from each line. In the context of `ls` results, this field contains the file path.

Finally, `xargs -I % sh -c 'echo /dev/null > %'` executes a command for each file path, redirecting the content of `/dev/null` (which is an empty file) to the file at the given path. This essentially "cleans" the file - its content is replaced with an empty file.

In conclusion, this command allows you to find open but deleted files on your system and then "clean" these files. It's a very useful tool for managing disk space, especially in situations where a process still maintains open file descriptors for files that have been deleted.

However, remember that this command can be dangerous if you're not sure what it does. It's always worth understanding commands before you start using them on your system.

More useful commands: (https://github.com/PBujakiewicz/linux-cheat-sheet)

---

## Managing Nodes in OpenShift Using oc adm

**ID:** 838 | **Typ:** post

OpenShift, the container management platform created by Red Hat, offers powerful tools for managing nodes in your cluster. Today, we're going to focus on a few key commands that will assist us in optimizing and maintaining our clusters.

**Managing Node Schedulability**

First off, sometimes we want to prevent new Pods from being placed on a particular node. This might be necessary, for instance, when we're planning maintenance or an upgrade. In this case, we can use the command:

```
oc adm manage-node node-1 --schedulable=false
```

This command marks the `node-1` as unschedulable for new Pods. OpenShift's scheduler system will not place new Pods on this node.

**Evacuating Pods**

Sometimes, especially during maintenance, we want to safely evacuate all Pods from a given node. We can do this with the command:

```
oc adm manage-node node-1 --evacuate --pod-selector=deployment=app-xx
```

This command evacuates all Pods matching the selector criterion `deployment=app-xx` from the node `node-1`. During evacuation, Pods are safely shut down and restarted on another node in the cluster.

**Restoring Node Schedulability**

After we're done with maintenance or an upgrade, we want to restore our node to full functionality. We can do this with the command:

```
oc adm manage-node node-1 --schedulable=true
```

This command sets the `node-1` back to a state where it is schedulable for new Pods. This means the scheduler system can again place new Pods on this node.

**Draining a Node**

Sometimes, instead of performing these operations separately, we may want to do them all at once. OpenShift offers the `drain` command, which combines these operations together:

```
oc adm drain node-1
```

This command cordons the `node-1` for new Pods and evacuates all existing Pods to other nodes. This is a perfect solution for performing maintenance or upgrades on a given node.

**Conclusion**

The key to effectively managing OpenShift clusters is understanding and utilizing the available tools. These `oc adm` commands will help you keep your clusters in top shape, no matter what the future brings. However, keep in mind that these operations can temporarily increase the load on other nodes in the cluster as they have to take over additional Pods. Plan accordingly and keep your clusters in excellent condition!

More useful commands: (https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)(https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)

---

## Manually sending logs to stdout in a container.

**ID:** 843 | **Typ:** post

When working with Docker containers, there may be times when you need to pass information to the Docker logging system. Although there are various methods to achieve this, one intriguing approach involves using the special `/proc` filesystem.

The `/proc` directory is a unique area in Unix-like systems, containing information about running processes. For Docker containers, the process with an identifier of PID 1 is the main process launched within the container. Its standard output can be accessed as `/proc/1/fd/1`.

The command `echo "test log1" >> /proc/1/fd/1` appends the string "test log1" to the standard output stream of the main process in the container. In practice, this string will be written into the Docker logging system.

To view this log, you can use the command `docker logs `. In this way, you can inspect information passed to `/proc/1/fd/1`.

However, this is a command that requires appropriate permissions within the container. In practice, you need to have root privileges to write directly to `/proc/1/fd/1`.

In conclusion, utilizing `/proc/1/fd/1` is an intriguing and useful approach for directly passing logs to Docker's logging system. Remember, this is but one tool in your container management arsenal, and as always, it's best to understand how it works before you start using it.

More useful commands: (https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)(https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)

---

## An Alternative to the 'kubectl get all' Command.

**ID:** 847 | **Typ:** post

Have you ever wondered how to get a more detailed and complete view of resources in an OpenShift namespace? The standard `oc get all` command is helpful, but it is unable to display all the resources in namespaces, only returning a basic set of resource types. Today, I want to introduce you to an alternative that allows for a full list of listable resources.

The command I want to share with you is:

```
kubectl get $(kubectl api-resources --namespaced=true --verbs=list -o name | awk '{printf "%s%s",sep,$0;sep=","}') --ignore-not-found -n ${NAMESPACE} -o=custom-columns=KIND:.kind,NAME:.metadata.name --sort-by='kind'
```

Although it may look complex at first glance, it's quite simple when broken down into parts. The command consists of three main segments.

1. First off, `kubectl api-resources --namespaced=true --verbs=list -o name` calls a list of all resources that are namespace-related and listable. The result is returned in name format.
2. The next segment, `awk '{printf "%s%s",sep,$0;sep=","}'`, is an AWK script that processes the result of the previous command into a comma-separated list.
3. The final command, `kubectl get $(...) --ignore-not-found -n ${NAMESPACE} -o=custom-columns=KIND:.kind,NAME:.metadata.name --sort-by='kind'`, utilizes the resultant format from the previous commands to fetch resources from a specified namespace, denoted by the `${NAMESPACE}` variable. It sets the `--ignore-not-found` parameter to ignore resources that are not found. The `-o=custom-columns=KIND:.kind,NAME:.metadata.name` option specifies the output format to only show the kind of resource and its name. Additionally, the `--sort-by='kind'` flag serves to sort the results by the kind of resource.

This command is particularly useful when you want to understand what resources are available in your namespace. It allows for the display of a full list of listable resources, as opposed to the `kubectl get all` command which only returns a basic set of resource types.

However, bear in mind that this command is not a one-size-fits-all solution. In the case of a large number of resources in a namespace, the results can be overwhelming. Nevertheless, it is a powerful tool that allows for a more comprehensive picture of what's happening in your OpenShift cluster.

So the next time you need a more complete picture of resources in an OpenShift namespace, consider utilizing this command. It might just be the tool you need to better understand and manage your infrastructure.

More useful commands: (https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)(https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet)

---

## Windows 10 Doesn't See Drives - Solving Installation Issues on Dell Precision T3600

**ID:** 865 | **Typ:** post

Recently, I encountered a rather unusual problem while installing Windows 10 on a Dell Precision T3600 computer. During the installation process, the Windows installer was unable to identify any of the hard drives mounted in the computer. On the motherboard, there were two types of connectors: "HDD" and "SATA", numbered from 0 to 2 and from 0 to 1 respectively.

Regardless of many attempts, none of my three drives (two HDDs and one SSD) were visible to the Windows installer. After extensive research, I concluded that connectors labelled as "HDD" are used for RAID configurations, and "SATA" connectors are ordinary ones, which are normally visible to the Windows installer.

I decided to install Windows 10 on the SSD by connecting it to the SATA connector. Despite successful installation, the system still couldn't detect the HDD drives.

I found the solution to this issue, which involved enabling RAID function in the BIOS and installing SAS drivers. After these changes, all drives were correctly detected and available in the system.

In summary, if you ever encounter a similar problem with Windows 10 installation on Dell Precision T3600 computers, check the configuration of connectors on the motherboard and remember about SAS drivers and RAID configuration. Although it may seem complicated, the steps above helped me solve the problem and they might also help you.

---

## Translating My CV into Code: A Journey from JSON to LaTeX

**ID:** 869 | **Typ:** post

When it comes to managing and formatting a curriculum vitae (CV), traditional methods can sometimes feel limiting. In an attempt to find a flexible and tech-savvy solution, I embarked on a journey to translate my CV into code. Here's what I discovered:

#### **Exploring JSON Resume**

My first experiment was with (https://jsonresume.org), a project that seemed initially promising. JSON Resume allows you to keep all of your data in a JSON file, and from there, you can simply generate a PDF with your chosen theme.

However, as I delved deeper, I found that the themes are developed by third parties, and if a theme does not support a particular section, it will be omitted. In my case, this led to the exclusion of an essential 'Certificates' section, a non-negotiable element for me.

#### **The Leap to LaTeX**

Discouraged but not defeated, I turned my attention to (https://www.latex-project.org), a high-quality typesetting system; but it comes with a steep learning curve. Writing a CV from scratch, especially for a LaTeX novice, can be a daunting task. The syntax and rules required substantial effort to grasp.

Thankfully, I stumbled upon a plethora of LaTeX templates for CVs on (https://www.overleaf.com/latex/templates/tagged/cv). These templates provided a strong foundation, allowing me to tailor each element to my specific needs.

#### **Embracing LaTeX: The Final Product**

After a few evenings of playing around with LaTeX, I was thrilled with the results. Not only did I have complete control over the appearance of my CV, but I also found that future updates would be much more enjoyable. The final product is available on my GitHub repository: (https://github.com/PBujakiewicz/resume).

#### **Conclusion**

My journey from JSON to LaTeX was enlightening, filled with lessons learned and unexpected challenges. If you're looking to take control over your CV's format and appearance, LaTeX is a powerful tool that's worth the initial learning curve. While JSON Resume may suit those with more straightforward requirements, LaTeX offers unparalleled flexibility for those willing to invest the time.

---

## Exploring Directories and Files Using Linux Commands

**ID:** 879 | **Typ:** post

In the Linux environment, we have access to a range of powerful tools that allow us to effectively manage and search for files and directories. In this article, I will focus on two exemplary commands that enable us to find directories with a specific name and search files for a particular pattern.

**1. Searching for directories with a specific name**

To find all directories named "text" in the current directory and all its subdirectories, we can use the `find` command:

```
find . -type d -name 'text'
```

Let's break down this command:

- `find` is the basic tool for searching files and directories in the system.
- `.` indicates the current directory. This is where the search will begin.
- `-type d` limits the search to directories (`d` stands for directory).
- `-name 'text'` specifies that we're looking for directories named "text".

**2. Searching files based on a pattern in their content**

To search files in the current directory and all its subdirectories for lines starting with the word "text", we can use the `grep` command:

```
grep -rni -E '^text' .
```

The individual elements of this command are:

- `grep` is a text-searching tool.
- `-r` (or `--recursive`) allows for searching all files in the current directory and subdirectories.
- `-n` displays the line number where a match is found.
- `-i` makes the search case-insensitive.
- `-E` enables the use of extended regular expressions.
- `^text` is a regular expression that matches lines starting with the word "text".
- `.` indicates the current directory as the starting point for the search.

I hope the above commands will become a handy tool in your daily work with Linux. They offer quick and effective file and directory management, and their capabilities go far beyond what's shown in this article. I encourage you to experiment and explore other options these tools provide!

More useful commands: (https://github.com/PBujakiewicz/linux-cheat-sheet/tree/main)

---

## Route vs Ingress: Differences Between OKD and Kubernetes

**ID:** 885 | **Typ:** post

If you're involved in container management or developing applications in cloud environments, you've likely come across the terms "Route" in OpenShift (often referred to as OKD in its community version) and "Ingress" in Kubernetes. While both mechanisms serve similar functions, that is directing traffic to containerized applications, there are key differences between them. Here are a few:

1. **Origin**: - **Route**: This is a concept introduced in OpenShift/OKD. Route allows for exposing services externally out of the cluster. - **Ingress**: This is a native Kubernetes object used for directing HTTP and HTTPS traffic from the outside to internal cluster services.
2. **Controller**: - **Route**: OpenShift/OKD provides its own routing controller, which integrates with the HAProxy router that handles all incoming requests. - **Ingress**: For Ingress to function in Kubernetes, you need an Ingress controller, such as the Nginx Ingress Controller, Traefik, among others. The Ingress controller dictates how traffic will be routed to services within the cluster.
3. **Configuration**: - **Route**: In OpenShift/OKD, you can use different types of routing, such as passthrough, edge, and reencrypt. These provide different levels of control over SSL/TLS certificates. - **Ingress**: Ingress configuration in Kubernetes is more standardized, but with added flexibility provided by the various Ingress controllers.
4. **Protocol Support**: - **Route**: Primarily focused on HTTP/HTTPS traffic, but passthrough allows for other protocols. - **Ingress**: Traditionally limited to HTTP/HTTPS traffic, but some Ingress controllers offer support for other protocols.
5. **Extensions and Plugins**: - **Route**: Is more tightly integrated with the OpenShift/OKD platform and makes use of platform-specific features. - **Ingress**: Can be more flexible due to the variety of available Ingress controllers, each with its own feature set and plugins.

In conclusion, both Route in OKD/OpenShift and Ingress in Kubernetes serve to direct traffic to containerized applications but differ in terms of configuration, implementation, and support for various protocols. The choice between them depends on the specific use case, needs, and the platform on which your application is running.

---

## How to find which folder under overlay2 directory belong to which container?

**ID:** 891 | **Typ:** post

If you've been working with Docker, you might have wondered where Docker actually stores the filesystems of your containers on your host machine. This information can be crucial for various reasons, such as debugging, storage optimization, or general curiosity.

Well, with a simple one-liner, you can get a neat list of all your containers along with their storage directories. Here it is:

```
for container in $(docker ps --all --quiet --format '{{ .Names }}'); do echo "$(docker inspect $container --format '{{.GraphDriver.Data.MergedDir }}' | grep -Po '^.+?(?=/merged)' ) = $container"; done
```

Run this command in your terminal, and it will output a list in the following format:

```
/path/to/container/storage = container_name_1
```

More useful commands: (https://github.com/PBujakiewicz/kubectl-oc-cheat-sheet/tree/main)

---

## Improving Network Performance with Receive Packet Steering (RPS)

**ID:** 895 | **Typ:** post

In modern systems, network performance can be limited by the host’s ability to handle packet processing, especially with a single receive queue. Receive Packet Steering (RPS) helps solve this issue by distributing incoming packets across multiple CPU cores, significantly improving performance. Proper configuration involves modifying system files related to the queues responsible for packet handling. Implementing RPS can lead to noticeable performance boosts in network-intensive environments.

For more details, visit (https://www.suse.com/support/kb/doc/?id=000018430).

---

