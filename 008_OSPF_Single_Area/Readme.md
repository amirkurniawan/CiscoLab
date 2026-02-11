# 008 — OSPF Single Area

## 🎯 Tujuan Lab

Mengkonfigurasi OSPF (Open Shortest Path First) agar router **otomatis belajar route** tanpa perlu config static route satu-satu. Dengan OSPF, tambah router baru = semua router langsung tahu — tanpa config manual di mana-mana.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **OSPF** | Dynamic routing protocol — router saling tukar info route otomatis |
| **Area** | Pengelompokan router OSPF. Area 0 = backbone (wajib ada) |
| **Router ID** | Identitas unik router di OSPF (format IP, tapi bukan IP) |
| **Neighbor** | Router lain yang sudah establish hubungan OSPF |
| **Adjacency** | Status neighbor yang sudah FULL (sync database route) |
| **Hello Packet** | Paket yang dikirim berkala untuk discover & maintain neighbor |
| **Cost** | Metric OSPF — bandwidth makin besar, cost makin kecil (path terbaik) |
| **LSA** | Link-State Advertisement — info route yang disebarkan antar router |
| **SPF Algorithm** | Dijkstra's algorithm — hitung shortest path ke semua tujuan |

### Kenapa OSPF, Bukan Static Route?

| | Static Route (Lab 007) | OSPF (Lab 008) |
|---|---|---|
| Tambah network baru | Config manual di **semua** router | Otomatis disebarkan |
| Failover | Lambat (tunggu interface down) | Cepat (detect neighbor hilang) |
| 5 router | 20+ static route | Config OSPF di tiap router, selesai |
| 20 router | 380+ static route 💀 | Sama — config OSPF di tiap router |
| Tahu kondisi network | Cuma cek next-hop langsung | Tahu seluruh topology |

### Bagaimana OSPF Bekerja

```
Step 1: HELLO — Router saling discover neighbor
  R1 ──hello──→ R2    "Halo, saya R1, area 0"
  R1 ←──hello── R2    "Halo, saya R2, area 0"

Step 2: EXCHANGE — Tukar database route
  R1 ←──LSA──→ R2     "Saya punya network 192.168.1.0"
                       "Saya punya network 192.168.2.0"

Step 3: CALCULATE — Hitung shortest path (SPF/Dijkstra)
  R1: "Ke 192.168.2.0 lewat R2, cost 1"
  R2: "Ke 192.168.1.0 lewat R1, cost 1"

Step 4: ROUTING TABLE — Otomatis terisi
  R1: O 192.168.2.0/24 [110/2] via 10.0.0.2
  R2: O 192.168.1.0/24 [110/2] via 10.0.0.1
```

### OSPF Neighbor States

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
                                                       ↑
                                              ini yang kita mau
```

Kalau neighbor stuck di **2-Way** atau **ExStart**, ada masalah config. Harus sampai **Full**.

---

## 🏢 Skenario

PT. Nusantara Digital sekarang punya 3 kantor: **Jakarta, Surabaya, dan Malang**. Semua terhubung dan butuh routing otomatis.

Dengan 3 router, kalau pakai static route butuh **6 route config**. Dengan OSPF, cukup **config OSPF di tiap router** — selesai. Kalau nanti tambah kantor Bali, tinggal config OSPF di router Bali, semua router lain otomatis tahu.

---

## 🌐 Topologi Network di Production

Sebelum masuk lab, penting untuk paham topologi apa saja yang umum dipakai di production dan kenapa kita pilih segitiga untuk lab ini.

### 1. Hub-and-Spoke (Star) — Paling Umum

```
             [Branch-1]
                  │
                  │ WAN
                  │
[Branch-3] ──── [HQ] ──── [Branch-2]
                  │
                  │ WAN
                  │
             [Branch-4]
```

| Kelebihan | Kekurangan |
|-----------|------------|
| Murah — setiap cabang cuma 1 link | HQ mati = semua cabang terisolasi |
| Simple config | Branch-1 ke Branch-2 harus lewat HQ |
| Mudah di-manage | Single point of failure di HQ |

**Dipakai oleh:** Mayoritas perusahaan Indonesia. Bank cabang, retail chain, kantor cabang pemerintahan.

### 2. Partial Mesh — Balance Cost & Redundancy

```
[Branch-1] ════ [HQ] ════ [Branch-2]
                 ║               ║
            [Branch-3] ════ [Branch-2]
