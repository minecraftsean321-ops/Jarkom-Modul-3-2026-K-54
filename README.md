# Jarkom-Modul-3-2026-K-54

| Nama                 | NRP        |
|----------------------|------------|
| Viko Rizky Fauzan    | 5027251017 |
| Sean Arthur Tamajaya | 5027251050 |

## Tabel IP Host

Asumsi prefix kelompok adalah `192.238`. Semua interface internal menggunakan subnet mask `/24`.

| Network | Node | Peran | Alamat IP | Default Gateway |
|---|---|---|---|---|
| NAT | `jichi` (`eth0`) | Koneksi WAN ke NAT | DHCP dari NAT | Dari NAT |
| Network 1 (`192.238.1.0/24`) | `jichi` (`eth1`) | Gateway DMZ | `192.238.1.1/24` | — |
| Network 1 | `utahime` | DNS Master | `192.238.1.2/24` | `192.238.1.1` |
| Network 1 | `shoko` | DNS Slave | `192.238.1.3/24` | `192.238.1.1` |
| Network 1 | `gojo` | Public reverse proxy | `192.238.1.4/24` | `192.238.1.1` |
| Network 2 (`192.238.2.0/24`) | `jichi` (`eth2`) | Gateway / DHCP relay | `192.238.2.1/24` | — |
| Network 2 | `itadori` | Klien Divisi A, Pool 1 | DHCP | `192.238.2.1` |
| Network 2 | `fushiguro` | Klien Divisi A, Pool 1 | DHCP | `192.238.2.1` |
| Network 3 (`192.238.3.0/24`) | `jichi` (`eth3`) | Gateway / DHCP relay | `192.238.3.1/24` | — |
| Network 3 | `zenin` | Klien Divisi B, Pool 2 | DHCP | `192.238.3.1` |
| Network 3 | `okkotsu` | Klien Divisi B, Pool 2 | DHCP | `192.238.3.1` |
| Network 3 | `ino` | Klien short lease | DHCP | `192.238.3.1` |
| Network 3 | `kusakabe` | Klien short lease | DHCP | `192.238.3.1` |
| Network 3 | `miwa` | Klien long lease | DHCP | `192.238.3.1` |
| Network 3 | `kugisaki` | Klien fixed address | `192.238.3.50/24` | `192.238.3.1` |
| Network 4 (`192.238.4.0/24`) | `jichi` (`eth4`) | Gateway data center | `192.238.4.1/24` | — |
| Network 4 | `tengen` | DHCP server | `192.238.4.2/24` | `192.238.4.1` |
| Network 4 | `geto` | Internal load balancer | `192.238.4.3/24` | `192.238.4.1` |
| Network 4 | `maki` | Worker frontend | `192.238.4.10/24` | `192.238.4.1` |
| Network 4 | `panda` | Worker frontend | `192.238.4.11/24` | `192.238.4.1` |
| Network 4 | `nanami` | Worker API utama | `192.238.4.12/24` | `192.238.4.1` |
| Network 4 | `todo` | Worker API utama | `192.238.4.13/24` | `192.238.4.1` |
| Network 4 | `kirara` | Worker API cadangan | `192.238.4.14/24` | `192.238.4.1` |
| Network 4 | `yaga` | Database Master MariaDB | `192.238.4.20/24` | `192.238.4.1` |
| Network 4 | `inumaki` | Database Slave MariaDB | `192.238.4.21/24` | `192.238.4.1` |

### Rentang DHCP yang disarankan

| Network | Rentang DHCP |
|---|---|
| Network 2 | `192.238.2.100`–`192.238.2.199` |
| Network 3 | `192.238.3.100`–`192.238.3.199` |

> Pastikan `192.238.3.50` untuk `kugisaki` dikecualikan dari rentang DHCP jika alamatnya dikonfigurasi manual.

## Soal 1: Membuat jichi menjadi router antara NAT dan empat network internal, lalu menetapkan IP statis pada host sesuai tabel yang telah dibuat.

### Langkah 1: Konfigurasi jichi 

Gunakan script ini ke file /root/jichi-config.sh

```
# Ambil alamat WAN dari NAT
dhclient eth0

# Tetapkan alamat gateway untuk empat network
ip addr replace 192.238.1.1/24 dev eth1
ip addr replace 192.238.2.1/24 dev eth2
ip addr replace 192.238.3.1/24 dev eth3
ip addr replace 192.238.4.1/24 dev eth4

ip link set eth0 up
ip link set eth1 up
ip link set eth2 up
ip link set eth3 up
ip link set eth4 up

# Aktifkan IPv4 forwarding
sysctl -w net.ipv4.ip_forward=1

# Izinkan trafik diteruskan antara jaringan internal dan WAN
iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth2 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth3 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth4 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o eth1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth2 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth3 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i eth0 -o eth4 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# NAT alamat dari seluruh jaringan internal saat keluar melalui WAN
iptables -t nat -A POSTROUTING -s 192.238.0.0/16 -o eth0 -j MASQUERADE

```

