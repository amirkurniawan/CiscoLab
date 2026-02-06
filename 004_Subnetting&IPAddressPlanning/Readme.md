# 004 — Subnetting & IP Address Planning

## 🎯 Tujuan Lab

Merancang IP addressing plan untuk sebuah kantor dengan beberapa divisi, menggunakan teknik subnetting VLSM (Variable Length Subnet Mask). Ini adalah skill **paling fundamental** di production network — salah design subnet di awal = reprovision seluruh infrastructure.

---

## 📖 Konsep yang Dipelajari

| Konsep | Deskripsi |
|--------|-----------|
| **Subnetting** | Membagi network besar menjadi subnet-subnet kecil sesuai kebutuhan |
| **VLSM** | Variable Length Subnet Mask — tiap subnet bisa punya ukuran berbeda |
| **IP Planning** | Menentukan alokasi IP untuk setiap segmen network secara terstruktur |
| **Reserved IP** | Best practice: reserved IP untuk gateway, server, dan management |
| **Dokumentasi** | Membuat IP addressing table — wajib ada di production |

---

## 🏢 Skenario

Kamu adalah network engineer di **PT. Nusantara Digital**. Perusahaan memiliki 1 gedung kantor dengan 4 divisi:

| Divisi | Jumlah Host |
|--------|-------------|
| Engineering | 50 host |
| Marketing | 25 host |
| Finance | 12 host |
| Management | 5 host |
| Server Farm | 6 host |
| Point-to-Point Link (antar router) | 2 host |

**Network yang diberikan oleh ISP:** `192.168.10.0/24`

Tugasmu: bagi network ini menjadi subnet-subnet menggunakan VLSM agar efisien.

---

## 📋 Task List

### Task 1 — Hitung VLSM (di kertas/notes dulu)

Urutkan divisi dari yang **paling besar** ke paling kecil, lalu alokasikan subnet:

| No | Divisi | Jumlah Host | Subnet yang Dibutuhkan | Network Address | Subnet Mask | Range IP | Broadcast | Gateway (.1) |
|----|--------|-------------|----------------------|-----------------|-------------|----------|-----------|---------------|
| 1 | Engineering | 50 | /26 (62 usable) | ? | ? | ? | ? | ? |
| 2 | Marketing | 25 | /27 (30 usable) | ? | ? | ? | ? | ? |
| 3 | Finance | 12 | /28 (14 usable) | ? | ? | ? | ? | ? |
| 4 | Server Farm | 6 | /28 (14 usable) | ? | ? | ? | ? | ? |
| 5 | Management | 5 | /29 (6 usable) | ? | ? | ? | ? | ? |
| 6 | P2P Link | 2 | /30 (2 usable) | ? | ? | ? | ? | ? |

> ⚠️ **Isi tabel ini sendiri sebelum lanjut ke Packet Tracer!**
> Ini adalah latihan paling penting di lab ini.

### Task 2 — Bangun Topologi di Packet Tracer

```
                        [Router-CORE]
                       /      |       \
                      /       |        \
               [SW-Eng]  [SW-Office]  [SW-Server]
                  |        /     \         |
               PC-Eng   PC-Mkt  PC-Fin   Server
                         PC-Mgmt
```

**Perangkat yang dibutuhkan:**

| Device | Model | Hostname |
|--------|-------|----------|
| Router | 2911 | R-CORE |
| Switch 1 | 2960 | SW-ENG |
| Switch 2 | 2960 | SW-OFFICE |
| Switch 3 | 2960 | SW-SERVER |
| PC | - | Minimal 1 PC per divisi |
| Server | - | SRV-01 |

### Task 3 — Konfigurasi Router (Sub-interface)

Karena semua subnet melewati 1 router, gunakan **Router-on-a-Stick** (sub-interface):

