# Instalar FreeRADIUS+daloRADIUS+MariaDB+Apache2 en Debian 10 Buster

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

### Introducción

#### ¿Qué es `FreeRADIUS`?

[FreeRADIUS](https://freeradius.org/) es una suite `RADIUS` gratuita, modular y de alto rendimiento, desarrollada y distribuida bajo la Licencia Pública General GNU, versión 2, y se puede descargar y usar libre de costo.

`RADIUS` (Remote Authentication Dial-In User Server), es un protocolo ampliamente empleado para controlar el acceso a servicios en red. Permite gestionar la autenticación, autorización y registro de usuarios remotos sobre un determinado recurso. La tupla "autenticación, autorización y registro" es más conocida como `AAA`, proveniente del acrónimo inglés `Authentication, Authorization and Accounting`.

#### ¿Qué es `daloEADIUS`?

[daloRADIUS](http://www.daloradius.com), se basa en una implementación de `FreeRADIUS` con un servidor de base de datos que actúa como backend.

`daloRADIUS` es una aplicación avanzada de administración web `RADIUS` destinada a administrar hotspots e implementaciones de ISP de propósito general. Cuenta con gestión de usuarios, informes gráficos, contabilidad, un motor de facturación y se integra con `GoogleMaps` para la localización geográfica. Está escrito en `PHP` y `JavaScript` y utiliza una capa de abstracción de base de datos, lo que significa que es compatible con muchos sistemas de bases de datos, entre ellos los populares `MySQL`, `MariaDB`, `PostgreSQL`, `Sqlite` y `MSSQL`.

### Requisitos previos

- Instalar servidor (físico, máquina virtual o contenedor) con sistema operativo `Debian 10 Buster`, básico

- Establecer internalización `en_US.UTF-8` y zona horaria `America/Havana`

```bash
dpkg-reconfigure locales
dpkg-reconfigure tzdata
```

- Establecer nombre `FQDN` del sistema

```bash
nano /etc/hosts

127.0.0.1       localhost.localdomain    localhost
192.168.0.100   daloradius.example.tld   daloradius
```

- Establecer parámetros de red

```bash
nano /etc/network/interfaces

auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
        address 192.168.0.100
        netmask 255.255.255.0
        gateway 192.168.0.254
```

- Configurar repositorio de paquetes

```bash
nano /etc/apt/apt.conf

Acquire::Check-Valid-Until "false";
```

```bash
nano /etc/apt/sources.list

deb http://deb.debian.org/debian/ buster main contrib
deb http://deb.debian.org/debian-security/ buster/updates main contrib
deb http://deb.debian.org/debian/ buster-updates main contrib
deb http://deb.debian.org/debian/ oldstable main contrib
```

>**NOTA**: La definición del repositorio `oldstable`, obedece a la dependencia de `Daloradius` del paquete `php-bd`, no disponible en `Debian 10 Buster`.

- Actualizar el sistema

```bash
apt update && apt full-upgrade -y
```

- Crear certificado de seguridad `TLS/SSL`

```bash
openssl req -x509 -nodes -days 3650 -sha512 \
    -subj "/C=CU/ST=Provincia/L=Ciudad/O=EXAMPLE TLD/OU=IT/CN=Daloradius/emailAddress=postmaster@example.tld/" \
    -addext "subjectAltName = DNS:daloradius.example.tld,IP:192.168.0.100" \
    -newkey rsa:4096 \
    -out /etc/ssl/certs/daloradius.crt \
    -keyout /etc/ssl/private/daloradius.key

openssl dhparam -out /etc/ssl/dh2048.pem 2048
chmod 0444 /etc/ssl/certs/daloradius.crt
chmod 0400 /etc/ssl/private/daloradius.key
```

- Descargar y desplegar `Daloradius`

```bash
cd /opt
wget https://github.com/lirantal/daloradius/archive/master.zip
unzip daloradius-master.zip
mv daloradius-master/ daloradius/
```

### Instalar y configurar servidor web `Apache2`

- Instalar paquetes necesarios

```bash
apt install apache2 php libapache2-mod-php php-{gd,common,mail,mail-mime,mysql,pear,mbstring,xml,curl} unzip
apt install php-db && apt-mark hold php-db
```

- Definir zona horaria

```bash
sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/;
    s/^;cgi\.fix_pathinfo=.*$/cgi\.fix_pathinfo = 0/" \
    /etc/php/7*/cli/php.ini
```

- Crear `VirtualHost`

```bash
nano /etc/apache2/sites-available/daloradius.conf

<VirtualHost *:80>
     ServerName daloradius.example.tld
     Redirect permanent / https://daloradius.example.tld/
</VirtualHost>
<IfModule mod_ssl.c>
    <VirtualHost *:443>
        ServerName daloradius.example.tld
        Protocols h2 h2c http/1.1
        ProtocolsHonorOrder Off
        ServerAdmin postmaster@example.tld
        DirectoryIndex index.php
        DocumentRoot /opt/daloradius/
        <Directory "/opt/daloradius/">
            Options +Indexes +FollowSymLinks +MultiViews
            AllowOverride All
            Require all granted
        </Directory>
        SSLEngine on
        SSLCertificateFile /etc/ssl/certs/daloradius.crt
        SSLCertificateKeyFile /etc/ssl/private/daloradius.key
        SSLOpenSSLConfCmd DHParameters "/etc/ssl/dh2048.pem"
        SSLProtocol -all +TLSv1.3 +TLSv1.2
        SSLOpenSSLConfCmd Curves X25519:secp521r1:secp384r1:prime256v1
        SSLCipherSuite EECDH+AESGCM:EDH+AESGCM
        SSLHonorCipherOrder on
        SSLCompression off
        SSLOptions +StrictRequire
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
            SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
            SSLOptions +StdEnvVars
        </Directory>
        BrowserMatch "MSIE [2-6]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0
        BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
        ErrorLog ${APACHE_LOG_DIR}/daloradius_error.log
        CustomLog ${APACHE_LOG_DIR}/daloradius_access.log combined
    </VirtualHost>
</IfModule>
```

- Activar módulos necesarios, `VirtualHost` y reiniciar el servicio

```bash
a2enmod rewrite ssl
a2dissite 000-default.conf
a2ensite daloradius.conf
systemctl restart apache2.service
```

### Instalar y configurar servidor de base de datos `MariaDB`

- Instalar paquetes necesarios

```bash
apt install mariadb-server mariadb-client
```

- Asegurar el servicio

```bash
mysql_secure_installation

Enter current password for root (enter for none):
Change the root password? [Y/n] y
New password:
Re-enter new password:
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```

```bash
mysql -u root

MariaDB [(none)]> UPDATE mysql.user SET plugin = 'mysql_native_password' WHERE User = 'root';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> QUIT;
```

- Crear usuario administrativo de bases de datos

```bash
mysql -u root -p

MariaDB [(none)]> GRANT ALL ON *.* TO 'db_admin'@'%' IDENTIFIED BY 'P@s$w0rd' WITH GRANT OPTION;
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> QUIT;
```

- Reiniciar el servicio

```bash
systemctl restart mariadb.service
```

### Instalar y configurar servidor `FreeRADIUS`

Si se desea utilizar la última versión estable proporcionada por los desarrolladores, se deben seguir estos sencillos pasos:

- Agregar la firma `PGP` del repositorio de paquetes `FreeRADIUS`

```bash
apt-key adv --keyserver keys.gnupg.net --recv-key 0x41382202
```

- Agregar al final del fichero `/etc/apt/sources.list`

```bash
deb https://packages.networkradius.com/releases/debian-buster/ buster main
```

- Actualizar base de datos de paquetes
```bash
 apt update
```

- Instalar paquetes necesarios

```bash
apt install freeradius freeradius-mysql freeradius-utils
```

- Crear e inicializar base de datos

```bash
mysql -u db_admin -p

MariaDB [(none)]> CREATE DATABASE radius;
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> QUIT;

mysql -u db_admin -p radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
```

- Configurar acceso a la base de datos

```bash
ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```

```bash
nano /etc/freeradius/3.0/mods-enabled/sql

sql {
    driver = "rlm_sql_mysql"
    dialect = "mysql"
    server = "localhost"
    port = 3306
    login = "db_admin"
    password = "P@s$w0rd"
    radius_db = "radius"
}
read_clients = yes
client_table = "nas"
```

> **NOTA**: Sólo se muestran los parámetros modificados.

Además, se deben descomentar las líneas de configuración que contengan `-sql`, existentes en los ficheros `/etc/freeradius/3.0/sites-available/default` y `/etc/freeradius/3.0/sites-available/inner-tunnel`.

- Establecer permisos y reiniciar el servicio

```bash
chgrp -h freerad /etc/freeradius/3.0/mods-available/sql
chown -R freerad:freerad /etc/freeradius/3.0/mods-enabled/sql
systemctl restart freeradius.service
```

### Configurar `Daloradius`

- Poblar la base de datos

```bash
cd /opt/daloradius
mysql -u db_admin -p radius < contrib/db/mysql-daloradius.sql
mysql -u db_admin -p radius < contrib/db/fr2-mysql-daloradius-and-freeradius.sql
```

- Establecer permisos

```bash
chown -R www-data:www-data daloradius/
cp library/daloradius.conf.php.sample library/daloradius.conf.php
chmod 664 library/daloradius.conf.php
```

- Configurar acceso a la base de datos

```bash
nano library/daloradius.conf.php
```
```php
$configValues['CONFIG_DB_ENGINE'] = 'mysqli';
$configValues['CONFIG_DB_HOST'] = 'localhost';
$configValues['CONFIG_DB_PORT'] = '3306';
$configValues['CONFIG_DB_USER'] = 'db_admin';
$configValues['CONFIG_DB_PASS'] = 'P@s$w0rd';
$configValues['CONFIG_DB_NAME'] = 'radius';
```

> **NOTA**: Sólo se muestran los parámetros modificados.

- Reiniciar los servicios correspondientes

```bash
systemctl restart freeradius.service apache2.service
```

### Acceder a `Daloradius`

Finalmente, acceder a la aplicación web, introduciendo la dirección `http://daloradius.example.tld/`, en el navegador de preferencia y usar el par usuario/contraseña (`administrator/radius`) para efectuar el `login` y comenzar a explotar el sistema.

> **NOTA**: Se debe cambiar la contraseña por defecto del usuario `Administrator`; así como renombrar el fichero `/opt/daloradius/update.php`.

### Referencias

* Fichero `INSTALL` del paquete [daloradius-master.zip](https://github.com/lirantal/daloradius/archive/master.zip)
* [FreeRADIUS HOWTOs](https://wiki.freeradius.org/guide/HOWTO)
* [Install FreeRADIUS and Daloradius on Debian 10 (Buster)](https://computingforgeeks.com/install-freeradius-and-daloradius-on-debian/)
* [How To Install MariaDB on Debian 10 Buster](https://computingforgeeks.com/how-to-install-mariadb-on-debian-10-buster/)
* [Error installing php-db in Debian 10 Buster](https://unix.stackexchange.com/questions/545755/error-installing-php-db-in-debian-10-buster)
* [Install FreeRADIUS with daloRADIUS on Debian 9](https://kifarunix.com/install-freeradius-with-daloradius-on-debian-9/)
