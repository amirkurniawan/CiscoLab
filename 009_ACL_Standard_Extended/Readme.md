# 009 — ACL Standard & Extended

## 🎯 Tujuan Lab

Mengkonfigurasi Access Control List (ACL) untuk **mengontrol traffic** — menentukan siapa boleh akses apa. Network sudah jalan (routing OK), sekarang saatnya **amankan**. Tanpa ACL, semua orang bisa akses semua — di production itu berbahaya.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **ACL** | Access Control List — aturan filter traffic di router |
| **Standard ACL** | Filter berdasarkan **source IP saja** (sederhana) |
| **Extended ACL** | Filter berdasarkan **source IP, destination IP, protocol, port** (detail) |
| **Inbound / Outbound** | Arah ACL diterapkan di interface |
| **Implicit Deny** | Aturan tersembunyi di akhir setiap ACL: deny semua yang tidak match |
| **Wildcard Mask** | Mask untuk menentukan range IP yang di-match ACL |

### Analogi ACL

```
ACL = Satpam di pintu gedung

Tanpa ACL:
  Semua orang bisa masuk ke semua ruangan
  → Engineering bisa akses server Finance
  → Tamu bisa akses data center
  → Bahaya!

Dengan ACL:
  Satpam cek: "Kamu siapa? Mau ke mana? Boleh gak?"
  → Engineering boleh akses server Engineering
  → Finance TIDAK boleh akses server Engineering
  → Tamu cuma boleh akses internet
```

### Standard vs Extended ACL

| | Standard ACL | Extended ACL |
|---|---|---|
| **Filter** | Source IP saja | Source, destination, protocol, port |
| **Nomor** | 1-99 | 100-199 |
| **Contoh** | "Block semua dari 192.168.1.0" | "Block 192.168.1.0 akses port 80 ke server" |
| **Presisi** | Rendah (block semua traffic dari source) | Tinggi (block traffic spesifik) |
| **Ditaruh** | Dekat **destination** | Dekat **source** |

> 💡 **Kenapa Standard dekat destination, Extended dekat source?**
> Standard cuma lihat source IP — kalau ditaruh dekat source, dia block **semua** traffic dari IP tersebut ke mana-mana. Ditaruh dekat destination biar cuma block traffic ke tujuan tertentu.
> Extended bisa lihat source DAN destination — jadi aman ditaruh dekat source, traffic lain tidak terganggu.

### Cara Router Proses ACL

```
Paket masuk interface
│
├── Ada ACL di interface?
│   ├── Tidak → forward seperti biasa
│   └── Ya ↓
│
├── Cek rule 1: match?
│   ├── Ya → permit / deny (stop, gak cek rule selanjutnya)
│   └── Tidak → lanjut rule 2
│
├── Cek rule 2: match?
│   ├── Ya → permit / deny (stop)
│   └── Tidak → lanjut rule 3
│
├── ... (cek semua rule)
│
└── Tidak ada yang match
    └── IMPLICIT DENY → paket di-drop ❌
```

> ⚠️ **Urutan rule SANGAT PENTING.** ACL diproses **top-down**, begitu match langsung eksekusi. Rule yang lebih spesifik harus di atas.

### Inbound vs Outbound

```
          [Router]
             │
   ──IN──→ [Interface] ──OUT──→
             │
  Paket     ACL cek          Paket
  masuk     di sini           keluar
```

| Direction | Kapan Dicek | Contoh |
|-----------|-------------|--------|
| **Inbound (in)** | Saat paket **masuk** ke interface | Filter traffic sebelum router proses |
| **Outbound (out)** | Saat paket **keluar** dari interface | Filter traffic setelah router proses routing |

---

## 🏢 Skenario

Kembali ke topologi **Lab 004-006** (PT. Nusantara Digital). Network sudah jalan dengan VLAN, DHCP, dan inter-VLAN routing. Sekarang IT Manager minta:

### Security Requirements