```
R-CORE:
  GigabitEthernet0/0.10  → Engineering subnet   → gateway .1
  GigabitEthernet0/0.20  → Marketing subnet     → gateway .1
  GigabitEthernet0/0.30  → Finance subnet       → gateway .1
  GigabitEthernet0/0.40  → Management subnet    → gateway .1
  GigabitEthernet0/0.50  → Server Farm subnet   → gateway .1
```

> 💡 **Production best practice:** Gateway selalu di `.1` agar konsisten dan mudah di-manage.

### Task 4 — Konfigurasi Switch (VLAN Mapping)

| VLAN ID | Name | Subnet |
|---------|------|--------|
| 10 | ENGINEERING | Engineering subnet |
| 20 | MARKETING | Marketing subnet |
| 30 | FINANCE | Finance subnet |
| 40 | MANAGEMENT | Management subnet |
| 50 | SERVER | Server Farm subnet |

- Trunk port ke router
- Access port ke PC/Server sesuai VLAN

### Task 5 — Assign IP ke End Devices

Setiap PC dan Server dikonfigurasi dengan:
- IP Address (dari range subnet masing-masing)
- Subnet Mask
- Default Gateway (`.1` dari subnet masing-masing)

### Task 6 — Verifikasi & Testing

```
# Dari PC Engineering, ping ke:
ping [PC Marketing IP]        ← harus reply (inter-VLAN routing)
ping [PC Finance IP]          ← harus reply
ping [Server IP]              ← harus reply
ping [Gateway Engineering]    ← harus reply

# Cek routing table di router
show ip route

# Cek interface status
show ip interface brief

# Cek VLAN di switch
show vlan brief
```

---

## ✅ Kriteria Sukses

- [ ] Tabel VLSM terisi lengkap dan benar (tidak ada overlap subnet)
- [ ] Semua PC bisa ping ke gateway masing-masing
- [ ] Semua PC bisa ping ke PC di subnet/VLAN lain (inter-VLAN works)
- [ ] Semua PC bisa ping ke Server
- [ ] `show ip route` di router menunjukkan semua connected subnet
- [ ] `show vlan brief` di switch menunjukkan VLAN assignment yang benar
- [ ] IP addressing terdokumentasi rapi

---

## 🏭 Production Best Practices

1. **Selalu dokumentasikan IP plan** — di production, ini jadi sumber kebenaran (source of truth). Tanpa dokumentasi, troubleshoot jadi nightmare.

2. **Gateway selalu di `.1`** — konsistensi = less human error.

3. **Sisakan ruang untuk growth** — jangan pakai `/26` kalau divisi Engineering mungkin berkembang ke 80 orang. Di production, pertimbangkan 2-3 tahun ke depan.

4. **Reserved IP ranges:**
   - `.1` → Gateway
   - `.2 - .10` → Network devices (AP, switch management, dll)
   - `.11 - .254` → End devices / DHCP pool

5. **Jangan pakai `/24` untuk semuanya** — di production, flat network = broadcast storm, security risk, dan susah di-manage.

6. **Point-to-Point link pakai `/30` atau `/31`** — hemat IP, ini standar di production untuk link antar router.

---

## 📝 Catatan untuk README di GitHub

Setelah selesai, update README ini dengan:
- Screenshot topologi dari Packet Tracer
- Tabel VLSM yang sudah terisi
- Config snippet penting (router sub-interface, VLAN)
- Hasil ping test (screenshot)

---

## 🔗 Referensi

- [Subnetting Practice — subnetting.org](https://subnetting.org)
- [VLSM Calculator — subnet-calculator.com](https://www.subnet-calculator.com/subnet.php)
- [Cisco VLSM Design Guide](https://www.cisco.com/c/en/us/support/docs/ip/ip-addressing-services/index.html)

---

## ⏭️ Lab Selanjutnya

→ **005_DHCP_Server** — Konfigurasi DHCP di router + DHCP relay agar tidak perlu assign IP manual ke setiap device.