```

| Kelebihan | Kekurangan |
|-----------|------------|
| Ada jalur alternatif | Lebih mahal dari hub-and-spoke |
| Kalau 1 link mati, masih ada backup | Config lebih complex |
| Critical site punya redundancy | Tidak semua site punya backup |

**Dipakai oleh:** Perusahaan menengah-besar yang butuh uptime tinggi di lokasi critical.

### 3. Full Mesh — Maximum Redundancy

```
[Site-1] ════ [Site-2]
   ║      ╲╱      ║
   ║      ╱╲      ║
[Site-3] ════ [Site-4]
```

| Kelebihan | Kekurangan |
|-----------|------------|
| Zero single point of failure | Sangat mahal (banyak link WAN) |
| Direct path antar semua site | Config complex |
| Fastest failover | Jumlah link = n(n-1)/2 |

**Contoh biaya:** 4 site = 6 link, 10 site = 45 link, 20 site = 190 link. Makanya biasanya cuma dipakai di **core layer** atau **data center interconnect**, bukan di semua site.

**Dipakai oleh:** Data center, enterprise core network, financial institution.

### 4. Ring — Redundancy Murah

```
[Site-1] ──── [Site-2]
   │                │
   │                │
[Site-4] ──── [Site-3]
```

| Kelebihan | Kekurangan |
|-----------|------------|
| Setiap site punya 2 path | Kalau 2 link mati, network split |
| Lebih murah dari full mesh | Latency tinggi untuk site yang jauh |
| Redundancy sederhana | Bandwidth shared |

**Dipakai oleh:** ISP backbone, metro ethernet, beberapa campus network.

### Kenapa Lab Ini Pakai Segitiga (Partial Mesh)?

```
        [R-JKT]
        ╱     ╲
   [R-SBY] ── [R-MLG]
```

Alasan kita pilih topologi segitiga:

| Alasan | Penjelasan |
|--------|------------|
| **Redundancy** | Setiap router punya 2 jalur — kalau 1 mati, masih bisa lewat jalur lain |
| **Failover demo** | Bisa matikan 1 link dan buktikan OSPF auto-reroute |
| **Realistis** | Partial mesh adalah topologi paling umum di production setelah hub-and-spoke |
| **Tidak terlalu complex** | 3 router = cukup untuk demo OSPF tanpa overwhelming |
| **Cost calculation** | Bisa lihat OSPF pilih direct path (cost kecil) vs indirect path (cost besar) |

> 💡 **OSPF tidak terikat topologi tertentu.** Semua topologi di atas bisa pakai OSPF. Yang membedakan hanyalah **jumlah neighbor** yang di-discover dan **jumlah path** yang tersedia untuk failover.

---

## 📋 IP Addressing Table

### LAN Segments

| Lokasi | Network | Subnet Mask | Gateway |
|--------|---------|-------------|---------|
| Jakarta LAN | 192.168.1.0/24 | 255.255.255.0 | 192.168.1.1 |
| Surabaya LAN | 192.168.2.0/24 | 255.255.255.0 | 192.168.2.1 |
| Malang LAN | 192.168.3.0/24 | 255.255.255.0 | 192.168.3.1 |

### WAN Links (Point-to-Point /30)

| Link | Network | Router A IP | Router B IP |
|------|---------|-------------|-------------|
| JKT ↔ SBY | 10.0.0.0/30 | 10.0.0.1 (R-JKT) | 10.0.0.2 (R-SBY) |
| SBY ↔ MLG | 10.0.1.0/30 | 10.0.1.1 (R-SBY) | 10.0.1.2 (R-MLG) |
| JKT ↔ MLG | 10.0.2.0/30 | 10.0.2.1 (R-JKT) | 10.0.2.2 (R-MLG) |

### Device IP Summary

| Device | Interface | IP Address | Subnet Mask |
|--------|-----------|------------|-------------|
| R-JKT | Gi0/0 | 192.168.1.1 | 255.255.255.0 |
| R-JKT | Se0/0/0 | 10.0.0.1 | 255.255.255.252 |
| R-JKT | Se0/0/1 | 10.0.2.1 | 255.255.255.252 |
| R-SBY | Gi0/0 | 192.168.2.1 | 255.255.255.0 |
| R-SBY | Se0/0/0 | 10.0.0.2 | 255.255.255.252 |
| R-SBY | Se0/0/1 | 10.0.1.1 | 255.255.255.252 |
| R-MLG | Gi0/0 | 192.168.3.1 | 255.255.255.0 |
| R-MLG | Se0/0/0 | 10.0.1.2 | 255.255.255.252 |
| R-MLG | Se0/0/1 | 10.0.2.2 | 255.255.255.252 |
| PC-JKT | NIC | 192.168.1.10 | 255.255.255.0 |
| PC-SBY | NIC | 192.168.2.10 | 255.255.255.0 |
| PC-MLG | NIC | 192.168.3.10 | 255.255.255.0 |

---

## 🖥️ Topologi

```
                        [R-JKT]
                    Gi0/0  Se0/0/0  Se0/0/1
                      │       │        │
                   [SW-JKT]   │        │
                      │       │        │
                   [PC-JKT]   │        │
                  192.168.1.0 │        │
                              │        │
                    10.0.0.0/30       10.0.2.0/30
                              │        │
                    Se0/0/0   │        │   Se0/0/1
                        [R-SBY]        [R-MLG]
                    Gi0/0  Se0/0/1──Se0/0/0  Gi0/0
                      │       10.0.1.0/30       │
                   [SW-SBY]                  [SW-MLG]
                      │                         │
                   [PC-SBY]                  [PC-MLG]
                  192.168.2.0              192.168.3.0