| No | Requirement | Tipe ACL |
|----|------------|----------|
| 1 | **Finance TIDAK boleh diakses oleh Engineering** — data keuangan sensitif | Extended |
| 2 | **Server Farm hanya boleh diakses oleh Management dan Finance** | Extended |
| 3 | **Management bisa akses semua** — IT admin butuh full access | (Tidak di-block) |
| 4 | **Semua divisi bisa ping gateway masing-masing** — basic connectivity tetap jalan | Extended (permit ICMP) |

---

## 📋 IP Reference (dari Lab 004)

| VLAN | Divisi | Subnet | Gateway |
|------|--------|--------|---------|
| 10 | Engineering | 192.168.10.0/26 | 192.168.10.1 |
| 20 | Marketing | 192.168.10.64/27 | 192.168.10.65 |
| 30 | Finance | 192.168.10.96/28 | 192.168.10.97 |
| 40 | Management | 192.168.10.128/29 | 192.168.10.129 |
| 50 | Server Farm | 192.168.10.112/28 | 192.168.10.113 |

---

## 🖥️ Topologi

Sama dengan Lab 004-006. Tidak ada perubahan fisik — hanya tambah ACL di R-CORE.

```
                        [R-CORE]
                   ACL diterapkan di sini
                  Gi0/0  Gi0/1   Gi0/2
                    │      │       │
                trunk  trunk   trunk
                    │      │       │
               [SW-ENG] [SW-OFFICE] [SW-SERVER]
                  │       │  │  │       │
               PC-Eng  PC-Mkt │ PC-Mgmt SRV-01
                       PC-Fin
```

---

## ⚙️ Konfigurasi

### Urutan Konfigurasi

```
Step 1: Pastikan semua bisa ping semua (sebelum ACL)
Step 2: Buat ACL rules
Step 3: Apply ACL ke interface
Step 4: Test — yang di-block gak bisa, yang di-permit bisa
```

> ⚠️ **Selalu test connectivity SEBELUM apply ACL.** Kalau sebelum ACL aja gak bisa ping, masalahnya bukan ACL — jangan bikin troubleshoot makin rumit.

---

### ACL Rules Design

Sebelum config, **design dulu di kertas:**

#### ACL 110 — Protect Finance (Applied Inbound di Gi0/0.10)

Block Engineering akses ke Finance:

```
10  permit icmp 192.168.10.0 0.0.0.63 192.168.10.1 0.0.0.0          → ping gateway sendiri OK
20  deny   ip   192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15        → block Eng → Finance
30  permit ip   192.168.10.0 0.0.0.63 any                            → Eng boleh akses lainnya
```

#### ACL 120 — Protect Server Farm (Applied Inbound di Gi0/2.50 outbound)

Hanya Management dan Finance boleh akses Server:

```
10  permit ip  192.168.10.128 0.0.0.7  192.168.10.112 0.0.0.15      → Management → Server OK
20  permit ip  192.168.10.96 0.0.0.15  192.168.10.112 0.0.0.15      → Finance → Server OK
30  deny   ip  any                     192.168.10.112 0.0.0.15      → Block semua lain → Server
40  permit ip  any any                                               → Traffic lain tetap jalan
```

> 💡 **Rule 40 (permit any any) penting!** Tanpa ini, implicit deny akan block SEMUA traffic yang lewat interface tersebut — termasuk traffic yang gak ada hubungannya dengan Server.

---

### R-CORE — ACL Configuration

