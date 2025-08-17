**Tutorial Instalasi FreeRADIUS + MySQL + phpMyAdmin di Ubuntu/Debian**

---

### **1️⃣ Update & Upgrade Sistem**

```bash
sudo apt update
sudo apt upgrade
```

- Memperbarui daftar paket dan mengupgrade sistem.

---

### **2️⃣ Instalasi Apache**

```bash
sudo apt -y install apache2
```

- Apache2 sebagai web server.

---

### **3️⃣ Instalasi PHP & Modul Tambahan**

```bash
sudo apt -y install php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,db,mbstring,xml,curl}
```

- PHP dan modul untuk kebutuhan web & RADIUS.

---

### **4️⃣ Instalasi MySQL**

```bash
sudo apt -y install mysql-server
sudo mysql_secure_installation
```

- Amankan MySQL, buat password root, hapus user anonim.

---

### **5️⃣ Instalasi FreeRADIUS + MySQL Module + Utils**

```bash
sudo apt -y install freeradius freeradius-mysql freeradius-utils
```

- FreeRADIUS server + integrasi MySQL + tools testing.

---

### **6️⃣ Menjalankan FreeRADIUS Mode Debug**

```bash
sudo systemctl stop freeradius
sudo freeradius -X
```

- Debug untuk memastikan konfigurasi benar.

---

### **7️⃣ Membuat Database & User RADIUS**

```sql
CREATE DATABASE radius;
CREATE USER 'radius'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost';
FLUSH PRIVILEGES;
```

- Database dan user untuk FreeRADIUS.

---

### **8️⃣ Import Schema MySQL FreeRADIUS**

```bash
mysql -u root -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
```

- Struktur tabel default: radcheck, radreply, nas, dll.

---

### **9️⃣ Mengaktifkan Modul SQL di FreeRADIUS**

```bash
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```

- Modul SQL aktif agar FreeRADIUS bisa konek ke MySQL.

---

### **10️⃣ Konfigurasi Modul SQL**

```bash
sudo nano /etc/freeradius/3.0/mods-enabled/sql
```

- Edit menjadi:

```text
driver = "rlm_sql_${dialect}"
dialect = "mysql"
server = "localhost"
login = "radius"
password = "P@ssw0rd"
radius_db = "radius"
read_clients = yes
client_table = "nas"
```

- Pastikan koneksi ke database berjalan.

---

### **11️⃣ Hak Akses File**

```bash
sudo chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
sudo chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql
```

- FreeRADIUS harus bisa membaca file konfigurasi.

---

### **12️⃣ Restart Service FreeRADIUS**

```bash
sudo systemctl restart freeradius.service
```

- Terapkan konfigurasi baru.

---

### **13️⃣ Instalasi phpMyAdmin**

```bash
sudo apt install -y phpmyadmin
```

- GUI web untuk mengelola database FreeRADIUS.

---

### **14️⃣ Menambahkan NAS, User, dan Profil Mikrotik Setelah Login phpMyAdmin**

- Setelah login ke phpMyAdmin, tambahkan NAS melalui tabel `nas` jika diperlukan:

```sql
INSERT INTO nas (nasname, shortname, type, ports, secret, description) VALUES
('172.123.123.2', 'NAS-2', 'other', 0, 'secret123', 'NAS segment 172.123.123.2'),
('172.123.123.3', 'NAS-3', 'other', 0, 'secret123', 'NAS segment 172.123.123.3'),
('172.123.123.4', 'NAS-4', 'other', 0, 'secret123', 'NAS segment 172.123.123.4'),
('172.123.123.5', 'NAS-5', 'other', 0, 'secret123', 'NAS segment 172.123.123.5');
```

- Tambahkan user dan password di tabel `radcheck`:

```sql
INSERT INTO radcheck (username, attribute, op, value) VALUES
('clien1', 'Cleartext-Password', ':=', 'secret123'),
('clien2', 'Cleartext-Password', ':=', 'secret123'),
('clien3', 'Cleartext-Password', ':=', 'secret123'),
('clien4', 'Cleartext-Password', ':=', 'secret123');
```

- Tambahkan profil Mikrotik untuk user di tabel `radreply`:

```sql
INSERT INTO radreply (username, attribute, op, value) VALUES
('clien1', 'Mikrotik-Profile', ':=', 'INTERNET-10M'),
('clien2', 'Mikrotik-Profile', ':=', 'INTERNET-20M'),
('clien3', 'Mikrotik-Profile', ':=', 'INTERNET-10M'),
('clien1', 'Mikrotik-Profile', ':=', 'INTERNET-20M');
```

- NAS, user, dan profil bisa ditambahkan secara dinamis kapan saja lewat phpMyAdmin.

---

### **15️⃣ Testing User (Opsional)**

```bash
radtest username password localhost 0 testing123
```

- Cek autentikasi FreeRADIUS.

---

**Catatan:**

- Pastikan firewall mengizinkan port **1812/UDP** (RADIUS Auth) dan **1813/UDP** (RADIUS Accounting).
- Semua perintah dijalankan sebagai **root** atau menggunakan `sudo`.

---

**Selesai!**\
FreeRADIUS + MySQL + phpMyAdmin siap digunakan dengan NAS, user, dan profil yang dapat ditambah melalui phpMyAdmin.