```

> 💡 **Topologi segitiga** — setiap router terhubung ke 2 router lain. Ini memberikan redundancy: kalau link JKT↔SBY mati, traffic bisa lewat JKT→MLG→SBY.

### Perangkat

| Device | Model | Hostname | Module |
|--------|-------|----------|--------|
| Router 1 | 2911 | R-JKT | HWIC-2T |
| Router 2 | 2911 | R-SBY | HWIC-2T |
| Router 3 | 2911 | R-MLG | HWIC-2T |
| Switch 1 | 2960 | SW-JKT | - |
| Switch 2 | 2960 | SW-SBY | - |
| Switch 3 | 2960 | SW-MLG | - |
| PC 1 | - | PC-JKT | - |
| PC 2 | - | PC-SBY | - |
| PC 3 | - | PC-MLG | - |

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Pasang module HWIC-2T di semua router
Step 2: Hostname semua device
Step 3: Pasang kabel → CATAT mapping kabel (Step 3a)
Step 4: Konfigurasi interface (LAN + WAN) di semua router
Step 5: Konfigurasi OSPF di semua router
Step 6: Assign IP ke PC
Step 7: Verifikasi & Testing failover
```

### ⚠️ Step 3a — Catat Mapping Kabel SEBELUM Config IP

Setelah pasang kabel Serial di Packet Tracer, **hover mouse di atas kabel** untuk lihat interface mana yang terhubung. Lalu catat:

```
KABEL 1: R-JKT [Se0/0/0] ←──10.0.0.0/30──→ [Se0/0/0] R-SBY
          IP: 10.0.0.1                       IP: 10.0.0.2

KABEL 2: R-SBY [Se0/0/1] ←──10.0.1.0/30──→ [Se0/0/0] R-MLG
          IP: 10.0.1.1                       IP: 10.0.1.2

KABEL 3: R-JKT [Se0/0/1] ←──10.0.2.0/30──→ [Se0/0/1] R-MLG
          IP: 10.0.2.1                       IP: 10.0.2.2
```

> 💡 **Ini mencegah kesalahan IP ketukar antar interface.** Salah assign IP ke interface yang salah = OSPF neighbor gagal, dan error-nya sulit dideteksi karena kabel kelihatan hijau. Selalu cocokkan: kabel ini nyambung ke mana → berarti subnet-nya apa → berarti IP-nya berapa.

---

### R-JKT

```cisco
enable
configure terminal
hostname R-JKT

! ============================================
! Interface
! ============================================
interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
exit

interface Serial0/0/0
 ip address 10.0.0.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

interface Serial0/0/1
 ip address 10.0.2.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

! ============================================
! OSPF
! ============================================
router ospf 1
 router-id 1.1.1.1
 network 192.168.1.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
exit

end
```

### R-SBY