```cisco
enable
configure terminal

! ============================================
! ACL 110 — Block Engineering → Finance
! ============================================
! Rule: Engineering gak boleh akses Finance
! Tapi Engineering boleh akses semua lainnya

access-list 110 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1
access-list 110 deny   ip   192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
access-list 110 permit ip   192.168.10.0 0.0.0.63 any

! ============================================
! ACL 120 — Protect Server Farm
! ============================================
! Rule: Hanya Management dan Finance boleh akses Server
! Traffic lain yang gak ke Server tetap boleh lewat

access-list 120 permit ip 192.168.10.128 0.0.0.7 192.168.10.112 0.0.0.15
access-list 120 permit ip 192.168.10.96 0.0.0.15 192.168.10.112 0.0.0.15
access-list 120 deny   ip any 192.168.10.112 0.0.0.15
access-list 120 permit ip any any

! ============================================
! Apply ACL ke Interface
! ============================================

! ACL 110 → inbound di sub-interface Engineering
interface GigabitEthernet0/0.10
 ip access-group 110 in
exit

! ACL 120 → inbound di sub-interface ke trunk SW-OFFICE dan Eng
! Apply di outbound server agar cek traffic SEBELUM masuk server
interface GigabitEthernet0/2.50
 ip access-group 120 out
exit

end
```

### Penjelasan Syntax

```cisco
access-list 110 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1
│           │   │      │    │              │       │    │
│           │   │      │    │              │       │    └── destination: gateway IP
│           │   │      │    │              │       └── "host" = wildcard 0.0.0.0 (exact match)
│           │   │      │    │              └── wildcard mask (/26 → 0.0.0.63)
│           │   │      │    └── source network (Engineering)
│           │   │      └── protocol (icmp/ip/tcp/udp)
│           │   └── permit atau deny
│           └── ACL number (100-199 = extended)
└── command
```

### Wildcard Mask Reference

| Subnet | Prefix | Wildcard | Artinya |
|--------|--------|----------|---------|
| 255.255.255.192 | /26 | 0.0.0.63 | 64 IP (Engineering) |
| 255.255.255.224 | /27 | 0.0.0.31 | 32 IP (Marketing) |
| 255.255.255.240 | /28 | 0.0.0.15 | 16 IP (Finance, Server) |
| 255.255.255.248 | /29 | 0.0.0.7 | 8 IP (Management) |
| 255.255.255.255 | /32 | 0.0.0.0 | 1 IP (host) |

Shortcut: **`host 192.168.10.1`** = **`192.168.10.1 0.0.0.0`** (sama aja, lebih gampang dibaca).

### Keyword Shortcut

| Keyword | Sama Dengan | Artinya |
|---------|-------------|---------|
| `host 192.168.10.1` | `192.168.10.1 0.0.0.0` | Exactly 1 IP |
| `any` | `0.0.0.0 255.255.255.255` | Semua IP |

---

## ✅ Verifikasi & Testing

### 1. Cek ACL Sudah Dibuat

```cisco
show access-lists
```

Output:

```
Extended IP access list 110
    10 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1
    20 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
    30 permit ip 192.168.10.0 0.0.0.63 any
Extended IP access list 120
    10 permit ip 192.168.10.128 0.0.0.7 192.168.10.112 0.0.0.15
    20 permit ip 192.168.10.96 0.0.0.15 192.168.10.112 0.0.0.15
    30 deny ip any 192.168.10.112 0.0.0.15
    40 permit ip any any
```

### 2. Cek ACL Applied di Interface

```cisco
show ip interface GigabitEthernet0/0.10
```

Cari baris:

```
Inbound access list is 110
Outbound access list is not set
```

```cisco
show ip interface GigabitEthernet0/2.50
```

```
Inbound access list is not set
Outbound access list is 120
```

### 3. Test Matrix

Ini yang harus kamu test satu-satu:

