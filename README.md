# Jarkom-Modul-3-2026-K-54

| Nama                 | NRP        |
|----------------------|------------|
| Viko Rizky Fauzan    | 5027251017 |
| Sean Arthur Tamajaya | 5027251050 |

## Tabel IP Host

Asumsi prefix kelompok adalah `192.238`. Semua interface internal menggunakan subnet mask `/24`.

| Network | Node | Peran / Fungsi | Alamat IP | Default Gateway |
|---|---|---|---|---|
| **NAT** | `jichi` (`eth0`) | Koneksi WAN ke NAT GNS3 | `192.168.122.100/24` | `192.168.122.1` |
| **Network 1** (`192.238.1.0/24`) | `jichi` (`eth1`) | Gateway DMZ | `192.238.1.1/24` | — |
| Network 1 | `utahime` | DNS Master | `192.238.1.2/24` | `192.238.1.1` |
| Network 1 | `shoko` | DNS Slave | `192.238.1.3/24` | `192.238.1.1` |
| Network 1 | `gojo` | Public Reverse Proxy | `192.238.1.4/24` | `192.238.1.1` |
| **Network 2** (`192.238.2.0/24`) | `jichi` (`eth2`) | Gateway / DHCP Relay Network 2 | `192.238.2.1/24` | — |
| Network 2 | `itadori` | Klien Divisi A (Pool 1 - Dev) | DHCP (`.2.10`–`.2.35`) | `192.238.2.1` |
| Network 2 | `fushiguro` | Klien Divisi A (Pool 1 - Dev) | DHCP (`.2.10`–`.2.35`) | `192.238.2.1` |
| Network 2 | `zenin` | Klien Divisi A (Pool 2 - Support) | DHCP (`.2.70`–`.2.95`) | `192.238.2.1` |
| Network 2 | `okkotsu` | Klien Divisi A (Pool 2 - Support) | DHCP (`.2.70`–`.2.95`) | `192.238.2.1` |
| **Network 3** (`192.238.3.0/24`) | `jichi` (`eth3`) | Gateway / DHCP Relay Network 3 | `192.238.3.1/24` | — |
| Network 3 | `ino` | Klien Divisi B (Short Lease - 30m) | DHCP (`.3.10`–`.3.89`) | `192.238.3.1` |
| Network 3 | `kusakabe` | Klien Divisi B (Short Lease - 30m) | DHCP (`.3.10`–`.3.89`) | `192.238.3.1` |
| Network 3 | `miwa` | Klien Divisi B (Long Lease - 12j) | DHCP (`.3.100`–`.3.199`) | `192.238.3.1` |
| Network 3 | `kugisaki` | Klien Fixed Address | `192.238.3.50/24` (DHCP Fixed) | `192.238.3.1` |
| **Network 4** (`192.238.4.0/24`) | `jichi` (`eth4`) | Gateway Data Center | `192.238.4.1/24` | — |
| Network 4 | `tengen` | DHCP Server Utama | `192.238.4.10/24` | `192.238.4.1` |
| Network 4 | `geto` | Internal Load Balancer | `192.238.4.3/24` | `192.238.4.1` |
| Network 4 | `maki` | Worker Frontend | `192.238.4.2/24` | `192.238.4.1` |
| Network 4 | `panda` | Worker Frontend | `192.238.4.11/24` | `192.238.4.1` |
| Network 4 | `nanami` | Worker API Utama | `192.238.4.12/24` | `192.238.4.1` |
| Network 4 | `todo` | Worker API Utama | `192.238.4.13/24` | `192.238.4.1` |
| Network 4 | `kirara` | Worker API Cadangan | `192.238.4.14/24` | `192.238.4.1` |
| Network 4 | `yaga` | Database Master MariaDB | `192.238.4.20/24` | `192.238.4.1` |
| Network 4 | `inumaki` | Database Slave MariaDB | `192.238.4.21/24` | `192.238.4.1` |

### Rentang DHCP yang disarankan

