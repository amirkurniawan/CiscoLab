# 🧪 Lab 01 — Basic LAN (Two PCs & One Switch)

Lab ini membahas konfigurasi jaringan LAN paling dasar menggunakan **2 PC dan 1 Switch** di Cisco Packet Tracer.

Tujuan utama:
> Memahami komunikasi dasar dalam satu jaringan lokal (LAN).

---

## 📌 Topology

PC0 ─── Switch ─── PC1


---

## 🖥 Devices Used

| Device | Type   | Quantity |
|--------|--------|----------|
| PC     | End Device | 2 |
| Switch | 2960   | 1 |

---

## 🌐 IP Addressing

| Device | Interface | IP Address     | Subnet Mask     |
|--------|-----------|----------------|-----------------|
| PC0    | Fa0       | 192.168.1.1    | 255.255.255.0   |
| PC1    | Fa0       | 192.168.1.2    | 255.255.255.0   |

> Note: Default Gateway tidak digunakan karena masih dalam satu network.

---

## ⚙️ Configuration Steps

### 1️⃣ Create Topology
- Tambahkan 2 PC dan 1 Switch
- Hubungkan menggunakan **Copper Straight-Through**

### 2️⃣ Configure IP Address
Pada masing-masing PC:

Desktop → IP Configuration → Set IP manually

### 3️⃣ Verify Connection
Gunakan Command Prompt pada PC0:

