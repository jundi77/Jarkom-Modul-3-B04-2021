# Jarkom-Modul-3-B04-2021

Naufal Fajar Imani             05111940000007

Jundullah H. R.                05111940000144

Yeremia D Limantara            05111940000232

---

![image](https://user-images.githubusercontent.com/40772378/141704365-19995c61-17ca-40c6-b4ea-c56bea9deea6.png)

Diinginkan struktur node-node seperti gambar di atas. Akan dibuat bahwa untuk node yang ada:

- EniesLobby sebagai DNS server
- Jipangu sebagai DHCP server
- Water7 sebagai Proxy server
- Foosha sebagai DHCP relay
- Skypie sebagai Webserver super.franky.b04.com

Langkah awal adalah membuat node-node terlebih dahulu, kemudian untuk seluruh client yang melalui Switch1 dan Switch3 kecuali Skypie, diberikan config `/etc/network/interfaces` sebagai berikut:

```
auto eth0
iface eth0 inet dhcp
```

Config tersebut memiliki makna bahwa pengambilan IP akan dilakukan otomatis dengan DHCP.

Kemudian, untuk host yang melalui Switch2, menggunakan config `/etc/network/interfaces` sebagai berikut

- EniesLobby
```
auto eth0
iface eth0 inet static
	address 10.9.2.2
	netmask 255.255.255.0
	gateway 10.9.2.1
```
Config ini mengikuti latihan modul dan praktikum sebelumnya.
- Water7
```
auto eth0
iface eth0 inet static
	address 10.9.2.3
	netmask 255.255.255.0
	gateway 10.9.2.1
```
Config ini mengikuti latihan modul dan praktikum sebelumnya.
- Jipangu
```
auto eth0
iface eth0 inet static
	address 10.9.2.4
	netmask 255.255.255.0
	gateway 10.9.2.1
```
Config ini mengikuti latihan modul dan praktikum sebelumnya.

Lalu untuk host Skypie, diberikan config `/etc/network/interfaces` DHCP yang menyatakan hwaddress, supaya IP host nantinya dapat diatur untuk tetap berada di `10.9.3.69`:
```
auto eth0
iface eth0 inet dhcp
hwaddress ether 7a:48:15:2c:35:8f
```

## Setup Host Foosha (Router dan Relay DHCP)

Pada Foosha, akan dilakukan setup untuk router data dan sebagai DHCP relay. Dijalankan script berikut:
```sh
echo Applying NAT settings
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE -s 10.9.0.0/16

# Install package yang diperlukan untuk relay DHCP
apt-get update
apt-get install -y isc-dhcp-relay && \
cp /root/isc-dhcp-relay /etc/default && \ # Copy config yang diperlukan untuk DHCP relay
dhcrelay 10.9.2.4 # Laksanakan relay ke Jipangu
```

Adapun untuk file-file config di Foosha:

- `/root/isc-dhcp-relay` sebagai file config relay DHCP
```
# What servers should the DHCP relay forward requests to?
SERVERS="10.9.2.4"

# On what interfaces should the DHCP relay (dhrelay) serve DHCP requests?
INTERFACES="eth1 eth3"

# Additional options that are passed to the DHCP relay daemon?
OPTIONS=""
```
Diatur bahwa relay mendengarkan request di `eth1` dan `eth3` (Switch1 dan Switch3), lalu request dilanjutkan ke `10.9.2.4` (Jipangu).

## Setup Host EniesLobby (DNS Server)

Pada EniesLobby, akan dilakukan setup untuk DNS server. Dijalankan script berikut:
```sh
echo nameserver 192.168.122.1 > /etc/resolv.conf # DNS dari Foosha agar bisa konek ke internet

# Install bind9 untuk server DHCP
apt-get update
apt-get install bind9 -y

rm -rf /etc/bind # Hapus config default bind9
cp -rf /root/bind /etc # Copy config yang diperlukan untuk DNS server
service bind9 restart # Apply config
```

Adapun untuk config yang diperlukan di DNS server:
- `/root/bind/named.conf.options`
```
options {
        directory "/var/cache/bind";
        
        // DNS forward ke Foosha agar dapat mengakses internet melalui DNS server ini
        forwarders {
                192.168.122.1;
        };
        allow-query{any;};

        auth-nxdomain no;    # conform to RFC1035
        listen-on-v6 { any; };
};
```

- `/root/bind/named.conf.local`
```
zone "jualbelikapal.b04.com" {
        type master;
        file "/etc/bind/kaizoku/jualbelikapal.b04.com";
};

zone "super.franky.b04.com" {
        type master;
        file "/etc/bind/kaizoku/super.franky.b04.com";
};
```

- `/root/bind/kaizoku/jualbelikapal.b04.com`, untuk domain proxy
```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     jualbelikapal.b04.com. root.jualbelikapal.b04.com. (
                     2021100401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      jualbelikapal.b04.com.
@       IN      A       10.9.2.3        ; IP Water7
```

- `/root/bind/kaizoku/jualbelikapal.b04.com`, untuk domain webserver
```
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     super.franky.B04.com. root.super.franky.B04.com. (
                     2021100401         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      super.franky.B04.com.
@       IN      A       10.9.3.69        ; IP Skypie
```

## Setup Host Jipangu (DHCP Server)

Pada host Jipangu, akan dilakukan setup untuk DHCP server. Dijalankan script berikut:
```sh
# Apply DNS EniesLobby
echo nameserver 10.9.2.2 > /etc/resolv.conf

# Install DHCP server
apt-get update
apt-get install -y isc-dhcp-server

# Apply config DHCP server
cp /root/dhcpd/dhcpd.conf /etc/dhcp/dhcpd.conf
service isc-dhcp-server restart
```

Adapun untuk config yang diperlukan di DHCP server:
- `/root/dhcpd/dhcpd.conf`
```
ddns-update-style none;
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;

log-facility local7;

### END OF SAMPLE FILE

subnet 10.9.1.0 netmask 255.255.255.0 {
#    interface eth0;
    range 10.9.1.20 10.9.1.99;
    range 10.9.1.150 10.9.1.169;
    option routers 10.9.1.1;
    option broadcast-address 10.9.1.255;
    option domain-name-servers 10.9.2.2;
    default-lease-time 360; #6m
    max-lease-time 720; #12m
}

subnet 10.9.3.0 netmask 255.255.255.0 {
#    interface eth0;
    range 10.9.3.30 10.9.3.50;
    option routers 10.9.3.1;
    option broadcast-address 10.9.3.255;
    option domain-name-servers 10.9.2.2;
    default-lease-time 720; #12m
    max-lease-time 7200; #120m
}

# https://unix.stackexchange.com/a/294947 Thanks for the answer!
subnet 10.9.2.0 netmask 255.255.255.0 {
#    interface eth0;
    option routers 10.9.2.1;
}

# Tetapkan IP yang diberikan untuk hwaddress yang diberikan
# Dalam config ini, hwaddress adalah milik Skypie
host Skypie {
    hardware ethernet 7a:48:15:2c:35:8f;
    fixed-address 10.9.3.69;
}
```
Dilakukan setup untuk tiap-tiap subnet:

- `10.9.1.0`, menyesuaikan soal nomor 3 bahwa client yang melalui Switch1 mendapatkan range IP dari 10.9.1.20 - 10.9.1.99 dan 10.9.1.150 - 10.9.1.169, menggunakan DNS EniesLobby, lease time 6 menit dengan maksimal 12 menit.
- `10.9.3.0`, menyesuaikan soal nomor 3 bahwa client yang melalui Switch1 mendapatkan range IP dari 10.9.3.30 - 10.9.1.50, menggunakan DNS EniesLobby, lease time 12 menit dengan maksimal 120 menit.
- `10.9.2.0`, untuk config subnet yang digunakan oleh Jipangu.

## Setup Host Water7 (Proxy Server)

Pada host Water7, akan dilakukan setup untuk proxy server. Dijalankan script berikut:
```sh
# Apply DNS EniesLobby
echo nameserver 10.9.2.2 > /etc/resolv.conf

# Install squid
apt-get install squid -y && \
cp -rf /root/squid /etc && \ # Apply config
service squid restart
```

Adapun untuk config yang diperlukan di proxy server:
- `/root/squid/squid.conf`
```
# acl untuk definisi waktu-waktu yang diperlukan
acl senin_kamis time MTWH 07:00-11:00
acl selasa_jumat_malam time TWHF 17:00-23:59
acl rabu_sabtu_pagi time WHFA 00:00-03:00

# acl untuk deteksi jika sedang membuka gambar png atau jpg, dengan asumsi lowercase
# untuk selanjutnya, jika ada kata gambar saja maka mengacu ke gambar png atau jpg
acl png url_regex .png
acl jpg url_regex .jpg

# Jalankan proxy di port 5000
http_port 5000
visible_hostname Water7

# Lakukan redireksi ke super.franky.b04.com jika menuju google
acl google url_regex google.com # acl untuk deteksi jika sedang membuka google.com
http_access deny google # tolak akses
deny_info http://super.franky.b04.com google # arahkan ke super.franky

# Konfigurasi user dan password untuk auth basic
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/.htpasswd # letak file yang berisi user dan password
auth_param basic children 5
auth_param basic realm Proxy
auth_param basic credentialsttl 2 hours
uth_param basic casesensitive on
acl luffy proxy_auth luffybelikapalb04 # acl untuk user yang login dengan luffy
acl zoro proxy_auth zorobelikapalb04 # acl untuk user yang login dengan zoro

# zoro tidak boleh mengakses gambar dan hanya boleh mengakses untuk waktu tertentu
http_access allow zoro !png !jpg senin_kamis
http_access allow zoro !png !jpg selasa_jumat_malam
http_access allow zoro !png !jpg rabu_sabtu_pagi

# luffy hanya boleh mengakses untuk waktu tertentu
http_access allow luffy senin_kamis
http_access allow luffy selasa_jumat_malam
http_access allow luffy rabu_sabtu_pagi

# Limit bandwidth
delay_pools 1
delay_class 1 1
delay_parameters 1 10000/10000 # 10kbps
delay_access 1 allow png # Laksanakan bandwidth jika membuka gambar png, atau ...
delay_access 1 allow jpg # jika membuka gambar jpg
```

- `/root/squid/.htpasswd`
```
luffybelikapalb04:$apr1$jCZkbVoL$IhWRN0m/2WVyc1Rxt4uZ2.
zorobelikapalb04:$apr1$XxLiE8gC$7U9FLmerIw2xxYnjOSRU/0
```

Berisi user dan password dalam MD5. Pembuatan file dilakukan dengan bantuan program `htpasswd` dengan argumen `-m` untuk password dalam MD5.


## Setup Host Skypie (Web Server)

Pada host Skypie, akan dilakukan setup untuk webserver. Dijalankan script berikut:
```sh
# Install apache2
apt-get update
apt-get install apache2

# Apply config
cp -rf /root/000-default.conf /etc/apache2/sites-enabled && \
cp -rf /root/super.franky.b04.com /var/www && \ # Copy file untuk super.franky
service apache2 restart
```

Adapun untuk config yang diperlukan di webserver:
- `/root/000-default.conf`
```
ServerName super.franky.b04.com

<VirtualHost *:80>
    ServerName  10.9.3.69
    Redirect    301 /   http://super.franky.b04.com
</VirtualHost>

<VirtualHost *:80>
    ServerName super.franky.b04.com
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/super.franky.b04.com
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Config menyatakan bahwa nantinya super.franky.b04.com akan dilayani di direktori `/var/www/super.franky.b04.com`.

## Setup Client Logouetown (Proxy Client)

Jalankan script berikut untuk memasang lynx untuk mencoba browsing dan apply proxy:
```sh
# Install lynx
apt-get update && apt-get install -y lynx

# Apply proxy
export http_proxy="http://jualbelikapal.b04.com:5000"
```

---

## Kendala

Kendala-kendala yang dialami adalah sebagai berikut:
- Tidak terbiasa dengan logic http_access squid, diketahui bahwa beda line sama dengan OR, sama line sama dengan AND.
- Paham saat DHCP server berada di router, namun agak bingung saat diletakkan di host. Diketahui ternyata memang bisa client dan server dalam satu interface.
