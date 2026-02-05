# 🧪 LAB 02 — Routing Basic (Router as Gateway)

### Target:
👉 PC beda network bisa komunikasi lewat router.

---

## 🖥 Device Specification

| Device | Model | Description |
|--------|-------|-------------|
| Router | Cisco 1941 | Used for basic routing |
| Switch | Cisco 2960 | Layer 2 switch |
| PC     | Generic PC | End device |



## Topologi
PC0 ─ Switch0 ─ Router ─ Switch1 ─ PC1

---

## 📌 IP Addressing Plan

Pada lab ini digunakan dua network yang berbeda:

### Network A
192.168.1.0/24


### Network B
192.168.2.0/24


### IP Assignment

| Device | Interface | IP Address   | Subnet Mask     | Default Gateway |
|--------|-----------|--------------|-----------------|-----------------|
| PC0    | Fa0       | 192.168.1.2  | 255.255.255.0   | 192.168.1.1     |
| R1     | Fa0/0     | 192.168.1.1  | 255.255.255.0   | -               |
| R1     | Fa0/1     | 192.168.2.1  | 255.255.255.0   | -               |
| PC1    | Fa0       | 192.168.2.2  | 255.255.255.0   | 192.168.2.1     |

> Default Gateway pada PC mengarah ke interface router pada network masing-masing.

---

## ⚙️ Router Configuration

Konfigurasi dilakukan melalui CLI pada router.

### Enter Privileged Mode
```
enable
configure terminal
```

### Configure Interface Fa0/0 (Network A)
```
interface fa0/0
ip address 192.168.1.1 255.255.255.0
no shutdown
exit
```

### Configure Interface Fa0/1 (Network B)
```
interface fa0/1
ip address 192.168.2.1 255.255.255.0
no shutdown
exit
```


### Save Configuration
```
end
write memory
```

---

## 🔍 Verification & Testing

### Check Interface Status
```
show ip interface brief
```
**Expected result:**
```
FastEthernet0/0 up up
FastEthernet0/1 up up
```

### Connectivity Test

Dari PC0:
ping 192.168.2.2

Jika reply diterima, maka routing berjalan dengan baik.

---

## 📖 Difference Between Router and Switch

| Aspect  | Switch                          | Router                          |
|---------|---------------------------------|----------------------------------|
| Layer   | Layer 2 (Data Link)             | Layer 3 (Network)                |
| Function| Local device connection         | Inter-network connection         |
| Address | MAC Address                     | IP Address                       |
| Routing | ❌ No                           | ✅ Yes                           |
| Gateway | ❌ No                           | ✅ Yes                           |

### Explanation

**Switch**
- Menghubungkan device dalam satu LAN
- Bekerja pada layer Data Link
- Tidak mendukung routing

**Router**
- Menghubungkan network yang berbeda
- Bekerja pada layer Network
- Berfungsi sebagai Default Gateway
- Menentukan jalur komunikasi

---

## 📚 Key Learnings

- Konsep Default Gateway
- Routing antar network
- Konfigurasi interface router
- CLI dasar Cisco
- Troubleshooting konektivitas

---

## 📝 Notes

- Pastikan semua interface dalam status `up`
- Periksa IP Address dan Gateway
- Gunakan kabel yang sesuai
- Lakukan testing setiap perubahan konfigurasi

---

## 🎯 Next Lab

➡️ Lab 03 — VLAN & Inter-VLAN Routing

Akan membahas segmentasi network menggunakan VLAN.