| Subnet | Alokasi Pool / Scope | Rentang Alamat IP | Pengguna Target | Keterangan / Kebijakan |
|---|---|---|---|---|
| **Network 2** (`192.238.2.0/24`) | Pool 1 (Dev) | `192.238.2.10` – `192.238.2.35` | `itadori`, `fushiguro` | Dibatasi via Class MAC Address |
| | Pool Exclude | `192.238.2.36` – `192.238.2.69` | — | Dikosongkan / Tidak Dialokasikan |
| | Pool 2 (Support) | `192.238.2.70` – `192.238.2.95` | `zenin`, `okkotsu` | Dibatasi via Class MAC Address |
| **Network 3** (`192.238.3.0/24`) | Short Lease Pool | `192.238.3.10` – `192.238.3.89` | `ino`, `kusakabe` | Lease Time: 30 Menit (1800d) |
| | Fixed Address | `192.238.3.50` | `kugisaki` | Reservasi IP Khusus |
| | Long Lease Pool | `192.238.3.100` – `192.238.3.199` | `miwa` | Lease Time: 12 Jam (43200d) |

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

### Langkah 3: Pengujian di node Tengen

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

## Soal 3: Konfigurasi DHCP Relay di Router `jichi`
jichi bertindak sebagai perantara (relay) yang meneruskan request DHCP dari Network 2 (eth2) dan Network 3 (eth3) menuju DHCP Server tengen (192.238.4.10 via eth4).

### Langkah 1: Konfigurasi Node Jichi 

```
cat << 'EOF' > /root/jichi-config.sh
#!/bin/sh
set -eu

echo "=== [JICHI] Mengonfigurasi Router, NAT, & DHCP Relay ==="

# 1. Aktifkan seluruh interface
ip link set eth0 up
ip link set eth1 up
ip link set eth2 up
ip link set eth3 up
ip link set eth4 up

# 2. IP Statis WAN ke Subnet NAT GNS3
ip addr replace 192.168.122.100/24 dev eth0
ip route replace default via 192.168.122.1
printf 'nameserver 192.168.122.1\n' > /etc/resolv.conf

# 3. IP Gateway Internal
ip addr replace 192.238.1.1/24 dev eth1
ip addr replace 192.238.2.1/24 dev eth2
ip addr replace 192.238.3.1/24 dev eth3
ip addr replace 192.238.4.1/24 dev eth4

# 4. IP Forwarding & Source NAT
sysctl -w net.ipv4.ip_forward=1 > /dev/null
iptables -F
iptables -t nat -F
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# 5. Instalasi isc-dhcp-relay jika belum terpasang
if ! command -v dhcrelay > /dev/null 2>&1; then
    echo "[*] Mengunduh isc-dhcp-relay..."
    apt-get update && apt-get install -y isc-dhcp-relay
fi

# 6. Konfigurasi /etc/default/isc-dhcp-relay
cat << 'RELAY_CONF' > /etc/default/isc-dhcp-relay
SERVERS="192.238.4.10"
INTERFACES="eth2 eth3 eth4"
OPTIONS=""
RELAY_CONF

# 7. Jalankan Service DHCP Relay
service isc-dhcp-relay restart || /etc/init.d/isc-dhcp-relay restart || true

echo "=== [JICHI] Setup Berhasil Selesai ==="
EOF

chmod +x /root/jichi-config.sh

# Daftarkan ke .profile dan .bashrc agar OTOMATIS RUNNING SAAT REBOOT
grep -qxF '/root/jichi-config.sh' /root/.profile || echo '/root/jichi-config.sh' >> /root/.profile
grep -qxF '/root/jichi-config.sh' /root/.bashrc || echo '/root/jichi-config.sh' >> /root/.bashrc

# Jalankan script sekarang
/root/jichi-config.sh
```

Verifikasi Langkah 1:

```
ps aux | grep dhcrelay
```
Pastikan terlihat proses dhcrelay berjalan mengarah ke 192.238.4.10.

<img width="1060" height="121" alt="image" src="https://github.com/user-attachments/assets/8980c94a-e1e3-4ff5-b21b-9987fd4a894b" />

### Langkah 2: Pengujian 

1. Uji DHCP Relay di klien Network 2 dan Network 3

   Itadori (Network 2) & Ino (Network 3):

   ```
    udhcpc -i eth0 -q
   ```
   <img width="985" height="144" alt="image" src="https://github.com/user-attachments/assets/ebacfee1-73c1-4a03-8f25-c643e9aca90e" />
   
   <img width="1022" height="141" alt="image" src="https://github.com/user-attachments/assets/1ed063e1-1098-4019-8cf9-126786eedc17" />

   Cek IP yang diterima:

   ```
    ip -br addr
    ip route
   ```

   <img width="988" height="251" alt="image" src="https://github.com/user-attachments/assets/32c8c958-f5e0-4039-a51c-acad3571f48d" />

   <img width="1032" height="249" alt="image" src="https://github.com/user-attachments/assets/523eb783-fb33-4709-a39a-190e4f28eb1f" />

