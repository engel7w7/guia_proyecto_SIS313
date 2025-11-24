# 🏰 Proyecto: Sentinel-LAN - Guía Maestra Definitiva v3.0

> **Arquitectura de Alta Disponibilidad con Nextcloud**  
> Sistema distribuido en 4 VMs con acceso seguro vía Jump Server

---

## 📋 Tabla de Direcciones IP

| VM   | Hostname | IP Gestión (ens18)   | IP Proyecto (vlan101) | Rol                    |
|------|----------|----------------------|-----------------------|------------------------|
| VM 1 | proxy    | 192.168.100.214      | 192.168.101.2         | Entrada Web (Nginx)    |
| VM 2 | app      | 192.168.100.196      | 192.168.101.4         | Nextcloud + PHP 8.3    |
| VM 3 | db       | 192.168.100.178      | 192.168.101.5         | MariaDB                |
| VM 4 | infra    | 192.168.100.209      | 192.168.101.3         | NFS / DNS / Backups    |

---

## 🎯 Características de la Arquitectura

- **Arquitectura**: 4 VMs + Salto SSH (Jump Server)
- **Red**: Puerto 81 interno (evasión de bloqueos) y puerto 443 externo
- **Software**: PHP 8.3, Nginx, MariaDB
- **Fixes**: Corrección de túneles, permisos NFS, límites de subida (1GB) y config.php para proxy inverso

---

## 🟢 Fase 1: Configuración de Red y Sistema Base

### 1.1. Configurar Netplan (En las 4 VMs)

Edita el archivo de configuración de red en cada VM:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Esto garantiza acceso a internet (vía `ens18`) y comunicación interna (vía `vlan101`).

#### Plantilla de configuración:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18:
      dhcp4: no
      addresses: ["192.168.100.XXX/24"] # <--- ¡CAMBIA ESTO POR LA IP DE GESTIÓN DE CADA VM!
      routes:
        - to: default
          via: 192.168.100.1 # Gateway de la facultad
      nameservers:
        addresses: [8.8.8.8, 192.168.101.3] # Google primero para asegurar descargas
  vlans:
    vlan101:
      link: ens18
      id: 101
      addresses: ["192.168.101.X/29"] # <--- ¡CAMBIA ESTO POR LA IP DE PROYECTO!
```

#### Aplicar la configuración:

```bash
sudo netplan apply
```

---

### 1.2. Instalación de Paquetes (En cada VM)

#### 📦 VM proxy (192.168.100.214):

```bash
sudo apt update && sudo apt install -y nginx prometheus-node-exporter
```

---

#### 📦 VM app (192.168.100.196) - ¡OJO! PHP 8.3:

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install -y nginx php8.3-fpm php8.3-mysql php8.3-xml php8.3-gd \
    php8.3-curl php8.3-zip php8.3-mbstring php8.3-intl php8.3-bcmath \
    php8.3-apcu unzip nfs-common prometheus-node-exporter
```

---

#### 📦 VM db (192.168.100.178):

```bash
sudo apt update && sudo apt install -y mariadb-server prometheus-node-exporter
```

---

#### 📦 VM infra (192.168.100.209):

```bash
sudo apt update && sudo apt install -y nfs-kernel-server dnsmasq prometheus grafana
```

---

## 🔵 Fase 2: Servicios de Infraestructura (VM infra)

> 🔌 **Conéctate a**: `192.168.100.209`

### 2.1. Configurar NFS (Almacenamiento)

#### Crear estructura de directorios:

```bash
sudo mkdir -p /srv/nextcloud/data
sudo chown -R www-data:www-data /srv/nextcloud/data
sudo chmod 770 /srv/nextcloud/data
```

#### Exportar a la VM APP:

```bash
echo "/srv/nextcloud/data 192.168.101.4(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
```

#### Aplicar y reiniciar servicio:

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

---

### 2.2. Configurar DNS (dnsmasq)

Edita el archivo de configuración:

```bash
sudo nano /etc/dnsmasq.conf
```

#### Agregar la siguiente configuración:

