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

```text
   mysql {
                # If any of the files below are set, TLS encryption is enabled
                #tls {
                #       ca_file = "/etc/ssl/certs/my_ca.crt"
                #       ca_path = "/etc/ssl/certs/"
                #       certificate_file = "/etc/ssl/certs/private/client.crt"
                #       private_key_file = "/etc/ssl/certs/private/client.key"
                #       cipher = "DHE-RSA-AES256-SHA:AES128-SHA"

                #       tls_required = yes
                #       tls_check_cert = no
                #       tls_check_cert_cn = no
                #}

                # If yes, (or auto and libmysqlclient reports warnings are
                # available), will retrieve and log additional warnings from
                # the server if an error has occured. Defaults to 'auto'
                warnings = auto
        }
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

<img width="961" height="255" alt="image" src="https://github.com/user-attachments/assets/e54c6742-36ac-4ad4-a415-17749adc2e39" />

<img width="1536" height="283" alt="image" src="https://github.com/user-attachments/assets/ca946f71-2db8-4312-b02f-7e00d909738b" />

<img width="1516" height="676" alt="image" src="https://github.com/user-attachments/assets/bc5e957b-470f-4ab0-b902-611a546d66c5" />

<img width="1919" height="944" alt="image" src="https://github.com/user-attachments/assets/5707662e-5f80-4523-a436-dd5d62f898aa" />

- GUI web untuk mengelola database FreeRADIUS.

---

### **14️⃣ Menambahkan NAS, User, dan Profil Mikrotik Setelah Login phpMyAdmin**

- Setelah login ke phpMyAdmin, tambahkan NAS melalui tabel `nas` jika diperlukan:

```sql
INSERT INTO nas (nasname, shortname, type, ports, secret, description) VALUES
('172.123.123.2', 'NAS-2', 'other', 0, 'secret123', 'NAS segment 172.123.123.2'),
```

- Tambahkan user dan password di tabel `radcheck`:

```sql
INSERT INTO radcheck (username, attribute, op, value) VALUES
('clien1', 'Cleartext-Password', ':=', 'secret123'),
```

- Tambahkan profil Mikrotik untuk user di tabel `radreply`:

```sql
INSERT INTO radreply (username, attribute, op, value) VALUES
('clien1', 'Mikrotik-Group', ':=', 'profile1'),
```

- NAS, user, dan profil bisa ditambahkan secara dinamis kapan saja lewat phpMyAdmin.

---

### **15️⃣ Testing User (Opsional)**

```bash
radtest username password localhost 0 testing123
```
- Berikut link Tools untuk check Radius Server
 
https://ntradping.apponic.com/

<img width="676" height="429" alt="image" src="https://github.com/user-attachments/assets/f238f497-9552-467b-ba49-e1e7e51f578c" />

- Cek autentikasi FreeRADIUS.

---

**Catatan:**

- Pastikan firewall mengizinkan port **1812/UDP** (RADIUS Auth) dan **1813/UDP** (RADIUS Accounting).
- Semua perintah dijalankan sebagai **root** atau menggunakan `sudo`.

---

**Selesai!**\
FreeRADIUS + MySQL + phpMyAdmin siap digunakan dengan NAS, user, dan profil yang dapat ditambah melalui phpMyAdmin.

