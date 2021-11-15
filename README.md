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

## Setup Host EniesLobby

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

## Setup Host Jipangu