| Dari | Ke | Seharusnya | Kenapa |
|------|----|------------|--------|
| PC-Eng | Gateway .1 | ✅ Permit | ACL 110 rule 10: permit icmp to gateway |
| PC-Eng | PC-Fin (.98) | ❌ Deny | ACL 110 rule 20: deny Eng → Finance |
| PC-Eng | PC-Mkt (.66) | ✅ Permit | ACL 110 rule 30: permit Eng → any |
| PC-Eng | SRV-01 (.114) | ❌ Deny | ACL 120 rule 30: deny any → Server |
| PC-Fin | SRV-01 (.114) | ✅ Permit | ACL 120 rule 20: permit Finance → Server |
| PC-Mgmt | SRV-01 (.114) | ✅ Permit | ACL 120 rule 10: permit Mgmt → Server |
| PC-Mkt | SRV-01 (.114) | ❌ Deny | ACL 120 rule 30: deny any → Server |
| PC-Mkt | PC-Fin (.98) | ✅ Permit | Tidak ada ACL yang block |
| PC-Mgmt | PC-Eng (.2) | ✅ Permit | Management akses semua |
| PC-Fin | PC-Eng (.2) | ✅ Permit | Tidak ada ACL yang block arah ini |

> ⚠️ **Test SEMUA kombinasi di atas.** ACL yang salah bisa block traffic yang seharusnya diperbolehkan (false positive) atau permit traffic yang seharusnya diblock (false negative). Kedua-duanya bahaya.

### 4. Cek ACL Hit Count

```cisco
show access-lists
```

```
Extended IP access list 110
    10 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1 (4 matches)
    20 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15 (2 matches)
    30 permit ip 192.168.10.0 0.0.0.63 any (8 matches)
```

`(X matches)` menunjukkan berapa paket yang match rule tersebut. Berguna untuk verifikasi ACL bekerja sesuai harapan.

### 5. Reset Hit Counter

```cisco
clear access-list counters
```

Berguna saat mau test ulang dari awal.

---

## 🔧 Troubleshooting Guide

### Master Flowchart

```
Setelah apply ACL, ada yang gak bisa diakses?
│
├── Apakah SEBELUM ACL bisa ping?
│   ├── ❌ Tidak → masalah bukan ACL (cek VLAN, trunk, routing)
│   └── ✅ Bisa → lanjut
│
├── Cek ACL sudah applied di interface yang benar
│   └── show ip interface [interface]
│       ├── "access list is not set" → ACL belum di-apply
│       └── ACL terpasang → lanjut
│
├── Cek arah ACL (in/out) benar
│   └── Salah arah = ACL gak nge-filter traffic yang diinginkan
│
├── Cek urutan rule
│   └── show access-lists
│       └── Rule lebih general di atas rule spesifik?
│           → Traffic match rule general duluan, rule spesifik gak pernah kena
│
├── Cek implicit deny
│   └── Ada "permit ip any any" di akhir?
│       ├── Tidak → SEMUA traffic yang gak match di-deny!
│       └── Ya → cek rule deny yang terlalu broad
│
└── Cek wildcard mask
    └── Wildcard salah = range IP yang di-match salah
```

### Traffic yang Seharusnya Permit Tapi Ke-Block

| Cek | Command | Kemungkinan Masalah |
|-----|---------|-------------------|
| Implicit deny? | `show access-lists` | Lupa `permit ip any any` di akhir ACL |
| Rule urutan salah? | `show access-lists` | Deny terlalu broad di atas permit spesifik |
| ACL di interface salah? | `show ip interface [intf]` | ACL di-apply di interface yang salah |
| Arah salah? | `show ip interface [intf]` | Harusnya `in` tapi di-apply `out` atau sebaliknya |
| Wildcard mask salah? | `show access-lists` | Range IP terlalu luas, kena block |

### Traffic yang Seharusnya Deny Tapi Bisa Lewat

| Cek | Command | Kemungkinan Masalah |
|-----|---------|-------------------|
| ACL sudah di-apply? | `show ip interface [intf]` | ACL dibuat tapi lupa di-apply ke interface |
| Hit count naik? | `show access-lists` | Kalau deny rule 0 matches = traffic gak kena rule |
| Rule urutan salah? | `show access-lists` | Permit terlalu broad di atas deny spesifik |
| Interface salah? | `show ip interface [intf]` | ACL di interface lain, traffic lewat interface ini |

### Kesalahan Paling Umum