Lalu atur file permissionnya:

```
chmod +x /root/jichi-config.sh
/root/jichi-config.sh
```

Agar fowarding tetap aktif setelah di reboot, simpan pengaturannya: 

```
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-jichi-router.conf
```

### Langkah 2: Tetapkan IP host infrastruktur

Gunakan script ini untuk node selain jichi di network 1 dan 4. Di script tersebut setiap node telah ditetapkan IP nya dan bisa diterapkan ke setiap node agar mengurangi pembuatan script yang hampir sama. Simpan di `/root/node-config.sh`. lalu `chmod +x /root/node-config.sh`.

```
#!/bin/sh
set -eu

NODE="$(hostname -s)"
IFACE="eth0"

set_static_ip() {
    IP="$1"
    GATEWAY="$2"

    ip link set "$IFACE" up
    ip addr replace "$IP/24" dev "$IFACE"
    ip route replace default via "$GATEWAY"
    printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf
}

# Nameserver awal sesuai ketentuan persiapan tugas
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf

case "$NODE" in
    utahime)
        set_static_ip 192.238.1.2 192.238.1.1
        ;;
    shoko)
        set_static_ip 192.238.1.3 192.238.1.1
        ;;
    gojo)
        set_static_ip 192.238.1.4 192.238.1.1
        ;;

    tengen)
        set_static_ip 192.238.4.2 192.238.4.1
        ;;
    geto)
        set_static_ip 192.238.4.3 192.238.4.1
        ;;
    maki)
        set_static_ip 192.238.4.10 192.238.4.1
        ;;
    panda)
        set_static_ip 192.238.4.11 192.238.4.1
        ;;
    nanami)
        set_static_ip 192.238.4.12 192.238.4.1
        ;;
    todo)
        set_static_ip 192.238.4.13 192.238.4.1
        ;;
    kirara)
        set_static_ip 192.238.4.14 192.238.4.1
        ;;
    yaga)
        set_static_ip 192.238.4.20 192.238.4.1
        ;;
    inumaki)
        set_static_ip 192.238.4.21 192.238.4.1
        ;;

    kugisaki)
        # Fixed address manual, di luar rentang DHCP yang direncanakan
        set_static_ip 192.238.3.50 192.238.3.1
        ;;

    itadori|fushiguro|zenin|okkotsu|ino|kusakabe|miwa)
        # Klien DHCP; lease dan pool ditentukan oleh konfigurasi DHCP server
        ip link set "$IFACE" up
        udhcpc -i "$IFACE" -q
        ;;

    jichi)
        echo "Untuk jichi, jalankan /root/jichi-config.sh"
        exit 1
        ;;

    *)
        echo "Hostname tidak dikenali: $NODE"
        exit 1
        ;;
esac

echo "Konfigurasi untuk $NODE selesai."
ip -br addr
ip route
```

Jalankan di masing-masing node:

```
/root/node-config.sh
```

Utahime:   
<img width="990" height="245" alt="image" src="https://github.com/user-attachments/assets/8f220f99-1991-405d-b305-f5774b65534e" />

Shoko:  
<img width="1021" height="247" alt="image" src="https://github.com/user-attachments/assets/3204942c-4ef1-4f34-9cba-cce08e63801e" />

Gojo:  
<img width="1018" height="250" alt="image" src="https://github.com/user-attachments/assets/2361be30-a2ac-4cd5-9ee4-d3e81513c288" />

Tengen:  
<img width="991" height="242" alt="image" src="https://github.com/user-attachments/assets/b1381f36-ae27-4a79-9769-c24acdb09199" />

Yaga:  
<img width="987" height="242" alt="image" src="https://github.com/user-attachments/assets/c8b6d667-86ed-40b7-83b9-ee6c73f46a2b" />

Maki:  
<img width="1025" height="247" alt="image" src="https://github.com/user-attachments/assets/6993bccc-889b-4145-9952-972299b13a28" />

Geto:  
<img width="1024" height="242" alt="image" src="https://github.com/user-attachments/assets/60b19ffc-2d92-4d76-9827-529eb489bef1" />

Panda:  
<img width="1024" height="246" alt="image" src="https://github.com/user-attachments/assets/5c3a87b8-960c-4952-a910-165f826106d7" />

Nanami:  
<img width="1027" height="246" alt="image" src="https://github.com/user-attachments/assets/d80bbb3e-49b1-44b6-865e-6ea4f45df848" />

Todo: 
<img width="1012" height="244" alt="image" src="https://github.com/user-attachments/assets/d7d5c071-90e5-4e54-ba42-7f3afa4e6fa3" />