```cisco
enable
configure terminal
hostname R-SBY

! ============================================
! Interface
! ============================================
interface GigabitEthernet0/0
 ip address 192.168.2.1 255.255.255.0
 no shutdown
exit

interface Serial0/0/0
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit

interface Serial0/0/1
 ip address 10.0.1.1 255.255.255.252
 clock rate 64000
 no shutdown
exit

! ============================================
! OSPF
! ============================================
router ospf 1
 router-id 2.2.2.2
 network 192.168.2.0 0.0.0.255 area 0
 network 10.0.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.3 area 0
exit

end
```

### R-MLG

```cisco
enable
configure terminal
hostname R-MLG

! ============================================
! Interface
! ============================================
interface GigabitEthernet0/0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
exit

interface Serial0/0/0
 ip address 10.0.1.2 255.255.255.252
 no shutdown
exit

interface Serial0/0/1
 ip address 10.0.2.2 255.255.255.252
 no shutdown
exit

! ============================================
! OSPF
! ============================================
router ospf 1
 router-id 3.3.3.3
 network 192.168.3.0 0.0.0.255 area 0
 network 10.0.1.0 0.0.0.3 area 0
 network 10.0.2.0 0.0.0.3 area 0
exit

end
```

### Penjelasan OSPF Config

```cisco
router ospf 1
│           │
│           └── process ID (lokal, gak harus sama antar router)
└── masuk OSPF config mode

 router-id 1.1.1.1
│           │
│           └── identitas unik router (format IP, tapi bukan IP asli)
└── WAJIB unik per router

 network 192.168.1.0 0.0.0.255 area 0
│        │            │          │
│        │            │          └── area 0 = backbone (wajib)
│        │            └── wildcard mask (kebalikan subnet mask)
│        └── network yang mau di-advertise via OSPF
└── "aktifkan OSPF di interface yang match network ini"
```

### Wildcard Mask

Wildcard = kebalikan dari subnet mask:

| Subnet Mask | Wildcard Mask | Artinya |
|-------------|---------------|---------|
| 255.255.255.0 (/24) | 0.0.0.255 | Semua IP di subnet /24 |
| 255.255.255.252 (/30) | 0.0.0.3 | Semua IP di subnet /30 |
| 255.255.255.192 (/26) | 0.0.0.63 | Semua IP di subnet /26 |

Cara hitung cepat: **255 - subnet mask oktet = wildcard oktet**

```
255.255.255.252 → wildcard:
  255-255 = 0
  255-255 = 0
  255-255 = 0
  255-252 = 3
→ 0.0.0.3
```

---

### Switch (Minimal Config)

```cisco
enable
configure terminal
hostname SW-JKT    ! atau SW-SBY / SW-MLG
end
```

### End Devices — IP Assignment

| Device | IP Address | Subnet Mask | Default Gateway |
|--------|------------|-------------|-----------------|
| PC-JKT | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| PC-SBY | 192.168.2.10 | 255.255.255.0 | 192.168.2.1 |
| PC-MLG | 192.168.3.10 | 255.255.255.0 | 192.168.3.1 |

### ⚠️ Verifikasi WAN IP Sebelum Lanjut

Sebelum test OSPF, **pastikan IP WAN cocok per link**. Di setiap router:

```cisco
show ip interface brief
```

Lalu cocokkan:

```
Link JKT ↔ SBY (10.0.0.0/30):  R-JKT Se0/0/0 = 10.0.0.1?  R-SBY Se0/0/0 = 10.0.0.2?
Link SBY ↔ MLG (10.0.1.0/30):  R-SBY Se0/0/1 = 10.0.1.1?  R-MLG Se0/0/0 = 10.0.1.2?
Link JKT ↔ MLG (10.0.2.0/30):  R-JKT Se0/0/1 = 10.0.2.1?  R-MLG Se0/0/1 = 10.0.2.2?
```

**Kalau ada yang gak cocok, fix sekarang** — jangan lanjut ke testing. IP ketukar adalah penyebab #1 OSPF neighbor gagal.

---

## ✅ Verifikasi & Testing

### 1. Cek OSPF Neighbor

**Ini yang paling penting dicek pertama!**

```cisco
show ip ospf neighbor
```

Output di R-JKT:

