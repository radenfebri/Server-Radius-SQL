FreeRADIUS dengan IP Pool Management
Dokumentasi ini menjelaskan langkah-langkah instalasi dan konfigurasi FreeRADIUS dengan MySQL dan manajemen IP Pool.

Prerequisites
Sistem operasi Ubuntu/Debian

Akses root/sudo

Instalasi Paket
1. Update Sistem dan Install FreeRADIUS
bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y freeradius freeradius-mysql freeradius-utils
2. Install Web Server dan Database
bash
sudo apt install -y apache2 php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,db,mbstring,xml,curl} mysql-server phpMyAdmin
Konfigurasi Database
1. Buat Database dan User Radius
bash
sudo mysql -u root -p
Jalankan perintah SQL berikut:

sql
CREATE DATABASE radius;
CREATE USER 'radius'@'localhost' IDENTIFIED BY 'password_radius';
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost';
FLUSH PRIVILEGES;
EXIT;
2. Import Skema Database
bash
mysql -u radius -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
mysql -u radius -p radius < /etc/freeradius/3.0/mods-config/sql/ippool/mysql/schema.sql
Konfigurasi FreeRADIUS
1. Aktifkan Modul SQL
bash
ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
ln -s /etc/freeradius/3.0/mods-available/sqlippool /etc/freeradius/3.0/mods-enabled/
2. Konfigurasi SQL IP Pool
Edit file /etc/freeradius/3.0/mods-available/sqlippool:

bash
nano /etc/freeradius/3.0/mods-available/sqlippool
Tambahkan konfigurasi berikut:

text
sqlippool {
    sql_module_instance = "sql"
    dialect = "mysql"
    pool_name = "Pool-Name"
    ippool_table = "radippool"
    lease_duration = 60 #satu menit
    attribute_name = Framed-IP-Address
    req_attribute_name = Framed-IP-Address
    pool_key = "%{User-Name}"
    $INCLUDE ${modconfdir}/sql/ippool/${dialect}/queries.conf
}
3. Konfigurasi Situs Default
Edit file /etc/freeradius/3.0/sites-enabled/default:

bash
nano /etc/freeradius/3.0/sites-enabled/default
Tambahkan sqlippool pada section authorize dan post-auth:

text
authorize {
    ...
    sqlippool
    ...
}

post-auth {
    ...
    sqlippool
    ...
}
4. Konfigurasi Query IP Pool
Edit file /etc/freeradius/3.0/mods-config/sql/ippool/mysql/queries.conf:

bash
nano /etc/freeradius/3.0/mods-config/sql/ippool/mysql/queries.conf
Tambahkan query berikut:

text
allocate_existing = "\
	SELECT framedipaddress FROM ${ippool_table} \
	WHERE pool_name = '%{control:${pool_name}}' AND pool_key = '${pool_key}' \
	ORDER BY expiry_time DESC \
	LIMIT 1 \
	FOR UPDATE ${skip_locked}"

allocate_find = "\
        SELECT framedipaddress FROM ${ippool_table} \
        WHERE pool_name = '%{control:${pool_name}}' \
        AND expiry_time < NOW() \
        ORDER BY expiry_time \
        LIMIT 1 \
        FOR UPDATE ${skip_locked}"

pool_check = "\
        SELECT id FROM ${ippool_table} \
        WHERE pool_name='%{control:${pool_name}}' \
        LIMIT 1"

allocate_update = "\
        UPDATE ${ippool_table} \
        SET \
                nasipaddress = '%{NAS-IP-Address}', pool_key = '${pool_key}', \
                callingstationid = '%{Calling-Station-Id}', \
                username = '%{User-Name}', expiry_time = NOW() + INTERVAL ${lease_duration} SECOND \
        WHERE framedipaddress = '%I'"

start_update = "\
        UPDATE ${ippool_table} \
        SET \
                expiry_time = NOW() + INTERVAL ${lease_duration} SECOND \
        WHERE pool_key = '${pool_key}' \
        AND username = '%{User-Name}' \
        AND callingstationid = '%{Calling-Station-Id}' \
        AND framedipaddress = '%{${attribute_name}}'"

stop_clear = "\
UPDATE ${ippool_table} \
SET \
    nasipaddress = NULL, \
    callingstationid = NULL, \
    calledstationid = NULL, \
    username = NULL, \
    expiry_time = NOW() \
WHERE framedipaddress = '%{Framed-IP-Address}'"

alive_update = "\
        UPDATE ${ippool_table} \
        SET \
                expiry_time = NOW() + INTERVAL ${lease_duration} SECOND \
        WHERE nasipaddress = '%{%{Nas-IP-Address}:-%{Nas-IPv6-Address}}' \
        AND pool_key = '${pool_key}' \
        AND username = '%{User-Name}' \
        AND callingstationid = '%{Calling-Station-Id}' \
        AND framedipaddress = '%{${attribute_name}}'"

on_clear = "\
        UPDATE ${ippool_table} \
        SET \
                expiry_time = NOW() \
        WHERE nasipaddress = '%{%{Nas-IP-Address}:-%{Nas-IPv6-Address}}'"

off_clear = "\
        UPDATE ${ippool_table} \
        SET \
                expiry_time = NOW() \
        WHERE nasipaddress = '%{%{Nas-IP-Address}:-%{Nas-IPv6-Address}}'"
Testing dan Timezone
1. Test FreeRADIUS
bash
sudo freeradius -X
2. Set Timezone
bash
sudo timedatectl set-timezone Asia/Jakarta
Script Maintenance
1. Script Clear Expired IP
Buat file /usr/local/bin/clear_expired_ip.sh:

bash
nano /usr/local/bin/clear_expired_ip.sh
Isi dengan:

bash
#!/bin/bash
# file: /usr/local/bin/clear_expired_ip.sh

DB_USER="radius-user"
DB_PASS="P@ssw0rd"
DB_NAME="radius"
DB_HOST="localhost"
TABLE="radippool"

# Ambil daftar IP yang expired
EXPIRED_IPS=$(mysql -u $DB_USER -p$DB_PASS -h $DB_HOST $DB_NAME -N -e "SELECT framedipaddress FROM $TABLE WHERE expiry_time < NOW();")

for IP in $EXPIRED_IPS; do
    # Cek apakah IP masih aktif di radacct
    COUNT=$(mysql -u $DB_USER -p$DB_PASS -h $DB_HOST $DB_NAME -N -e "SELECT COUNT(*) FROM radacct WHERE framedipaddress='$IP' AND acctstoptime IS NULL;")
    
    if [ "$COUNT" -eq 0 ]; then
        # User sudah logout → clear IP
        mysql -u $DB_USER -p$DB_PASS -h $DB_HOST $DB_NAME -e "UPDATE $TABLE SET 
            nasipaddress=NULL,
            callingstationid=NULL,
            calledstationid=NULL,
            username=NULL,
            pool_key=NULL,
            expiry_time=NOW()
            WHERE framedipaddress='$IP';"
        echo "Cleared IP $IP"
    else
        echo "IP $IP masih digunakan, tidak di-clear"
    fi
done
2. Script Clear Old Radius Data
Buat file /usr/local/bin/clear_old_radius.sh:

bash
nano /usr/local/bi