#### 1. Lupa Implicit Deny

```cisco
! SALAH — semua traffic selain Mgmt→Server di-block
access-list 120 permit ip 192.168.10.128 0.0.0.7 192.168.10.112 0.0.0.15
! ← implicit deny all di sini! Semua traffic lain DROP

! BENAR — tambah permit any any di akhir
access-list 120 permit ip 192.168.10.128 0.0.0.7 192.168.10.112 0.0.0.15
access-list 120 deny   ip any 192.168.10.112 0.0.0.15
access-list 120 permit ip any any
! ← traffic yang gak ke Server tetap bisa lewat
```

> ⚠️ **Ini error #1 di ACL.** Setiap ACL punya invisible "deny all" di akhir. Kalau kamu cuma tulis permit rules tanpa "permit any any" di akhir, SEMUA traffic lain di-drop.

#### 2. Urutan Rule Salah

```cisco
! SALAH — permit any any di atas, deny gak pernah kena
access-list 120 permit ip any any                                    ← match semua, stop
access-list 120 deny   ip any 192.168.10.112 0.0.0.15              ← gak pernah dicek!

! BENAR — deny dulu, baru permit
access-list 120 deny   ip any 192.168.10.112 0.0.0.15
access-list 120 permit ip any any
```

#### 3. ACL Dibuat Tapi Lupa Apply

```cisco
! ACL dibuat...
access-list 110 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
access-list 110 permit ip any any

! ...tapi lupa apply ke interface!
! Harus tambah:
interface GigabitEthernet0/0.10
 ip access-group 110 in        ← INI YANG SERING LUPA
```

#### 4. Salah Interface / Arah

```cisco
! SALAH — ACL block Eng→Finance di-apply di interface Finance (outbound)
! Traffic dari Marketing ke Finance juga kena block!
interface GigabitEthernet0/1.30
 ip access-group 110 out       ← wrong interface & direction

! BENAR — apply di interface Engineering (inbound)
! Hanya filter traffic yang keluar dari Engineering
interface GigabitEthernet0/0.10
 ip access-group 110 in        ← correct
```

#### 5. Wildcard Mask Salah

```cisco
! SALAH — wildcard 0.0.0.255 = /24, terlalu luas
access-list 110 deny ip 192.168.10.0 0.0.0.255 192.168.10.96 0.0.0.15
! Ini block 192.168.10.0 - 192.168.10.255 → SEMUA VLAN kena!

! BENAR — wildcard 0.0.0.63 = /26, hanya Engineering
access-list 110 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
! Ini block 192.168.10.0 - 192.168.10.63 → hanya Engineering
```

### Cara Hapus dan Buat Ulang ACL

```cisco
configure terminal

! Hapus seluruh ACL 110
no access-list 110

! Hapus ACL dari interface dulu kalau mau ganti
interface GigabitEthernet0/0.10
 no ip access-group 110 in
exit

! Buat ulang
access-list 110 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1
access-list 110 deny   ip   192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
access-list 110 permit ip   192.168.10.0 0.0.0.63 any

! Apply lagi
interface GigabitEthernet0/0.10
 ip access-group 110 in
exit

end
```

> ⚠️ **`no access-list 110` hapus SEMUA rule di ACL 110.** Di Cisco IOS, kamu gak bisa hapus 1 rule tertentu dari numbered ACL — harus hapus semua lalu buat ulang. Ini salah satu kelemahan numbered ACL. Di production, pakai **Named ACL** yang bisa edit per rule.

### Named ACL (Production Alternative)