2. Cek record lease di DHCP Server (Tengen)

   ```
    cat /var/lib/dhcp/dhcpd.leases
   ```

   <img width="1058" height="627" alt="image" src="https://github.com/user-attachments/assets/c58a229b-234d-4500-8029-525371ae724f" />

## Soal 4: DHCP Class & Dual Pool di Node

| Nama Node | Kelompok / Class | MAC Address | Rentang IP Pool |
|---|---|---|---|
| **itadori** | Dev (`pool1_dev`) | `02:42:ba:30:f3:00` | `192.238.2.10` – `192.238.2.35` |
| **fushiguro** | Dev (`pool1_dev`) | `02:42:93:96:61:00` | `192.238.2.10` – `192.238.2.35` |
| **zenin** | Support (`pool2_support`) | `02:42:02:87:12:00` | `192.238.2.70` – `192.238.2.95` |
| **okkotsu** | Support (`pool2_support`) | `02:42:89:e7:e4:00` | `192.238.2.70` – `192.238.2.95` |

### Langkah 1: Eksekusi script di Node Tengen

```
cat << 'EOF' > /root/tengen-dhcp.sh
#!/bin/sh
set -eu

echo "=== [TENGEN] Mengonfigurasi DHCP Server dengan Class MAC Address ==="

cat << 'DEFAULT_CONF' > /etc/default/isc-dhcp-server
INTERFACESv4="eth0"
INTERFACESv6=""
DEFAULT_CONF

cat << 'DHCPD_CONF' > /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

option domain-name-servers 192.238.1.2, 192.238.1.3;

# -------------------------------------------------------------
# Kelas Client berdasarkan MAC Address
# -------------------------------------------------------------
class "pool1_dev" {
    match hardware;
}
subclass "pool1_dev" 1:02:42:ba:30:f3:00;
subclass "pool1_dev" 1:02:42:93:96:61:00;

class "pool2_support" {
    match hardware;
}
subclass "pool2_support" 1:02:42:02:87:12:00;
subclass "pool2_support" 1:02:42:89:e7:e4:00;

# -------------------------------------------------------------
# Network 4: Data Center (Subnet Lokal Tengen)
# -------------------------------------------------------------
subnet 192.238.4.0 netmask 255.255.255.0 {
    option routers 192.238.4.1;
    option broadcast-address 192.238.4.255;
}

# -------------------------------------------------------------
# Network 2: Scope Divisi A (Dual Pool dengan Class)
# -------------------------------------------------------------
subnet 192.238.2.0 netmask 255.255.255.0 {
    option routers 192.238.2.1;
    option broadcast-address 192.238.2.255;

    # Pool 1: Dev (Itadori & Fushiguro)
    pool {
        allow members of "pool1_dev";
        range 192.238.2.10 192.238.2.35;
    }

    # Pool 2: Support (Zenin & Okkotsu)
    pool {
        allow members of "pool2_support";
        range 192.238.2.70 192.238.2.95;
    }
}

# -------------------------------------------------------------
# Network 3: Scope Divisi B
# -------------------------------------------------------------
subnet 192.238.3.0 netmask 255.255.255.0 {
    range 192.238.3.10 192.238.3.100;
    option routers 192.238.3.1;
    option broadcast-address 192.238.3.255;
}
DHCPD_CONF

# Validasi sintaks
/usr/sbin/dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Restart Service
service isc-dhcp-server restart || /etc/init.d/isc-dhcp-server restart || true

echo "=== [TENGEN] Setup Soal 4 Selesai ==="
EOF

chmod +x /root/tengen-dhcp.sh

# Daftarkan ke file startup agar otomatis running saat restart
grep -qxF '/root/tengen-dhcp.sh' /root/.profile || echo '/root/tengen-dhcp.sh' >> /root/.profile
grep -qxF '/root/tengen-dhcp.sh' /root/.bashrc || echo '/root/tengen-dhcp.sh' >> /root/.bashrc

# Eksekusi script
/root/tengen-dhcp.sh
```

