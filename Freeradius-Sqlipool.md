sudo apt update && sudo apt upgrade -y


sudo apt install -y freeradius freeradius-mysql freeradius-utils


sudo apt install -y apache2 php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,db,mbstring,xml,curl} mysql-server phpMyAdmin


sudo mysql -u root -p

# Buat user radius dan database
CREATE DATABASE radius;
CREATE USER 'radius'@'localhost' IDENTIFIED BY 'password_radius';
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost';
FLUSH PRIVILEGES;
EXIT;



mysql -u radius -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
mysql -u radius -p radius < /etc/freeradius/3.0/mods-config/sql/ippool/mysql/schema.sql


ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
ln -s /etc/freeradius/3.0/mods-available/sqlippool /etc/freeradius/3.0/mods-enabled/


nano /et/freeradius/3.0/mods-available/sqlippool
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



nano /etc/freeradius/3.0/sites-enabled/default
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




nano /etc/freeradius/3.0# cat mods-config/sql/ippool/mysql/queries.conf

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


sudo freeradius -X


sudo timedatectl set-timezone Asia/Jakarta


nano /usr/local/bin/clear_expired_ip.sh
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




nano /usr/local/bin/clear_old_radius.sh
root@server:/etc/freeradius/3.0# cat /usr/local/bin/clear_old_radius.sh 
#!/bin/bash
# file: /usr/local/bin/clean_old_radius.sh

DB_USER="radius-user"
DB_PASS="P@ssw0rd"
DB_NAME="radius"
DB_HOST="localhost"

# Hapus data di radacct yang lebih dari 30 hari
mysql -u $DB_USER -p$DB_PASS -h $DB_HOST $DB_NAME -e "
DELETE FROM radacct
WHERE acctstarttime < NOW() - INTERVAL 30 DAY;
"

# Hapus data di radpostauth yang lebih dari 30 hari
mysql -u $DB_USER -p$DB_PASS -h $DB_HOST $DB_NAME -e "
DELETE FROM radpostauth
WHERE authdate < NOW() - INTERVAL 30 DAY;
"

echo "Old radius data older than 30 days cleaned at $(date)"



crontab -e                                                                  
* * * * * /usr/local/bin/clear_expired_ip.sh
0 3 1 * * /usr/local/bin/clear_old_radius.sh



chmod +x /usr/local/bin/clear_expired_ip.sh
chmod +x /usr/local/bin/clear_old_radius.sh