```cisco
! Named ACL — bisa hapus/tambah rule individual
ip access-list extended BLOCK-ENG-TO-FIN
 10 permit icmp 192.168.10.0 0.0.0.63 host 192.168.10.1
 20 deny   ip   192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
 30 permit ip   192.168.10.0 0.0.0.63 any

! Hapus 1 rule saja:
ip access-list extended BLOCK-ENG-TO-FIN
 no 20     ← hapus rule nomor 20 saja, sisanya tetap

! Sisipkan rule baru:
ip access-list extended BLOCK-ENG-TO-FIN
 15 deny tcp 192.168.10.0 0.0.0.63 host 192.168.10.114 eq 22  ← sisip antara 10 dan 20

! Apply sama seperti numbered:
interface GigabitEthernet0/0.10
 ip access-group BLOCK-ENG-TO-FIN in
```

### Command Cheat Sheet

| Command | Fungsi |
|---------|--------|
| `show access-lists` | Lihat semua ACL dan hit count |
| `show access-lists 110` | Lihat ACL 110 saja |
| `show ip interface [intf]` | Cek ACL apa yang di-apply dan arahnya |
| `show running-config \| section access-list` | Lihat semua ACL di running config |
| `clear access-list counters` | Reset hit count semua ACL |
| `clear access-list counters 110` | Reset hit count ACL 110 saja |
| `no access-list 110` | Hapus seluruh ACL 110 |
| `no ip access-group 110 in` | Lepas ACL dari interface |

---

## 🏭 Production Best Practices

1. **Design ACL di kertas/spreadsheet dulu** — jangan langsung config. Satu rule salah bisa lock out semua user, termasuk kamu sendiri.

2. **Test SEBELUM apply** — pastikan connectivity jalan tanpa ACL. Kalau sudah rusak sebelum ACL, ACL cuma bikin makin susah di-debug.

3. **Selalu ingat implicit deny** — setiap ACL diakhiri oleh invisible `deny any`. Kalau kamu cuma tulis deny rules tanpa permit, SEMUA traffic di-drop.

4. **Pakai Named ACL di production** — numbered ACL harus hapus semua kalau mau edit. Named ACL bisa edit per rule — jauh lebih aman di production.

5. **Log ACL match** — di production, tambah `log` di akhir rule penting:

```cisco
access-list 110 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15 log
```

Ini log ke console/syslog setiap ada paket yang match — berguna untuk monitoring dan audit.

6. **Document setiap rule** — gunakan `remark`:

```cisco
access-list 110 remark === Block Engineering to Finance ===
access-list 110 deny ip 192.168.10.0 0.0.0.63 192.168.10.96 0.0.0.15
```

7. **Principle of Least Privilege** — hanya permit yang benar-benar dibutuhkan, deny sisanya. Lebih aman daripada permit semua lalu block satu-satu.

8. **Test setiap rule setelah apply** — jangan apply semua lalu baru test. Apply satu ACL, test, baru lanjut ACL berikutnya.

9. **Backup config sebelum apply ACL** — kalau salah, bisa rollback:

```cisco
copy running-config startup-config
```

10. **ACL bukan pengganti firewall** — ACL itu stateless (gak track connection state). Di production, ACL di-combine dengan firewall (ASA, Palo Alto, Fortinet) untuk security yang proper.

---

## 📝 Catatan untuk README di GitHub

Setelah selesai, tambahkan:
- Screenshot `show access-lists` (dengan hit count)
- Screenshot ping test matrix (yang permit dan yang deny)
- Screenshot `show ip interface` yang menunjukkan ACL applied
- File `.pkt`

---

## 🔄 Perbandingan: Network Tanpa vs Dengan ACL

| | Tanpa ACL | Dengan ACL |
|---|---|---|
| Engineering → Finance | ✅ Bisa | ❌ Blocked |
| Engineering → Marketing | ✅ Bisa | ✅ Bisa |
| Marketing → Server | ✅ Bisa | ❌ Blocked |
| Management → Server | ✅ Bisa | ✅ Bisa |
| Finance → Server | ✅ Bisa | ✅ Bisa |
| Security | ❌ Zero | ✅ Controlled |

---

## ⏭️ Lab Selanjutnya

→ **010_NAT_PAT** — Network Address Translation: semua device internal bisa akses internet pakai 1 public IP.