Kirara:  
<img width="986" height="243" alt="image" src="https://github.com/user-attachments/assets/0805afa5-7978-42a6-af05-a49cd4059527" />

Inumaki:  
<img width="984" height="244" alt="image" src="https://github.com/user-attachments/assets/ae6dcd4e-9b8c-4d6f-a17c-e780ef5adb4a" />


## Langkah 3: Pengujian 

1. Pengujian di Jichi

   ```
    ip -br addr
    ip route
    sysctl net.ipv4.ip_forward
    iptables -t nat -L POSTROUTING -n -v
   ```

   <img width="1062" height="612" alt="image" src="https://github.com/user-attachments/assets/8a9a4573-2270-4f4c-8fe3-8e0b18f52b6c" />

    ```
    ping -c 3 192.168.122.1
    ping -c 3 1.1.1.1
    ```
    <img width="923" height="493" alt="image" src="https://github.com/user-attachments/assets/078bd472-0817-4cab-b919-1feda247cef2" />

2. Uji gateaway setiap host

   ```
    ip -br addr
    ip route
    ping -c 3 <IP-gateway>
   ```

   <img width="987" height="488" alt="image" src="https://github.com/user-attachments/assets/d003b399-b73f-4ce1-8df4-a3fdeac5c442" />

3. Uji akses eksternal dari host

   ```
    ping -c 3 1.1.1.1
    getent hosts deb.debian.org
   ```

   <img width="890" height="308" alt="image" src="https://github.com/user-attachments/assets/013df6bd-d7da-4ada-942a-6cde292121c0" />


## Soal 2: Mengonfigurasikan Tengen agar dapat mengalokasikan alamat yang membutuhkan konfigurasi dinamis 

Karena di soal 2 Tengen diminta untuk berada di IP 192.238.4.10 perlu adanya perubahan dengan IP sebelumnya.

```
ip addr replace 192.238.4.10/24 dev eth0
ip addr del 192.238.4.2/24 dev eth0
```

Script diatas merubah IP Tengen yang awalnya 192.238.4.2 menjadi 192.238.4.10 sesuai dengan yang diminta soal.  

<img width="1051" height="249" alt="image" src="https://github.com/user-attachments/assets/f95bbb73-760c-4e7e-af8c-9fef0f67cf48" />

Lalu karena IP sekarang bentrok dengan IP dari node Maki maka di IP node Maki perlu dirubah ke IP kosong disini saya memilih 192.238.4.22.

```
ip addr replace 192.238.4.22/24 dev eth0
ip addr del 192.238.4.10/24 dev eth0
```
<img width="1053" height="245" alt="image" src="https://github.com/user-attachments/assets/3d1a0721-ecd4-4308-b4b5-9ec77cd1b7cd" />

### Langkah 1: Pasang dan konfigurasi DHCP server di Tengen

```
apt-get update
apt-get install -y isc-dhcp-server
```

### Langkah 2: Validasi dan jalankan service 

```
dhcpd -t -4 -cf /etc/dhcp/dhcpd.conf
```
<img width="871" height="228" alt="image" src="https://github.com/user-attachments/assets/7aa8bffe-c704-4f77-8db1-a1b83af82494" />

Restart dan cek service:

```
service isc-dhcp-server restart
service isc-dhcp-server status
```

<img width="767" height="140" alt="image" src="https://github.com/user-attachments/assets/25c49a6a-d17a-4ad3-9336-09334071c529" />

### Langkah 3: Pengujian 

Pastikan alamat dan interface sudah sesuai dengan ketentuan yang ada di soal:

```
ip -4 addr show dev eth0
grep '^INTERFACESv4' /etc/default/isc-dhcp-server
```
<img width="1060" height="198" alt="image" src="https://github.com/user-attachments/assets/7dda26ee-9522-4468-bf7b-e5596721debb" />

Jalankan pengujian relay di jichi untuk memastikan bahwa 

```
apt-get install -y isc-dhcp-relay
nano /etc/default/isc-dhcp-relay
```

Atur nilai di file /etc/default/isc-dhcp-relay menjadi 

```
SERVERS="192.238.4.10"
INTERFACES="eth2 eth3"
OPTIONS=""
```

Simpan lalu restard dan cek relay: 

```
service isc-dhcp-relay restart
service isc-dhcp-relay status
```

Lihat apakah service relay telah berjalan: 

```
pgrep -a dhcrelay
```  
<img width="884" height="63" alt="image" src="https://github.com/user-attachments/assets/1ae17fc7-fff6-4dc5-9b60-18c4be281992" />

Uji di network 2 itadori: 

```
dhclient -v eth0
ip -4 addr show dev eth0
ip route
```

<img width="933" height="579" alt="image" src="https://github.com/user-attachments/assets/4a45680c-c3c6-4248-b250-189092e73f26" />