```conf
listen-address=127.0.0.1,192.168.101.3
# El dominio apunta al PROXY (101.2)
address=/nextcloud.rootcode.com.bo/192.168.101.2
server=8.8.8.8
```

#### Reiniciar el servicio:

```bash
sudo systemctl restart dnsmasq
```

---

### 2.3. Generar Certificados SSL

#### Crear directorio de trabajo:

```bash
mkdir -p ~/pki && cd ~/pki
```

#### Generar Autoridad Certificadora (CA):

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
    -subj "/CN=RootCode-CA"
```

#### Generar certificado para el dominio:

```bash
openssl genrsa -out cloud.key 4096
openssl req -new -key cloud.key -out cloud.csr \
    -subj "/CN=nextcloud.rootcode.com.bo"
openssl x509 -req -in cloud.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out cloud.crt -days 825 -sha256
```

#### Preparar certificado para Nginx:

```bash
sudo mkdir -p /etc/ssl/private
sudo bash -c 'cat cloud.key cloud.crt ca.crt > /etc/ssl/private/cloud.pem'
sudo chmod 600 /etc/ssl/private/cloud.pem
```

---

## 🟡 Fase 3: Base de Datos (VM db)

> 🔌 **Conéctate a**: `192.168.100.178`

### 3.1. Asegurar la instalación de MariaDB

```bash
sudo mysql_secure_installation
```

Responde a las preguntas de seguridad (establecer contraseña root, remover usuarios anónimos, etc.)

---

### 3.2. Configurar acceso remoto

Edita el archivo de configuración:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Busca y **cambia** la línea:

```ini
bind-address = 192.168.101.5
```

Reinicia el servicio:

```bash
sudo systemctl restart mariadb
```

---

### 3.3. Crear base de datos y usuario

Accede a MariaDB:

```bash
sudo mysql -u root -p
```

Ejecuta los siguientes comandos SQL:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

CREATE USER 'ncuser'@'192.168.101.4' IDENTIFIED BY 'P@ssword_Segura_2025';

GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'192.168.101.4';

FLUSH PRIVILEGES;

EXIT;
```

---

## 🟠 Fase 4: Aplicación (VM app)

> 🔌 **Conéctate a**: `192.168.100.196`

### 4.1. Montar almacenamiento NFS

#### Crear punto de montaje:

```bash
sudo mkdir -p /mnt/nextcloud_data
```

#### Montar volumen NFS:

```bash
sudo mount 192.168.101.3:/srv/nextcloud/data /mnt/nextcloud_data
```

#### Agregar al fstab para montaje automático:

```bash
echo "192.168.101.3:/srv/nextcloud/data /mnt/nextcloud_data nfs defaults 0 0" | sudo tee -a /etc/fstab
```

---

### 4.2. Descargar e instalar Nextcloud

#### Descargar la última versión:

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
```

#### Mover a la ubicación web:

```bash
sudo rm -rf /var/www/nextcloud
sudo mv nextcloud /var/www/
```

#### Asignar permisos correctos:

```bash
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chown -R www-data:www-data /mnt/nextcloud_data
```

---

### 4.3. Configuración de PHP (Límites y Tipos)

#### Aumentar límites de PHP

Edita el archivo de configuración:

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

Busca y modifica las siguientes líneas:

```ini
memory_limit = 512M
upload_max_filesize = 1024M
post_max_size = 1024M
```

---

#### Corregir tipos MIME (Error JS)

Edita el archivo de tipos MIME:

```bash
sudo nano /etc/nginx/mime.types
```

Dentro del bloque `types { }`, añade:

```nginx
application/javascript  mjs;
```

---

#### Reiniciar PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

---

### 4.4. Configurar Nginx (Backend Puerto 81)

#### Crear archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

#### Agregar la siguiente configuración:

```nginx
server {
    listen 81; # <-- PUERTO 81 PARA EVADIR BLOQUEO
    server_name nextcloud.rootcode.com.bo 192.168.101.4;
    root /var/www/nextcloud;
    index index.php;
    client_max_body_size 1024M;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
        deny all;
    }

    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;
        try_files $fastcgi_script_name =404;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        
        fastcgi_pass unix:/run/php/php8.3-fpm.sock; # <-- PHP 8.3 CORRECTO
    }
}
```

---

#### Activar el sitio y reiniciar Nginx:

```bash
sudo ln -sf /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t  # Verificar sintaxis
sudo systemctl restart nginx
```

---

## 🟣 Fase 5: Proxy Inverso (VM proxy)

> 🔌 **Conéctate a**: `192.168.100.214`

### 5.1. Copiar Certificado SSL

#### Crear directorio para certificados:

```bash
sudo mkdir -p /etc/ssl/private
```

#### Copiar certificado desde VM infra:

```bash
sudo scp adminsrv@192.168.100.209:/etc/ssl/private/cloud.pem /etc/ssl/private/cloud.pem
```

#### Asignar permisos restrictivos:

```bash
sudo chmod 600 /etc/ssl/private/cloud.pem
```

---

### 5.2. Configurar Nginx (Frontend)

#### Crear archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/cloud.lan
```