### Langkah 2: Pengujian di Divisi A 

1. Di itadori atau fushiguro:

   ```
    udhcpc -i eth0 -q
    ip -br addr
   ```
   Harus menerima IP di rentang 192.238.2.10–192.238.2.35.

   <img width="989" height="302" alt="image" src="https://github.com/user-attachments/assets/0f66b4ed-9cac-4a27-b253-57959c62c9f2" />

   <img width="991" height="300" alt="image" src="https://github.com/user-attachments/assets/3b624427-24e1-4f14-b6d3-9a8f10d1a9ec" />


2. Di zenin atau okkotsu:

   ```
    udhcpc -i eth0 -q
    ip -br addr
   ```
   Harus menerima IP di rentang 192.238.2.70–192.238.2.95.

   <img width="979" height="303" alt="image" src="https://github.com/user-attachments/assets/0a09cecd-6725-4913-8ea2-d6bc769c2d78" />

   <img width="987" height="294" alt="image" src="https://github.com/user-attachments/assets/9edd683d-1495-4383-abb7-4c1a3d69251b" />

## Soal 5: Mengatur durasi lease di Network 3 menggunakan DHCP Class berdasarkan MAC Address

- Short Lease (30 menit = 1800 detik): Pool 192.238.3.10–192.238.3.89 untuk ino dan kusakabe.   
- Long Lease (12 jam = 43200 detik): Pool 192.238.3.100–192.238.3.199 untuk miwa.
- DHCP Options: DNS 192.238.1.2, 192.238.1.3 dan domain-search.

### Langkah 1: Cek MAC Address Klien Network 3 (kusakabe & miwa) 

```
ip link show eth0
```

<img width="1059" height="115" alt="image" src="https://github.com/user-attachments/assets/df74653f-68b0-42be-a29d-91906b43699f" />

<img width="1061" height="116" alt="image" src="https://github.com/user-attachments/assets/87304dca-ae50-41f4-b98d-cf1a2466b9ac" />

<img width="1061" height="112" alt="image" src="https://github.com/user-attachments/assets/fa3e02c1-231c-4e07-97f7-ce89f2697915" />  


| Nama Node | Kelompok / Class | MAC Address | Rentang IP Pool | Default / Max Lease Time |
|---|---|---|---|---|
| **ino** | Short Lease (`short_lease`) | `02:42:a5:41:3b:00` | `192.238.3.10` – `192.238.3.89` | 1800 detik (30 menit) |
| **kusakabe** | Short Lease (`short_lease`) | `02:42:89:2f:5f:00` | `192.238.3.10` – `192.238.3.89` | 1800 detik (30 menit) |
| **miwa** | Long Lease (`long_lease`) | `02:42:ea:de:c4:00` | `192.238.3.100` – `192.238.3.199` | 43200 detik (12 jam) |


### Langkah 2:  **Script Akumulatif `/root/tengen-dhcp.sh` di Node `tengen`**

Jalankan seluruh blok kode berikut di terminal node **`tengen`**:

