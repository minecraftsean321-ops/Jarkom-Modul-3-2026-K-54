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