```
Neighbor ID   Pri  State     Dead Time  Address      Interface
2.2.2.2         0  FULL/  -  00:00:32   10.0.0.2     Se0/0/0
3.3.3.3         0  FULL/  -  00:00:35   10.0.2.2     Se0/0/1
```

**State harus FULL.** Kalau tidak muncul atau stuck di INIT/2WAY, ada masalah config.

### 2. Cek Routing Table

```cisco
show ip route
```

Output di R-JKT:

```
O    192.168.2.0/24 [110/65] via 10.0.0.2, Se0/0/0       ← learned via OSPF
O    192.168.3.0/24 [110/65] via 10.0.2.2, Se0/0/1       ← learned via OSPF
C    192.168.1.0/24 is directly connected, Gi0/0
C    10.0.0.0/30 is directly connected, Se0/0/0
C    10.0.2.0/30 is directly connected, Se0/0/1
```

`O` = OSPF route. `[110/65]` = AD 110, cost 65.

> 💡 **Kamu tidak config static route sama sekali** — tapi routing table sudah terisi otomatis via OSPF!

### 3. Cek OSPF Detail

```cisco
show ip ospf interface brief
```

Menampilkan interface mana yang menjalankan OSPF.

```cisco
show ip protocols
```

Menampilkan OSPF process ID, router ID, dan network yang di-advertise.

### 4. Ping Test

```
# Dari PC-JKT:
ping 192.168.2.10    ← PC Surabaya
ping 192.168.3.10    ← PC Malang

# Dari PC-SBY:
ping 192.168.1.10    ← PC Jakarta
ping 192.168.3.10    ← PC Malang

# Dari PC-MLG:
ping 192.168.1.10    ← PC Jakarta
ping 192.168.2.10    ← PC Surabaya
```

### 5. Traceroute

```
# Dari PC-JKT ke PC-SBY:
tracert 192.168.2.10
```

Output:

```
1   192.168.1.1     ← R-JKT
2   10.0.0.2        ← R-SBY (direct path)
3   192.168.2.10    ← PC-SBY
```

### 6. Test Failover ⚡

#### Step A — Matikan Link JKT ↔ SBY

```cisco
! Di R-JKT:
configure terminal
interface Serial0/0/0
 shutdown
end
```

#### Step B — Tunggu 10-40 Detik

OSPF butuh waktu detect neighbor hilang.

#### Step C — Cek Routing Table

```cisco
show ip route
```

```
O    192.168.2.0/24 [110/129] via 10.0.2.2, Se0/0/1     ← otomatis ganti jalur!
```

Traffic ke Surabaya sekarang **lewat Malang** (JKT → MLG → SBY). Cost naik dari 65 ke 129 karena lewat 2 hop.

#### Step D — Ping & Traceroute

```
# Dari PC-JKT:
tracert 192.168.2.10
```

```
1   192.168.1.1     ← R-JKT
2   10.0.2.2        ← R-MLG (reroute lewat Malang!)
3   10.0.1.1        ← R-SBY
4   192.168.2.10    ← PC-SBY
```

#### Step E — Nyalakan Lagi

```cisco
configure terminal
interface Serial0/0/0
 no shutdown
end
```

Route otomatis kembali ke jalur langsung.

---

## 🔧 Troubleshooting Guide

### Master Flowchart

