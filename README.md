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
cat << 'EOF' > /root/jichi-config.sh
#!/bin/sh
set -eu

echo "=== [JICHI] Mengonfigurasi Router & Gateway ==="

# 1. Nyalakan seluruh interface (eth0 - eth4)
ip link set eth0 up
ip link set eth1 up
ip link set eth2 up
ip link set eth3 up
ip link set eth4 up

# 2. Set IP Statis WAN (eth0) langsung ke Subnet NAT GNS3 (Hanya pakai perintah 'ip' bawaan Linux)
ip addr replace 192.168.122.100/24 dev eth0
ip route replace default via 192.168.122.1
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf

# 3. Tetapkan Alamat IP Gateway internal untuk Network 1 - 4
ip addr replace 192.238.1.1/24 dev eth1
ip addr replace 192.238.2.1/24 dev eth2
ip addr replace 192.238.3.1/24 dev eth3
ip addr replace 192.238.4.1/24 dev eth4

# 4. Aktifkan IPv4 Forwarding
sysctl -w net.ipv4.ip_forward=1 > /dev/null

# 5. Konfigurasi Source NAT (MASQUERADE)
iptables -F
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

echo "=== [JICHI] Konfigurasi Berhasil Disetup ==="
EOF

chmod +x /root/jichi-config.sh

# Tambahkan ke .bashrc agar PERMANEN & otomatis jalan saat container restart
grep -qxF '/root/jichi-config.sh' /root/.bashrc || echo '/root/jichi-config.sh' >> /root/.bashrc

# Jalankan sekarang
/root/jichi-config.sh
```

Lalu atur file permissionnya:

```
chmod +x /root/jichi-config.sh
/root/jichi-config.sh
```

Agar fowarding tetap aktif setelah di reboot, simpan pengaturannya: 

```
echo '/root/jichi-config.sh' >> /root/.profile
```

### Langkah 2: Tetapkan IP host infrastruktur

Gunakan script ini untuk node selain jichi di network 1 dan 4. Di script tersebut setiap node telah ditetapkan IP nya dan bisa diterapkan ke setiap node agar mengurangi pembuatan script yang hampir sama. Simpan di `/root/node-config.sh`. lalu `chmod +x /root/node-config.sh`.

```
cat << 'EOF' > /root/node-config.sh
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

# DNS Resolver awal untuk persiapan instalasi paket
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf

case "$NODE" in
    # --- Network 1 (DMZ) ---
    utahime)
        set_static_ip 192.238.1.2 192.238.1.1
        ;;
    shoko)
        set_static_ip 192.238.1.3 192.238.1.1
        ;;
    gojo)
        set_static_ip 192.238.1.4 192.238.1.1
        ;;

    # --- Network 4 (Data Center) ---
    tengen)
        # Sesuai instruksi soal No. 2 (192.238.4.10)
        set_static_ip 192.238.4.10 192.238.4.1
        ;;
    maki)
        set_static_ip 192.238.4.2 192.238.4.1
        ;;
    geto)
        set_static_ip 192.238.4.3 192.238.4.1
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

    # --- Network 3 (Fixed Address) ---
    kugisaki)
        set_static_ip 192.238.3.50 192.238.3.1
        ;;

    # --- Client DHCP (Divisi A & B) ---
    itadori|fushiguro|zenin|okkotsu|ino|kusakabe|miwa)
        ip link set "$IFACE" up
        ;;

    jichi)
        echo "Untuk jichi, gunakan script /root/jichi-config.sh"
        exit 0
        ;;

    *)
        echo "Hostname tidak dikenali: $NODE"
        exit 1
        ;;
esac

echo "Konfigurasi $NODE selesai."
EOF

chmod +x /root/node-config.sh
```

Agar bisa otomatis ter boot ketika restart daftarkan ke /root/.bashrc dan /root/.profile:

```
# Tambahkan ke .bashrc dan .profile
grep -qxF '/root/node-config.sh' /root/.bashrc || echo '/root/node-config.sh' >> /root/.bashrc
grep -qxF '/root/node-config.sh' /root/.profile || echo '/root/node-config.sh' >> /root/.profile