#### Agregar la siguiente configuración:

```nginx
upstream nextcloud_backend {
    server 192.168.101.4:81; # <-- APUNTA AL PUERTO 81
}

server {
    listen 80;
    server_name nextcloud.rootcode.com.bo;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name nextcloud.rootcode.com.bo;

    ssl_certificate /etc/ssl/private/cloud.pem;
    ssl_certificate_key /etc/ssl/private/cloud.pem;
    client_max_body_size 1024M;

    location / {
        proxy_pass http://nextcloud_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

---

#### Activar el sitio y reiniciar Nginx:

```bash
sudo ln -sf /etc/nginx/sites-available/cloud.lan /etc/nginx/sites-enabled/cloud.lan
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t  # Verificar sintaxis
sudo systemctl restart nginx
```

---

## 🔴 Fase 6: Firewalls (UFW) - ¡CRÍTICO!

Ejecuta esto en cada VM para evitar el `Connection refused` o `Timed Out`.

### VM proxy (100.214):

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp # SSH Gestión
sudo ufw allow 443/tcp # ACCESO WEB (Túnel)
sudo ufw allow 80/tcp
sudo ufw enable
```

### VM app (100.196):

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.2 to any port 81 proto tcp # Backend 81
sudo ufw enable
```

### VM db (100.178):

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.4 to any port 3306 proto tcp # MariaDB desde APP
sudo ufw enable
```

### VM infra (100.209):

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.4 to any port 2049 proto tcp # NFS desde APP
sudo ufw allow from 192.168.101.0/29 to any port 53 proto udp # DNS interno
sudo ufw enable
```

---

## ⚫ Fase 7: Configuración de Acceso Cliente

### 7.1. Archivo hosts (PC Windows)

Abre **PowerShell como Administrador** y ejecuta:

```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```

Agrega la siguiente línea al final:

```text
127.0.0.1    nextcloud.rootcode.com.bo
```

Guarda y cierra el archivo.

---

### 7.2. Túnel SSH (PowerShell)

Abre **PowerShell** y ejecuta el siguiente comando (déjalo abierto):

```powershell
ssh -J usrproxy@201.131.45.42 adminsrv@192.168.100.214 -L 8443:127.0.0.1:443 -N
```

> 💡 **Nota**: Este túnel debe permanecer activo mientras uses Nextcloud.

---

### 7.3. Instalación Web de Nextcloud

#### Accede al instalador:

Abre tu navegador y ve a:

```
https://nextcloud.rootcloud.com.bo:8443
```

#### Completa el formulario con los siguientes datos:

| Campo                      | Valor                        |
|----------------------------|------------------------------|
| **Usuario Administrador**  | (elige uno)                  |
| **Contraseña**             | (elige una segura)           |
| **Carpeta de datos**       | `/mnt/nextcloud_data`        |
| **Tipo de base de datos**  | MySQL/MariaDB                |
| **Usuario BD**             | `ncuser`                     |
| **Contraseña BD**          | `P@ssword_Segura_2025`       |
| **Nombre BD**              | `nextcloud`                  |
| **Host BD**                | `192.168.101.5`              |

Haz clic en **Finalizar configuración**.

---

### 7.4. Ajuste Final config.php (Post-Install)

Después de la instalación, edita la configuración en la **VM app**:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Agrega/modifica las siguientes líneas:

```php
  'trusted_domains' => 
  array (
    0 => '127.0.0.1:8443',
    1 => 'nextcloud.rootcode.com.bo:8443',
  ),
  'overwrite.cli.url' => 'https://nextcloud.rootcode.com.bo:8443',
  'overwritehost' => 'nextcloud.rootcode.com.bo:8443',
  'overwriteprotocol' => 'https',
  'trusted_proxies' => ['192.168.101.2'],
  'memcache.local' => '\OC\Memcache\APCu',