```
Ping antar kantor gagal?
│
├── Step 1: Ping gateway sendiri (PC → router LAN)
│   ├── ❌ Gagal → masalah LAN (kabel, switch, IP PC, interface router)
│   └── ✅ Reply → lanjut
│
├── Step 2: Ping antar router (WAN link)
│   │
│   │  ⚠️ PENTING: Ping ke IP LAWAN di link yang SAMA
│   │
│   │  Cara cek IP lawan yang benar:
│   │  - Lihat kabel: Se0/0/0 kamu nyambung ke Se berapa di router lawan?
│   │  - Lihat IP addressing table: IP lawan di link tersebut berapa?
│   │  - Jangan asal ping — pastikan subnet-nya cocok!
│   │
│   ├── Dari R-JKT:
│   │   ping 10.0.0.2  ← R-SBY di link JKT↔SBY (10.0.0.0/30)
│   │   ping 10.0.2.2  ← R-MLG di link JKT↔MLG (10.0.2.0/30)
│   │
│   ├── ❌ Gagal → link fisik atau IP config bermasalah (lihat Step 2a)
│   └── ✅ Reply → lanjut ke Step 3
│
├── Step 2a: Ping WAN gagal — cek detail
│   │
│   ├── Cek interface up/up
│   │   └── show ip interface brief
│   │
│   ├── Cek IP kedua sisi di SUBNET YANG SAMA
│   │   └── show running-config | include interface|ip address
│   │       Kedua router di 1 link harus di subnet yang sama!
│   │       R-JKT Se0/0/1 = 10.0.2.1/30 ← subnet 10.0.2.0
│   │       R-MLG Se0/0/1 = 10.0.2.2/30 ← subnet 10.0.2.0 ✅ COCOK
│   │
│   └── ⚠️ SERING TERJADI: IP ketukar antar interface (lihat section di bawah)
│
├── Step 3: Cek OSPF neighbor
│   └── show ip ospf neighbor
│       ├── Semua FULL → lanjut Step 4
│       ├── Sebagian missing → cek link ke neighbor yang hilang
│       └── Semua kosong → OSPF config bermasalah
│
├── Step 4: Cek routing table
│   └── show ip route
│       ├── Ada route O ke semua network → routing OK, cek PC tujuan
│       └── Route O missing → cek OSPF neighbor & network statement
│
└── Step 5: Cek PC tujuan
    ├── IP address benar?
    ├── Subnet mask benar?
    └── Default gateway benar?
```

### ⚠️ IP Ketukar Antar Interface (Kesalahan Paling Umum!)

Ini error yang **sangat sering terjadi** dan sulit dideteksi. Contoh kasus nyata:

#### Masalah

```
Yang di-config (SALAH):
  R-MLG Se0/0/0 = 10.0.2.2  ← harusnya IP untuk link ke SBY
  R-MLG Se0/0/1 = 10.0.1.2  ← harusnya IP untuk link ke JKT

Yang seharusnya (BENAR):
  R-MLG Se0/0/0 = 10.0.1.2  ← link ke SBY (subnet 10.0.1.0/30)
  R-MLG Se0/0/1 = 10.0.2.2  ← link ke JKT (subnet 10.0.2.0/30)
```

#### Gejala

- OSPF neighbor **sebagian** tidak muncul (bukan semua)
- Ping antar router yang directly connected **timeout** (padahal kabel hijau)
- Traceroute bisa ke router tapi timeout ke PC di belakang router lawan
- Satu arah jalan, arah sebaliknya tidak

#### Kenapa Ini Terjadi

```
R-JKT Se0/0/1 (10.0.2.1) ════kabel════ R-MLG Se0/0/1 (10.0.1.2)
      subnet 10.0.2.0/30                 subnet 10.0.1.0/30
                          ↑
              BEDA SUBNET! Gak bisa komunikasi!
```

Kedua ujung kabel **harus di subnet yang sama**. Kalau beda subnet, gak bisa ping walaupun kabel connected.

#### Cara Detect

```cisco
! Di kedua router, cek IP per interface:
show ip interface brief

! Lalu cocokkan dengan tabel:
! - Interface Se0/0/0 nyambung ke router mana?
! - IP-nya sudah di subnet yang sama dengan ujung lawan?
```

**Checklist per link:**

```
Link JKT ↔ SBY (10.0.0.0/30):
  ✅ R-JKT Se0/0/0 = 10.0.0.1 (subnet 10.0.0.0)
  ✅ R-SBY Se0/0/0 = 10.0.0.2 (subnet 10.0.0.0)
  → COCOK

Link SBY ↔ MLG (10.0.1.0/30):
  ✅ R-SBY Se0/0/1 = 10.0.1.1 (subnet 10.0.1.0)
  ✅ R-MLG Se0/0/0 = 10.0.1.2 (subnet 10.0.1.0)
  → COCOK

Link JKT ↔ MLG (10.0.2.0/30):
  ✅ R-JKT Se0/0/1 = 10.0.2.1 (subnet 10.0.2.0)
  ✅ R-MLG Se0/0/1 = 10.0.2.2 (subnet 10.0.2.0)
  → COCOK
```

#### Cara Fix

```cisco
! Di R-MLG, tukar IP-nya:
configure terminal

interface Serial0/0/0
 ip address 10.0.1.2 255.255.255.252

interface Serial0/0/1
 ip address 10.0.2.2 255.255.255.252

end
```