```bash
cat << 'EOF' > /root/tengen-dhcp.sh
#!/bin/sh
set -eu

echo "=== [TENGEN] Mengonfigurasi DHCP Server (Soal 1-5 Akumulatif) ==="

# =========================================================================
# MAC ADDRESS CLIENT NETWORK 3
# =========================================================================
MAC_INO="02:42:a5:41:3b:00"
MAC_KUSAKABE="02:42:89:2f:5f:00"
MAC_MIWA="02:42:ea:de:c4:00"

DOMAIN_NAME="jarkom.com"
# =========================================================================

cat << 'DEFAULT_CONF' > /etc/default/isc-dhcp-server
INTERFACESv4="eth0"
INTERFACESv6=""
DEFAULT_CONF

cat << DHCPD_CONF > /etc/dhcp/dhcpd.conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

# DHCP Options Global (Soal 5)
option domain-name-servers 192.238.1.2, 192.238.1.3;
option domain-name "${DOMAIN_NAME}";
option domain-search "${DOMAIN_NAME}";

# -------------------------------------------------------------
# CLASS DIVISI A - NETWORK 2 (Soal 4)
# -------------------------------------------------------------
class "pool1_dev" {
    match hardware;
}
subclass "pool1_dev" 1:02:42:ba:30:f3:00;
subclass "pool1_dev" 1:02:42:93:96:61:00;

class "pool2_support" {
    match hardware;
}
subclass "pool2_support" 1:02:42:02:87:12:00;
subclass "pool2_support" 1:02:42:89:e7:e4:00;

# -------------------------------------------------------------
# CLASS DIVISI B - NETWORK 3 (Soal 5)
# -------------------------------------------------------------
class "short_lease" {
    match hardware;
}
subclass "short_lease" 1:${MAC_INO};
subclass "short_lease" 1:${MAC_KUSAKABE};

class "long_lease" {
    match hardware;
}
subclass "long_lease" 1:${MAC_MIWA};

# -------------------------------------------------------------
# Network 4: Data Center (Subnet Lokal Tengen)
# -------------------------------------------------------------
subnet 192.238.4.0 netmask 255.255.255.0 {
    option routers 192.238.4.1;
    option broadcast-address 192.238.4.255;
}

# -------------------------------------------------------------
# Network 2: Scope Divisi A (Dual Pool dengan Class - Soal 4)
# -------------------------------------------------------------
subnet 192.238.2.0 netmask 255.255.255.0 {
    option routers 192.238.2.1;
    option broadcast-address 192.238.2.255;

    # Pool 1: Dev (Itadori & Fushiguro)
    pool {
        allow members of "pool1_dev";
        range 192.238.2.10 192.238.2.35;
    }

    # Pool 2: Support (Zenin & Okkotsu)
    pool {
        allow members of "pool2_support";
        range 192.238.2.70 192.238.2.95;
    }
}

# -------------------------------------------------------------
# Network 3: Scope Divisi B (Short & Long Lease Pool - Soal 5)
# -------------------------------------------------------------
subnet 192.238.3.0 netmask 255.255.255.0 {
    option routers 192.238.3.1;
    option broadcast-address 192.238.3.255;

    # Pool Short Lease: Ino & Kusakabe (30 Menit = 1800 detik)
    pool {
        allow members of "short_lease";
        range 192.238.3.10 192.238.3.89;
        default-lease-time 1800;
        max-lease-time 1800;
    }

    # Pool Long Lease: Miwa (12 Jam = 43200 detik)
    pool {
        allow members of "long_lease";
        range 192.238.3.100 192.238.3.199;
        default-lease-time 43200;
        max-lease-time 43200;
    }
}
DHCPD_CONF

# Validasi sintaks
/usr/sbin/dhcpd -t -cf /etc/dhcp/dhcpd.conf

# Restart Service
service isc-dhcp-server restart || /etc/init.d/isc-dhcp-server restart || true

echo "=== [TENGEN] Setup Berhasil Diperbarui ==="
EOF

chmod +x /root/tengen-dhcp.sh

# Garansi Otomatis Reboot
grep -qxF '/root/tengen-dhcp.sh' /root/.profile || echo '/root/tengen-dhcp.sh' >> /root/.profile
grep -qxF '/root/tengen-dhcp.sh' /root/.bashrc || echo '/root/tengen-dhcp.sh' >> /root/.bashrc

# Jalankan script
/root/tengen-dhcp.sh

```

### Langkah 3: Pengujian 

1. Di ino & kusakabe

   ```
    udhcpc -i eth0 -q
   ```
   IP berada di rentang 192.238.3.10 – 192.238.3.89 dengan keterangan lease time 1800.

   <img width="1011" height="145" alt="image" src="https://github.com/user-attachments/assets/764c1242-3790-4ae8-89ea-8bc1ad28852f" />

   <img width="1011" height="139" alt="image" src="https://github.com/user-attachments/assets/f4cbbc85-4af6-4b91-91f8-b4bbfc050f8b" />

    
   
2. Di miwa

   ```
    udhcpc -i eth0 -q
   ```

   <img width="1026" height="145" alt="image" src="https://github.com/user-attachments/assets/78a07fe0-0fc2-4d1f-aba6-610fcea39307" />

3. Cek DNS & Domain Search pada klien Network 3

   ```
    cat /etc/resolv.conf
   ```

   Menampilkan nameserver 192.238.1.2, nameserver 192.238.1.3, dan search jarkom.com

   <img width="447" height="109" alt="image" src="https://github.com/user-attachments/assets/0f12dc13-cad0-4c74-afd8-e16164f2f6b0" />