```

#### Reinicia PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

---

## 🟤 Fase 8: Backups Automatizados

> 🔌 **Ejecuta en VM infra** como usuario `adminsrv`

### 8.1. Configurar acceso SSH sin contraseña

#### Generar par de llaves SSH:

```bash
ssh-keygen -t rsa -b 4096 -C "backup@infra"
```

Presiona `Enter` para aceptar la ubicación predeterminada y no uses passphrase.

#### Copiar llave pública a VM db:

```bash
ssh-copy-id adminsrv@192.168.101.5
```

---

### 8.2. Configurar credenciales de base de datos

#### En la VM db, crear archivo de credenciales:

Conéctate a la **VM db** (`192.168.100.178`) y ejecuta:

```bash
nano /home/adminsrv/.my.cnf
```

Agrega el siguiente contenido:

```ini
[client]
user=ncuser
password=P@ssword_Segura_2025
```

#### Asignar permisos restrictivos:

```bash
chmod 600 /home/adminsrv/.my.cnf
```

---

### 8.3. Crear script de backup

#### En la VM infra, crear el directorio de scripts:

```bash
sudo mkdir -p /opt/admin_scripts
```

#### Crear el script de backup:

```bash
sudo nano /opt/admin_scripts/backup_completo.sh
```

#### Agregar el siguiente contenido:

```bash
#!/bin/bash

# Configuración
DB_HOST="192.168.101.5"
DB_NAME="nextcloud"
SSH_USER="adminsrv"
BACKUP_DIR="/var/backups/nextcloud_full"
DATE=$(date +%Y%m%d_%H%M)

# Crear directorio del backup
mkdir -p $BACKUP_DIR/$DATE

# Dump remoto de la base de datos
ssh $SSH_USER@$DB_HOST "mysqldump --defaults-file=/home/adminsrv/.my.cnf $DB_NAME" | \
    gzip > $BACKUP_DIR/$DATE/db.sql.gz

# Copia local de archivos (desde NFS)
tar -czf $BACKUP_DIR/$DATE/data.tar.gz -C /srv/nextcloud data

# Limpieza de backups antiguos (mantener últimos 7 días)
find $BACKUP_DIR -maxdepth 1 -mtime +7 -exec rm -rf {} \;

echo "Backup completado: $BACKUP_DIR/$DATE"
```

#### Asignar permisos de ejecución:

```bash
sudo chmod +x /opt/admin_scripts/backup_completo.sh
```

#### Crear directorio de backups:

```bash
sudo mkdir -p /var/backups/nextcloud_full
```

---

### 8.4. Automatización con Cron

#### Editar crontab del usuario root:

```bash
sudo crontab -e
```

#### Agregar la siguiente línea:

```cron
0 3 * * * /opt/admin_scripts/backup_completo.sh
```

> 💡 **Nota**: Esto ejecutará el backup todos los días a las 3:00 AM.

#### Verificar que el cron se agregó correctamente:

```bash
sudo crontab -l
```

---

### 8.5. Probar el backup manualmente

```bash
sudo /opt/admin_scripts/backup_completo.sh
```

#### Verificar que se creó el backup:

```bash
ls -lh /var/backups/nextcloud_full/
```

---

## 📊 Diagrama de Arquitectura

```
                    Internet
                       |
                 [Jump Server]
               201.131.45.42:22
                       |
            ┌──────────┴──────────┐
            │                     │
     [Gestión: ens18]      [Proyecto: vlan101]
      192.168.100.x         192.168.101.x
            │                     │
    ┌───────┼─────────────────────┼───────┐
    │       │                     │       │