#### Cara Mencegah

**Sebelum config, gambar dulu tabel seperti ini:**

```
KABEL 1: R-JKT [Se0/0/0] ←──10.0.0.0/30──→ [Se0/0/0] R-SBY
          IP: 10.0.0.1                       IP: 10.0.0.2

KABEL 2: R-SBY [Se0/0/1] ←──10.0.1.0/30──→ [Se0/0/0] R-MLG
          IP: 10.0.1.1                       IP: 10.0.1.2

KABEL 3: R-JKT [Se0/0/1] ←──10.0.2.0/30──→ [Se0/0/1] R-MLG
          IP: 10.0.2.1                       IP: 10.0.2.2
```

> 💡 **Pro tip:** Di Packet Tracer, hover mouse di atas kabel — akan terlihat interface mana yang terhubung di kedua ujung. Cocokkan dengan tabel sebelum config IP.

---

### OSPF Neighbor Tidak Muncul

```
show ip ospf neighbor kosong atau sebagian?
│
├── Cek interface up/up
│   └── show ip interface brief
│
├── Cek IP kedua sisi di subnet yang sama (lihat section di atas!)
│   └── show ip interface brief di KEDUA router
│       └── IP ketukar? → fix IP di interface yang benar
│
├── Cek OSPF running di interface tersebut
│   └── show ip ospf interface brief
│       └── Interface gak muncul → network statement salah
│
├── Cek network statement include interface WAN
│   └── show running-config | section ospf
│       └── Harus ada: network 10.0.x.0 0.0.0.3 area 0
│
├── Cek area sama di kedua sisi
│   └── R-JKT area 0, R-SBY area 1 → ❌ gak bisa neighbor
│
├── Cek wildcard mask benar
│   └── network 10.0.0.0 0.0.0.3 (bukan 255.255.255.252)
│
└── Cek Hello/Dead timer match
    └── show ip ospf interface Se0/0/0
        └── Hello 10, Dead 40 (default, harus sama di kedua sisi)
```

### Bisa Ping Router Tapi Timeout ke PC di Belakang Router

```
Dari R-MLG: ping 10.0.1.1 (R-SBY interface) → ✅ Reply
Dari R-MLG: ping 192.168.2.10 (PC-SBY)      → ❌ Timeout

Kenapa?
│
├── Paket dari R-MLG → R-SBY → PC-SBY        ✅ sampai
│
└── Reply dari PC-SBY → R-SBY → R-MLG?
    └── R-SBY punya route ke 192.168.3.0 (MLG LAN)?
        ├── ❌ Tidak → reply gak bisa balik!
        │   └── Fix: pastikan OSPF neighbor MLG↔SBY established
        │       agar R-SBY tahu route ke 192.168.3.0
        └── ✅ Ya → cek masalah lain (PC gateway, dll)
```

> 💡 **Ingat: ping butuh 2 arah.** Paket pergi harus sampai, DAN reply harus bisa balik. Kalau route balik tidak ada, tetap timeout.

### OSPF Route Tidak Muncul di Routing Table

| Cek | Command | Solusi |
|-----|---------|--------|
| Neighbor sudah FULL? | `show ip ospf neighbor` | Kalau belum, fix neighbor dulu |
| Network di-advertise? | `show ip protocols` | Tambah network statement yang kurang |
| Interface masuk OSPF? | `show ip ospf interface brief` | Cek wildcard mask |

### Kesalahan Umum

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| **IP ketukar antar interface** | **Config IP di interface yang salah** | **Cek kabel → cocokkan IP dengan subnet** |
| Neighbor sebagian hilang | IP beda subnet di 1 link | Cocokkan IP kedua ujung per link |
| Ping router OK, ping PC timeout | Route balik tidak ada | Fix OSPF agar semua neighbor FULL |
| Neighbor gak muncul | Area ID beda | Samakan area di kedua sisi |
| Neighbor gak muncul | Wildcard mask salah | 255 - subnet = wildcard |
| Neighbor gak muncul | Interface down | `no shutdown` |
| Route gak muncul | Network belum di-advertise | Tambah `network` statement |
| Router ID conflict | 2 router pakai ID sama | Set `router-id` unik per router |
| OSPF gak jalan | Lupa `router ospf 1` | Masuk OSPF config dulu |
| Cost salah | Bandwidth interface gak sesuai | `bandwidth` command di interface |

