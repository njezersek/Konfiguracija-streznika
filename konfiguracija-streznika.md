# Konfiguracija strežnika Ubuntu server 18.04 LTS
*30. 10. 2018, Nejc Jezeršek*
## Linux containers
Linux containers omogoča virtualizacijo na nivoju operacijskega sistema.

Najprej namestimo LXD (Linux containers daemon), ki skrbi za delovanje containerjev. In ga inicializiramo (med inicijalizacijo nas program vpraša za nekaj nastavitev, ki jih postimo na privzetih vrednostih).

```bash
apt install lxd lxd-client
lxd init
```
Primer postopka inicializacije:
```
Would you like to use LXD clustering? (yes/no) [default=no]:
Do you want to configure a new storage pool? (yes/no) [default=yes]:
Name of the new storage pool [default=default]:
Would you like to connect to a MAAS server? (yes/no) [default=no]:
Would you like to create a new local network bridge? (yes/no) [default=yes]:
What should the new bridge be called? [default=lxdbr0]:
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]:
Would you like LXD to be available over the network? (yes/no) [default=no]:
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```
#### Upravljanje s containerji
Z ukazom `lxc` lahko upravljamo s containerji.
Če želimo narediti nov container uporabimo ukaz `lxc launch <slika operacijskega sistema> <ime containerja>`. Za sliko operacijskega sistema dobimo iz oddaljenega strežnika slik (https://us.images.linuxcontainers.org/). Če ne nastavimo imena containerja bo ta dobil ime sestavljeno iz naključnega pridevnika in samostalnika.
``` bash
lxc launch ubuntu:18.04 MojContainer
```
Da se je container res ustvaril lahko preverimo z ukazmom `lxc ls` ali `lxc list`, ki prikaže vse kontainerje.
``` bash
lxc ls
```
```
+--------------+---------+---------------------+------+------------+-----------+
|     NAME     |  STATE  |        IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+--------------+---------+---------------------+------+------------+-----------+
| MojContainer | RUNNING | 10.7.4.123 (eth0)   | ...  | PERSISTENT |           |
+--------------+---------+---------------------+------+------------+-----------+
```
Če želimo ukazovati containerju uporabimo ukaz `lxc exec <ime containerja> -- <ukaz>`. Na ta način lahko v containerju odpremo terminal:
``` bash
lxc exec MojContainer -- /bin/bash
```

Container lahko ustavimo `lxc stop <ime containerja>`, zaženemo `lxc start <ime containerja>` in ponovno zaženemo `lxc restart <ime containerja>`:
``` bash
lxc stop MojContainer
lxc start MojContainer
lxc restart MojContainer
```
Container lahko izbrišemo z ukazom `lxc remove <ime containerja>`:
``` bash
lxc remove MojContainer
```

#### Omrežne nastavitve
Po privzetih nastavitvah so containerji povezani v virtualno omrežje, ki je prek gostitelja, ki se obnaša kot virtualni router, povezano na internet.

Če želimo dostopati do containerjev iz drugih omrežij moramo nastaviti da so povezani v isto omrežje kot gostitelj. To naredimo tako da containerju dodelimo profil z nastavitvami. Privzeto vsak container uporablja profil *default*. Naredimo nov profil tako, da skopiramo z ukazom `lxc profile copy <ime originalnega profila> <ime profila kopije>` privzetega in ga uredimo z ukazom `lxc profile edit <ime profila>`.

**Iz praktičnih razlogov izberemo čim krajše ime profila!**
Primer: namesto *MacAddressVirtualLocalAreaNetworkServer* uporabimo *macvlan*.
``` bash
lxc profile copy default MojProfil
lxc profile edit MojProfil
```
Konfiguracijska datoteka se odpre z urejevalnikom besedila *nano*.

``` yaml
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    nictype: macvlan
    parent: <networking interface>
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: MojProfil
used_by: []
```
Spremenimo *nictype* iz `bridged` na `macvlan` in *parent* na ime omrežnega vmesnika.
Ime omrežngea vmesnika lahko dobimo z ukazom `ifconfig` ali pa `ip route show default 0.0.0.0/0`.
``` bash
ifconfig
```
```
enp4s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.115  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::6245:cbff:fe87:6d1b  prefixlen 64  scopeid 0x20<link>
        inet6 2a00:ee2:206:9700:6245:cbff:fe87:6d1b  prefixlen 64  scopeid 0x0<global>
        ether 60:45:cb:87:6d:1b  txqueuelen 1000  (Ethernet)
        RX packets 668789  bytes 772971078 (772.9 MB)
        RX errors 0  dropped 54  overruns 0  frame 0
        TX packets 335206  bytes 62407432 (62.4 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...
```
V tem primeru je ime omrežnega vmesnika **enp4s0**.

Z ukazom `lxc profile assign <ime containerja>` nastavimo profil containerju. Da uveljavimo spremembe ga moramo še ponovno zagnait.
``` bash
lxc profile assign MojContainer
lxc restart MojContainer
```
Zdaj bo container prejel svoj MAC naslov in se povezal v isto omrežje kot gostiltelj.

#### Reference:
*	https://linuxcontainers.org
*	https://help.ubuntu.com/lts/serverguide/lxd.html.en
* https://us.images.linuxcontainers.org/
* https://www.youtube.com/watch?v=3f57PovdY44
* https://tutorials.ubuntu.com/tutorial/tutorial-setting-up-lxd-1604#0

## Apache
Apache je primarno spletni strežnik. Omogoča pa tudi preusmeritev prometa na drug strežnik s pomočjo proxy-ja.

#### Osnovna namestitev
Začnimo z nameščanjem Apache na glavnem strežniku:
``` bash
apt install apache2 -y
```
Apache privzeto prikazue spletno stran, ki je shranjena v mapi `/var/www/html`. Spremenimo vsebino spletne strani.
``` bash
nano /var/www/html/index.html
```
Z oreodjem `curl` lahko stestiramo spletni strežnik.
``` bash
curl localhost
```
Če pošljemo poizvedbo na localhost dobimo vsebino spletne strani:
``` html
<h1>Hello World!</h1>
to je glavni strežnik
```

#### Konfiguracija
Večina konfiguracijskih datotek za Apache se nahaja v mapi `/etc/apache2`. Če pogledamo vsebino te mape najdemo kar nekaj datotek in map:
* `apache2.conf`: To je glavna konfiguracijska datoteka. Skoraj vse bi lahko nastavili v tej datotieki, a je priporočljivo, da uporabimo več datotek za lažjo preglednost.
* `ports.conf`: Ta datoteka je namenjena nastavitvi portov (vrat) na katerih so dostopne spletne strani.
* `sites-available`: V tej mapi so shranjene konfiguracijske datoteke za vse spletne strani.
* `sites-enabled`: V tej mapi so linki do konfiguracijsikih datotek, ki so v mapi *sites-available*, za spletne strani, ki so trenutno omogočene.
* `mods-enabled`, `mods-available`: Ti dve mapi delujeta podobno kot *sites-* mapi, le da se jih uporablja za nastavitve modulov, ki jih uporablja Apache.

#### Virtual hosts
Če želimo prikazati različne strani za različne domene, naredimo novo mapo z imenom domene v mapi `/var/www` in vanjo shranimo spletno stran za to domeno.
``` bash
mkdir /var/www/mojaDomena
```
Nato naredimo novo konfiguracijsko datoteko v mapi `sites-available`. To naredimo tako da skopiramo privzeto konfiguracijsko datoteko v novo datoteko in jo uredimo.
``` bash
cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/mojaDomena.conf
nano /etc/apache2/sites-available/mojaDomena.conf
```
Nastavimo parametere:
* `ServerName` na domeno za katero bi radi prikazali drugačno spletno stran.
* `ServerAdmin` na email andministratorja spletne strani.
* `DocumentRoot` na absolutno pot do mape s spletno stranjo.
* `ErrorLog` in `CustomLog` povesta kam se shranjujejo zapiski o napakah in poizvedbah. Lahko ju pustimo na privzetih vrednostih.

``` xml
<VirtualHost *:80>
        ServerName moja-domena.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/mojaDomena
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Da se bo konfiguracijska datoteka, ki smo jo pravkar naredili, uporabila moramo narediti link v mapi `sites-enabled`, ki kaže na našo konfiguracijsko datoteko. To naredimo z ukazom `a2ensite <konfiguracijska datoteka>` (za ta ukaz poterbujemo samo ime konfiguracijske datoteka v mapi sites-available, ni potrebno pisati končnice).
``` bash
a2ensite mojaDomena
```
Da je ukaz res naredil link v mapi `sites-enabled` lahko pogledamo z ukazom `ls -l` (zastavica `-l` pomeni da pokaže dolgo verzijo izpisa direktorija).
``` bash
ls -l /ect/apache2/sites-enabled
```
```
total 0
lrwxrwxrwx 1 root root 35 Oct 29 17:58 000-default.conf -> ../sites-available/000-default.conf
lrwxrwxrwx 1 root root 28 Oct 30 20:14 mojaDomena.conf -> ../sites-available/mojaDomena.conf
```
Če želimo konfiguracijsko datoteko onemogočiti (odstraniti link), uporabimo ukaz `a2dissite <konfiguracijska datoteka>`;

Da se uveljavijo spremembe moramo ponovno zagnati apache:
``` bash
service apache2 restart
```

Zdaj bo apache poizvedbam iz domene *moja-domena.com* postregel iz mape `/var/www/mojaDomena`, vsem ostalim poizvedbam pa iz mape `/var/www/html`.
Če želimo imeti še več strani za različne domene postopek ponovimo.

#### Proxy
Recimo da imamo več strežnikov (ki so lahko tudi virtualni - containerji) v istem omrežju z enim zunanjim IP naslovom. Želeli bi za različne domene prikazati različno spletno stran, ki jo gosti različni strežnik.

To lahko naredimo tako, da ves promet najprej pripeljemo na glavni strežnik. Ta pa se glede na domeno kateri je namenjen promet preusmeri na pravi strežnik.

Najprej namestimo Apache (lahko tudi kateri koli drugi strežnik) na drugem (lahko tudi virtalnem) strežniku. Pomembno je da spletnemu strežniku spremenimo port. Če uprabljamo apache to naredimo v datotekah `/etc/apache2/ports.conf` in `/etc/apache2/sites-available/000-default.conf`

v `/etc/apache2/ports.conf` `Listen` nastavimo iz 80 na 8080.

``` xml
Listen 8080

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>
```

V `/etc/apache2/sites-available/000-default.conf` spremenimo 80 v 8080
``` xml
<VirtualHost *:8080>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html


        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```

Nato lahko na glavnem strežniku uporabimo proxy modul, ki je privzeto nameščen poleg apache. Treba ga je samo omogočiti. To naredimo podobno kot smo omogočili konfiguracijsko datoteko za spletno stran, ki se prikaže samo na dolečeni domeni.
``` bash
a2enmod proxy
a2enmod proxy_http
a2enmod proxy_balancer
a2enmod lbmethod_byrequests
```

Nato naredimo novo konfiguracijsko datoteko za *VirtualHost* (gelj prejšnje podpoglavje) in jo omogočimo.
Ker želimo da določena domena prikazuje spletno stran iz drugega strežnika moramo konfiguracijsko datoteko nekoliko spremeniti. Dodamo `ProxyPass` in `ProxyPassReverse`.

``` xml
<VirtualHost *:80>
        ServerName moja-domena.com
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/mojaDomena
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        ProxyPreserveHost On
        ProxyPass / http://10.7.4.15:8080/
        ProxyPassReverse / http://10.7.4.15:8080/
</VirtualHost>
```
* `ProxyPreserveHost` prenese originalni `Host` header na drugi strežnik. To naj bi preprečilo, da te drugi strežnik redirekta na svoj IP.
* `ProxyPass` v tem primeru pove da vse poizvedbe (`/`) preusmeri na drugi strežnik z IP naslovom 10.7.4.15 in portom 8080.
* `ProxyPassReverse` mora imeti enake nastavitve kot `ProxyPass`. Tudi ta naj bi preprečil, nezaželene preusmeritve končnega uporabnika, na IP naslov drugega strežnika, ki navzven ni dostopen.

Da se te spremembe uveljavijo moramo ponovno zagnati apache.

Če zdaj obiščemo moja-domena.com bo apache vrnil spletno stran iz drugega strežnika.


#### Reference
* https://httpd.apache.org/docs/2.4/howto/reverse_proxy.html
* http://httpd.apache.org/docs/2.2/mod/mod_proxy.html#proxypass
* https://stackoverflow.com/questions/9831594/apache-and-node-js-on-the-same-server
* https://www.digitalocean.com/community/tutorials/how-to-use-apache-as-a-reverse-proxy-with-mod_proxy-on-ubuntu-16-04
* https://www.digitalocean.com/community/tutorials/how-to-configure-the-apache-web-server-on-an-ubuntu-or-debian-vps
* https://tutorials.ubuntu.com/tutorial/install-and-configure-apache#3
* https://manpages.debian.org/jessie/apache2/a2ensite.8.en.html

## Nextcloud
Nextcloud je zelo dober opensource program za ustvarjanje svojega oblaka za hrambo podatkov.

Namestimo ga z ukazom:
``` bash
snap install nextcloud
```
Če ga želimo uporabljati prek proxyja mu moramo spremeniti port:
``` bash
sudo snap set nextcloud ports.http=8080
```
#### Reference
* https://nextcloud.com/
* https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-nextcloud-on-ubuntu-18-04