# Jalankan script-nya sekarang
/root/node-config.sh
```

## Langkah 3: Pengujian di node Tengen

1. Cek IP & Default Route:

   ```
    ip -br addr
    ip route
   ```

   <img width="1003" height="249" alt="image" src="https://github.com/user-attachments/assets/8d006b5d-8134-4b9c-8610-43e0de1c3f8e" />

2. Cek koneksi Internet 
    ```
    ping -c 2 8.8.8.8
    ping -c 2 deb.debian.org
    ```
    <img width="1058" height="436" alt="image" src="https://github.com/user-attachments/assets/5f4291d9-2774-41ce-89a9-d00d58a87018" />

## Soal 2: Mengonfigurasikan Tengen agar dapat mengalokasikan alamat yang membutuhkan konfigurasi dinamis 

### Langkah 1: Pasang dan konfigurasi DHCP server di Tengen

```
apt-get update && apt-get install -y isc-dhcp-server
```

<img width="1063" height="645" alt="image" src="https://github.com/user-attachments/assets/8f88c97e-a2aa-4a85-92b1-2b80afbfc39a" />

### Langkah 2: Konfigurasi DHCP Server (`/root/tengen-dhcp.sh`);

```
cat << 'EOF' > /root/tengen-dhcp.sh
#!/bin/sh
set -eu

echo "=== Mengonfigurasi isc-dhcp-server di Tengen ==="

# 1. Atur interface listening ke eth0
cat << 'DEFAULT_CONF' > /etc/default/isc-dhcp-server
INTERFACESv4="eth0"
INTERFACESv6=""
DEFAULT_CONF

# 2. Buat konfigurasi utama DHCP Server
cat << 'DHCPD_CONF' > /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

# DNS Name Server
option domain-name-servers 192.238.1.2, 192.238.1.3;

# -------------------------------------------------------------
# Network 4: Data Center (192.238.4.0/24)
# Subnet lokal tempat tengen berada (192.238.4.10)
# Wajib dideklarasikan TANPA pool dinamis (tanpa range)
# agar tidak muncul error "No subnet declaration for eth0"
# -------------------------------------------------------------
subnet 192.238.4.0 netmask 255.255.255.0 {
    option routers 192.238.4.1;
    option broadcast-address 192.238.4.255;
}

# -------------------------------------------------------------
# Network 2: Scope Divisi A (192.238.2.0/24)
# -------------------------------------------------------------
subnet 192.238.2.0 netmask 255.255.255.0 {
    range 192.238.2.10 192.238.2.100;
    option routers 192.238.2.1;
    option broadcast-address 192.238.2.255;
}

# -------------------------------------------------------------
# Network 3: Scope Divisi B (192.238.3.0/24)
# -------------------------------------------------------------
subnet 192.238.3.0 netmask 255.255.255.0 {
    range 192.238.3.10 192.238.3.100;
    option routers 192.238.3.1;
    option broadcast-address 192.238.3.255;
}
DHCPD_CONF

# 3. Uji Sintaks Konfigurasi
/usr/sbin/dhcpd -t -cf /etc/dhcp/dhcpd.conf

# 4. Restart Service DHCP Server
service isc-dhcp-server restart || /etc/init.d/isc-dhcp-server restart || true

echo "=== Konfigurasi Tengen Selesai ==="
EOF

chmod +x /root/tengen-dhcp.sh
/root/tengen-dhcp.sh
```

<img width="1064" height="494" alt="image" src="https://github.com/user-attachments/assets/07578645-6c68-48d5-a646-f903db0083ef" />

Agar script diatas bisa berjalan otomatis saat ter reboot:

```
# Tambahkan perintah restart service DHCP di akhir file /root/node-config.sh
echo 'service isc-dhcp-server restart || /etc/init.d/isc-dhcp-server restart || true' >> /root/node-config.sh
```

Bagaimana Cara Kerja Alurnya Saat Reboot?
1. Saat container tengen di-restart, /root/.profile / /root/.bashrc akan mengeksekusi /root/node-config.sh.
2. Script /root/node-config.sh mengatur IP statis 192.238.4.10 dan gateway 192.238.4.1.
3. Setelah IP aktif, baris tambahan di atas akan langsung menyalakan daemon isc-dhcp-server.   

### Langkah 3: Pengujian 

1. Uji Sintaks File Configuration:

```
/usr/sbin/dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

<img width="1063" height="229" alt="image" src="https://github.com/user-attachments/assets/4978f3de-ece0-4372-a0f5-60386cc99114" />

Terlihat bahwa tidak ada error

2. Cek Log / Status Service:

```
service isc-dhcp-server status
```

<img width="773" height="92" alt="image" src="https://github.com/user-attachments/assets/853186bb-35a5-49be-9ba7-35dd4671fa9d" />

```
ps aux | grep dhcpd
```
Harus muncul baris proses `/usr/sbin/dhcpd` yang sedang berjalan.

<img width="1062" height="118" alt="image" src="https://github.com/user-attachments/assets/6da4a23e-4398-420d-9367-8374eba7a181" />

```
/usr/sbin/dhcpd -f -d eth0
```  
<img width="1067" height="370" alt="image" src="https://github.com/user-attachments/assets/52eda09a-112a-43cb-b3ad-1cfa94384259" />