┌───┴───┐ ┌─┴──┐ ┌────┐        ┌─┴──┐ ┌──┴──┐
│ Proxy │ │App │ │ DB │        │Infra│ │Monit│
│ .214  │ │.196│ │.178│        │.209 │ │     │
│ :443  │ │:81 │ │:3306       │NFS  │ │     │
└───────┘ └────┘ └────┘        │DNS  │ └─────┘
                                │Backup│
                                └──────┘
```

---

## 🔧 Troubleshooting Común

### ❌ Error: "Connection refused"

**Diagnóstico:**

```bash
# Verificar estado del firewall
sudo ufw status

# Verificar servicios activos
sudo systemctl status nginx
sudo systemctl status php8.3-fpm
```

**Solución:**
- Revisar reglas de UFW (ver Fase 6)
- Verificar que los servicios estén corriendo
- Comprobar que los puertos estén en escucha: `sudo netstat -tlnp`

---

### ❌ Error: "Trusted domain"

**Diagnóstico:**

```bash
sudo cat /var/www/nextcloud/config/config.php | grep trusted_domains
```

**Solución:**

Editar el archivo:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Añadir el dominio/IP en el array `trusted_domains` (ver Fase 7.4)

---

### ❌ Error: "Can't write to data directory"

**Diagnóstico:**

```bash
ls -la /mnt/nextcloud_data
```

**Solución:**

```bash
sudo chown -R www-data:www-data /mnt/nextcloud_data
sudo chmod 770 /mnt/nextcloud_data
```

---

### ❌ NFS no monta

**Diagnóstico:**

```bash
# En VM infra
sudo exportfs -v

# En VM app
showmount -e 192.168.101.3
```

**Solución:**

```bash
# Probar montaje manual
sudo mount -t nfs 192.168.101.3:/srv/nextcloud/data /mnt/test

# Verificar conectividad
ping 192.168.101.3

# Verificar firewall permite NFS (puerto 2049)
sudo ufw status
```

---

### ❌ Error: Archivos grandes no se suben

**Diagnóstico:**

Verificar límites en `/etc/php/8.3/fpm/php.ini` y en la configuración de Nginx.

**Solución:**

Ver Fase 4.3 para ajustar `upload_max_filesize`, `post_max_size` y `client_max_body_size`.

---

### ❌ Túnel SSH se desconecta

**Diagnóstico:**

El túnel SSH puede cerrarse por inactividad.

**Solución:**

Agregar opciones al comando SSH:

```powershell
ssh -J usrproxy@201.131.45.42 adminsrv@192.168.100.214 -L 8443:127.0.0.1:443 -N -o ServerAliveInterval=60
```

---

## 📝 Checklist de Verificación

- [ ] Todas las VMs tienen conectividad en ambas redes (ens18 y vlan101)
- [ ] NFS montado correctamente en VM app
- [ ] MariaDB acepta conexiones desde 192.168.101.4
- [ ] Certificados SSL copiados a VM proxy
- [ ] UFW configurado en todas las VMs
- [ ] Túnel SSH funcional desde cliente
- [ ] Nextcloud accesible vía `https://nextcloud.rootcode.com.bo:8443`
- [ ] Archivo de 100MB sube correctamente (prueba límites)
- [ ] Backup automatizado funciona (verificar en `/var/backups/nextcloud_full`)
- [ ] DNS resuelve internamente (desde VMs: `dig @192.168.101.3 nextcloud.rootcode.com.bo`)

---

## 📚 Referencias

- [Documentación Oficial Nextcloud](https://docs.nextcloud.com/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [MariaDB Documentation](https://mariadb.org/documentation/)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)

---

## 👥 Equipo del Proyecto

**Curso**: SIS313 - Administración de Redes  
**Proyecto**: Sentinel-LAN  
**Versión**: 3.0 (Definitiva)  
**Fecha**: Noviembre 2025

---

> **Nota Importante**: Esta guía consolida todas las correcciones realizadas durante la depuración del proyecto. Sigue los pasos en orden secuencial para garantizar una implementación exitosa.