### Command Cheat Sheet

| Command | Fungsi |
|---------|--------|
| `show ip ospf neighbor` | Lihat neighbor dan state (harus FULL) |
| `show ip route ospf` | Lihat hanya OSPF route |
| `show ip route` | Lihat semua route |
| `show ip interface brief` | Cek IP semua interface (cocokkan per link!) |
| `show ip ospf interface brief` | Interface mana yang jalankan OSPF |
| `show ip ospf interface Se0/0/0` | Detail OSPF di interface tertentu |
| `show ip protocols` | OSPF process, router ID, networks |
| `show ip ospf database` | OSPF link-state database |
| `show running-config \| section ospf` | Lihat semua config OSPF |
| `show running-config \| include ip address` | Cek semua IP yang di-config |
| `show controllers Serial0/0/0` | Cek DCE/DTE (untuk clock rate) |
| `clear ip ospf process` | Reset OSPF (ketik `yes` saat diminta) |
| `debug ip ospf events` | Live debug OSPF (hati-hati di production!) |

---

## 🏭 Production Best Practices

1. **Selalu set router-id manual** — kalau tidak, OSPF ambil dari IP tertinggi yang aktif. Ini bisa berubah kalau interface down, menyebabkan OSPF flap.

2. **Semua internal network masuk area 0** — di single area OSPF, semua wajib area 0. Multi-area dipelajari nanti.

3. **Passive interface di LAN** — LAN interface gak perlu kirim Hello packet (gak ada router lain di LAN):

```cisco
router ospf 1
 passive-interface GigabitEthernet0/0
```

Ini prevent OSPF Hello packet di-broadcast ke PC — hemat bandwidth dan lebih secure.

4. **Jangan debug di production** — `debug ip ospf events` bisa flood console dan crash router di network besar. Gunakan `show` commands aja.

5. **Monitor neighbor** — di production, alert kalau OSPF neighbor hilang. Neighbor down = ada link yang mati.

6. **Authentication** — di production, OSPF neighbor harus pakai authentication agar router asing gak bisa join:

```cisco
interface Serial0/0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 SecretKey123
```

7. **Summarize route di ABR** — untuk multi-area nanti, summarize route di border router untuk kurangi routing table size.

---

## 📝 Catatan untuk README di GitHub

Setelah selesai, tambahkan:
- Screenshot topologi segitiga di Packet Tracer
  <img width="1109" height="1101" alt="image" src="https://github.com/user-attachments/assets/18990866-ff73-4b57-bfe4-55c4c3774af5" />

- Screenshot `show ip ospf neighbor` (semua FULL)
  <img width="857" height="133" alt="image" src="https://github.com/user-attachments/assets/ee1a0422-1744-46fb-8e72-bf06e4ff0a0b" />

- Screenshot `show ip route` (ada route O)
  <img width="718" height="172" alt="image" src="https://github.com/user-attachments/assets/d67d78a2-6212-426d-961c-29807f14b430" />

- Screenshot traceroute normal path
  <img width="705" height="218" alt="image" src="https://github.com/user-attachments/assets/50b8d1d9-0124-4988-bb27-fa041ec91b31" />

- Screenshot traceroute setelah failover (beda jalur)
  <img width="1174" height="1021" alt="image" src="https://github.com/user-attachments/assets/72424747-0d9b-4b3c-8b8e-672517bf0044" />
  <img width="644" height="245" alt="image" src="https://github.com/user-attachments/assets/327f8806-b8ea-4c8a-b2de-3cd0e5b77e71" />


- File `.pkt`

---

## 🔄 Perbandingan Lab 007 vs Lab 008

| | Lab 007 (Static Route) | Lab 008 (OSPF) |
|---|---|---|
| Route config | Manual per tujuan | Otomatis via OSPF |
| Failover | Floating static (manual AD) | Auto reroute |
| Tambah router baru | Config di semua router | Config di router baru saja |
| Routing table | `S` (Static) | `O` (OSPF) |
| Scalability | Kecil (2-3 router) | Besar (100+ router) |

---

## ⏭️ Lab Selanjutnya

→ **009_ACL_Standard_Extended** — Kontrol traffic: siapa boleh akses apa. Sekarang network sudah jalan, saatnya amankan.
