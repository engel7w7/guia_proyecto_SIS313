# 🚀 Proyecto Final SIS313: Sentinel-LAN

> **Asignatura:** SIS313: Infraestructura, Plataformas Tecnológicas y Redes  
> **Semestre:** 2/2025  
> **Docente:** Ing. Marcelo Quispe Ortega

**Arquitectura de Alta Disponibilidad con Nextcloud**  
Sistema distribuido en 4 VMs con acceso seguro vía Jump Server

## 👥 Miembros del Equipo (Grupo 1)

| Nombre Completo | Rol en el Proyecto | Contacto (GitHub/Email) |
| :--- | :--- | :--- |
| Romero Morales Jhojan Erick | Arquitecto de Infraestructura y Redes | @engel7w7 |
| Galván Porcel Joel | Ingeniero de Seguridad y Hardening | - |
| Mamani Calizaya Jose Mario | Administrador de Base de Datos | - |
| Campos Alfaro Dilan Domingo | Especialista en Automatización y Backups | - |

---

## 🎯 I. Objetivo del Proyecto

> **Objetivo:** Diseñar e implementar una infraestructura de almacenamiento en la nube de alta disponibilidad utilizando Nextcloud, con arquitectura distribuida en 4 VMs, implementando conceptos de proxy inverso, balanceo de carga, seguridad perimetral, almacenamiento NFS y backups automatizados para garantizar la continuidad operacional y acceso seguro a los datos.

---

## 💡 II. Justificación e Importancia

> **Justificación:** Este proyecto resuelve la problemática de almacenamiento centralizado y disponibilidad de datos en entornos educativos o empresariales. Implementa conceptos de Alta Disponibilidad (T2) mediante la separación de servicios en múltiples VMs, eliminando puntos únicos de fallo. La arquitectura con proxy inverso y acceso vía Jump Server garantiza la Seguridad (T5) del sistema, mientras que la automatización de backups (T6) asegura la continuidad operacional (T1). El uso de almacenamiento NFS permite escalabilidad horizontal y la implementación de túneles SSH proporciona acceso seguro desde redes externas.

---

## 🛠️ III. Tecnologías y Conceptos Implementados

### 3.1. Tecnologías Clave

* **Nginx:** Proxy Inverso con SSL/TLS para acceso seguro, balanceo de carga y limitación de tamaño de archivos (1GB).
* **Nextcloud 29:** Plataforma de almacenamiento en la nube autohospedada con soporte para sincronización y colaboración.
* **PHP 8.3-FPM:** Motor de procesamiento backend optimizado para aplicaciones web con configuración de límites de subida.
* **MariaDB 10.x:** Sistema de gestión de base de datos relacional para almacenamiento de metadatos de Nextcloud.
* **NFS (Network File System):** Sistema de archivos distribuido para compartir almacenamiento entre servidores.
* **dnsmasq:** Servidor DNS ligero para resolución interna de nombres de dominio.
* **OpenSSL:** Generación de certificados SSL/TLS autofirmados con CA personalizada.
* **UFW (Uncomplicated Firewall):** Firewall de aplicación para control de tráfico entre VMs.
* **Prometheus & Grafana:** Stack de monitoreo para recolección de métricas y visualización de rendimiento.
* **Bash Scripts:** Automatización de backups programados con sincronización remota vía SSH.

### 3.2. Conceptos de la Asignatura Puestos en Práctica (T1 - T6)

* ✅ **Alta Disponibilidad (T2) y Tolerancia a Fallos:** Arquitectura distribuida en 4 VMs con separación de servicios (proxy, aplicación, base de datos, infraestructura). El almacenamiento NFS permite migración rápida del servicio en caso de fallo del servidor de aplicación.

* ✅ **Seguridad y Hardening (T5):** Implementación de firewall UFW con reglas restrictivas por VM, uso de certificados SSL/TLS autofirmados, túnel SSH con Jump Server para acceso remoto seguro, puerto no estándar (81) para backend evitando bloqueos, y configuración de trusted proxies en Nextcloud.

* ✅ **Automatización y Gestión (T6):** Scripts automatizados de backup con mysqldump remoto y compresión de datos, programación via cron para ejecución nocturna, y limpieza automática de backups antiguos (retención de 7 días).

* ✅ **Proxy Inverso y Seguridad de Aplicaciones (T3/T4):** Nginx como proxy inverso con redirección HTTP→HTTPS, configuración de headers de seguridad (X-Real-IP, X-Forwarded-For, X-Forwarded-Proto), y limitación de tamaño de carga (client_max_body_size).

* ✅ **Monitoreo (T4/T1):** Integración de Prometheus Node Exporter en todas las VMs para recolección de métricas de sistema, y preparación para visualización con Grafana.

* ✅ **Networking Avanzado (T3):** Implementación de VLANs (vlan101) para segmentación de red del proyecto, configuración de Netplan con red dual (gestión + proyecto), y uso de Jump Server para acceso seguro mediante túneles SSH encadenados.

---

## 🌐 IV. Diseño de la Infraestructura y Topología

### 4.1. Diseño Esquemático

#### Tabla de Direcciones IP

| VM/Host | Hostname | Rol | IP Gestión (ens18) | IP Proyecto (vlan101) | Red Lógica | SO |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VM 1** | proxy | Proxy Inverso / Frontend HTTPS | 192.168.100.214 | 192.168.101.2 | VLAN 101 | Ubuntu 22.04 |
| **VM 2** | app | Servidor Nextcloud / Backend HTTP:81 | 192.168.100.196 | 192.168.101.4 | VLAN 101 | Ubuntu 22.04 |
| **VM 3** | db | Base de Datos MariaDB | 192.168.100.178 | 192.168.101.5 | VLAN 101 | Ubuntu 22.04 |
| **VM 4** | infra | NFS / DNS / Backups / Monitoreo | 192.168.100.209 | 192.168.101.3 | VLAN 101 | Ubuntu 22.04 |
| **Jump Server** | - | Salto SSH para acceso externo | 201.131.45.42 | - | Internet | Linux |

#### Diagrama de Arquitectura



#### Fase 1: Configuración de Red y Sistema Base

### 1.1 Configurar Netplan (En las 4 VMs)

Edita `/etc/netplan/50-cloud-init.yaml` en cada VM:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Configuración de red dual: `ens18` para internet y `vlan101` para comunicación interna.

**Configuración:**

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

Aplicar cambios:

```bash
sudo netplan apply
```

### 1.2 Instalación de Paquetes

**VM proxy** (192.168.100.214):

```bash
sudo apt update && sudo apt install -y nginx prometheus-node-exporter
```

**VM app** (192.168.100.196) - Importante: PHP 8.3:

```bash
sudo add-apt-repository universe
sudo apt update
sudo apt install -y nginx php8.3-fpm php8.3-mysql php8.3-xml php8.3-gd \
    php8.3-curl php8.3-zip php8.3-mbstring php8.3-intl php8.3-bcmath \
    php8.3-apcu unzip nfs-common prometheus-node-exporter
```

**VM db** (192.168.100.178):

```bash
sudo apt update && sudo apt install -y mariadb-server prometheus-node-exporter
```

**VM infra** (192.168.100.209):

```bash
sudo apt update && sudo apt install -y nfs-kernel-server dnsmasq prometheus grafana
```

#### Fase 2: Servicios de Infraestructura (VM infra)

Conéctate a `192.168.100.209`

### 2.1 Configurar NFS (Almacenamiento)

Crear estructura de directorios:

```bash
sudo mkdir -p /srv/nextcloud/data
sudo chown -R www-data:www-data /srv/nextcloud/data
sudo chmod 770 /srv/nextcloud/data
```

Exportar a la VM APP:

```bash
echo "/srv/nextcloud/data 192.168.101.4(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
```

Aplicar cambios:

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### 2.2 Configurar DNS (dnsmasq)

Edita el archivo de configuración:

```bash
sudo nano /etc/dnsmasq.conf
```

Agregar:

```conf
listen-address=127.0.0.1,192.168.101.3
# El dominio apunta al PROXY (101.2)
address=/nextcloud.rootcode.com.bo/192.168.101.2
server=8.8.8.8
```

Reiniciar servicio:

```bash
sudo systemctl restart dnsmasq
```

### 2.3 Generar Certificados SSL

Crear directorio de trabajo:

```bash
mkdir -p ~/pki && cd ~/pki
```

Generar Autoridad Certificadora (CA):

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
    -subj "/CN=RootCode-CA"
```

Generar certificado para el dominio:

```bash
openssl genrsa -out cloud.key 4096
openssl req -new -key cloud.key -out cloud.csr \
    -subj "/CN=nextcloud.rootcode.com.bo"
openssl x509 -req -in cloud.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
    -out cloud.crt -days 825 -sha256
```

Preparar para Nginx:

```bash
sudo mkdir -p /etc/ssl/private
sudo bash -c 'cat cloud.key cloud.crt ca.crt > /etc/ssl/private/cloud.pem'
sudo chmod 600 /etc/ssl/private/cloud.pem
```

#### Fase 3: Base de Datos (VM db)

Conéctate a `192.168.100.178`

### 3.1 Asegurar instalación de MariaDB

```bash
sudo mysql_secure_installation
```

Responde a las preguntas de seguridad (contraseña root, remover usuarios anónimos, etc.)

### 3.2 Configurar acceso remoto

Edita el archivo de configuración:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Cambiar la línea:

```ini
bind-address = 192.168.101.5
```

Reinicia el servicio:

```bash
sudo systemctl restart mariadb
```

### 3.3 Crear base de datos y usuario

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

#### Fase 4: Aplicación (VM app)

Conéctate a `192.168.100.196`

### 4.1 Montar almacenamiento NFS

Crear punto de montaje:

```bash
sudo mkdir -p /mnt/nextcloud_data
```

Montar volumen NFS:

```bash
sudo mount 192.168.101.3:/srv/nextcloud/data /mnt/nextcloud_data
```

Agregar al fstab para montaje automático:

```bash
echo "192.168.101.3:/srv/nextcloud/data /mnt/nextcloud_data nfs defaults 0 0" | sudo tee -a /etc/fstab
```

### 4.2 Descargar e instalar Nextcloud

Descargar la última versión:

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
```

Mover a la ubicación web:

```bash
sudo rm -rf /var/www/nextcloud
sudo mv nextcloud /var/www/
```

Asignar permisos:

```bash
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chown -R www-data:www-data /mnt/nextcloud_data
```

### 4.3 Configuración de PHP

Aumentar límites de PHP:

Edita el archivo de configuración:

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

Modificar:

```ini
memory_limit = 512M
upload_max_filesize = 1024M
post_max_size = 1024M
```

Corregir tipos MIME (para archivos .mjs):

Edita el archivo de tipos MIME:

```bash
sudo nano /etc/nginx/mime.types
```

Añadir dentro del bloque `types { }`:

```nginx
application/javascript  mjs;
```

Reiniciar PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

### 4.4 Configurar Nginx (Backend Puerto 81)

Crear archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/nextcloud
```

Agregar:

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

Activar el sitio:

```bash
sudo ln -sf /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/nextcloud
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t  # Verificar sintaxis
sudo systemctl restart nginx
```

#### Fase 5: Proxy Inverso (VM proxy)

Conéctate a `192.168.100.214`

### 5.1 Copiar Certificado SSL

Crear directorio:

```bash
sudo mkdir -p /etc/ssl/private
```

Copiar certificado desde VM infra:

```bash
sudo scp adminsrv@192.168.100.209:/etc/ssl/private/cloud.pem /etc/ssl/private/cloud.pem
```

Asignar permisos:

```bash
sudo chmod 600 /etc/ssl/private/cloud.pem
```

### 5.2 Configurar Nginx (Frontend)

Crear archivo de configuración:

```bash
sudo nano /etc/nginx/sites-available/cloud.lan
```

Agregar:

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

Activar el sitio:

```bash
sudo ln -sf /etc/nginx/sites-available/cloud.lan /etc/nginx/sites-enabled/cloud.lan
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t  # Verificar sintaxis
sudo systemctl restart nginx
```

#### Fase 6: Configuración de Firewalls (UFW)

Configurar correctamente el firewall en cada VM para evitar errores de conexión.

### VM proxy (192.168.100.214)

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp # SSH Gestión
sudo ufw allow 443/tcp # ACCESO WEB (Túnel)
sudo ufw allow 80/tcp
sudo ufw enable
```

### VM app (192.168.100.196)

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.2 to any port 81 proto tcp # Backend 81
sudo ufw enable
```

### VM db (192.168.100.178)

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.4 to any port 3306 proto tcp # MariaDB desde APP
sudo ufw enable
```

### VM infra (192.168.100.209)

```bash
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow from 192.168.100.0/24 to any port 22 proto tcp
sudo ufw allow from 192.168.101.4 to any port 2049 proto tcp # NFS desde APP
sudo ufw allow from 192.168.101.0/29 to any port 53 proto udp # DNS interno
sudo ufw enable
```

#### Fase 7: Configuración de Acceso desde el Cliente

### 7.1 Archivo hosts (PC Windows)

Abre **PowerShell como Administrador** y ejecuta:

```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```

Agrega la siguiente línea al final:

```text
127.0.0.1    nextcloud.rootcode.com.bo
```

Guardar y cerrar.

### 7.2 Túnel SSH (PowerShell)

Abre **PowerShell** y ejecuta el siguiente comando (déjalo abierto):

```powershell
ssh -J usrproxy@201.131.45.42 adminsrv@192.168.100.214 -L 8443:127.0.0.1:443 -N
```

**Nota**: Este túnel debe permanecer activo mientras uses Nextcloud.

### 7.3 Instalación Web de Nextcloud

Acceder al instalador:

Abre tu navegador y ve a:

```
https://nextcloud.rootcode.com.bo:8443
```

Completar el formulario:

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

Hacer clic en **Finalizar configuración**.

### 7.4 Ajuste Final config.php

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

Reiniciar PHP-FPM:

```bash
sudo systemctl restart php8.3-fpm
```

#### Fase 8: Backups Automatizados

Ejecutar en VM infra como usuario `adminsrv`

### 8.1 Configurar acceso SSH sin contraseña

Generar par de llaves SSH:

```bash
ssh-keygen -t rsa -b 4096 -C "backup@infra"
```

Presionar `Enter` para aceptar la ubicación predeterminada (sin passphrase).

Copiar llave pública a VM db:

```bash
ssh-copy-id adminsrv@192.168.101.5
```

### 8.2 Configurar credenciales de base de datos

En la VM db, crear archivo de credenciales:

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

Asignar permisos:

```bash
chmod 600 /home/adminsrv/.my.cnf
```

### 8.3 Crear script de backup

En la VM infra, crear directorio de scripts:

```bash
sudo mkdir -p /opt/admin_scripts
```

Crear el script:

```bash
sudo nano /opt/admin_scripts/backup_completo.sh
```

Contenido del script:

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

Asignar permisos de ejecución:

```bash
sudo chmod +x /opt/admin_scripts/backup_completo.sh
```

Crear directorio de backups:

```bash
sudo mkdir -p /var/backups/nextcloud_full
```

### 8.4 Automatización con Cron

Editar crontab:

```bash
sudo crontab -e
```

Agregar:

```cron
0 3 * * * /opt/admin_scripts/backup_completo.sh
```

Esto ejecutará el backup todos los días a las 3:00 AM.

Verificar:

```bash
sudo crontab -l
```

### 8.5 Probar el backup manualmente

```bash
sudo /opt/admin_scripts/backup_completo.sh
```

Verificar que se creó el backup:

```bash
ls -lh /var/backups/nextcloud_full/
```

```
                         Internet
                            |
                      [Jump Server]
                    201.131.45.42:22
                            |
                 ┌──────────┴──────────┐
                 │                     │
          [Gestión: ens18]      [Proyecto: vlan101]
           192.168.100.x          192.168.101.x
                 │                     │
         ┌───────┼─────────────────────┼───────┐
         │       │                     │       │
    ┌────┴───┐ ┌─┴───┐ ┌─────┐      ┌─┴───┐ ┌─────┐
    │ Proxy  │ │ App │ │ DB  │      │Infra│ │Prome│
    │  .214  │ │ .196│ │ .178│      │.209 │ │theus│
    │  :443  │ │ :81 │ │:3306│      │ NFS │ │Grafa│
    │  SSL   │ │ PHP │ │Maria│      │ DNS │ │ na  │
    └────────┘ └─────┘ └─────┘      │Back │ └─────┘
         │         │        │        │up   │
         └─────────┴────────┴────────┴─────┘
                   VLAN 101 (Red Interna)
```

### 4.2. Estrategia Adoptada

* **Estrategia de Segmentación:** Se implementó una arquitectura de 4 capas (proxy, aplicación, datos, infraestructura) para separar responsabilidades y mejorar la seguridad mediante el principio de defensa en profundidad. El uso de VLAN 101 para comunicación interna y ens18 para gestión permite aislamiento del tráfico.

* **Estrategia de Seguridad:** Se optó por un proxy inverso Nginx en capa frontal para centralizar el cifrado SSL/TLS y ocultar la topología interna. El puerto 81 en backend evita bloqueos institucionales mientras mantiene la seguridad mediante el túnel SSH. El firewall UFW implementa listas blancas específicas por servicio.

* **Estrategia de Almacenamiento:** NFS centralizado en VM infra permite escalabilidad horizontal, facilita backups y habilita migración rápida del servicio de aplicación. El montaje automático vía fstab garantiza disponibilidad tras reinicios.

* **Estrategia de Hardening:** Certificados SSL autofirmados con CA propia, configuración de trusted domains y proxies en Nextcloud, límites de subida configurados en múltiples capas (PHP, Nginx), y acceso SSH únicamente desde red de gestión.

---

## 📋 V. Guía de Implementación y Puesta en Marcha

### 5.1. Pre-requisitos

* 4 VMs con Ubuntu 22.04 LTS y acceso root/sudo
* Conectividad de red en dos interfaces: ens18 (gestión) y vlan101 (proyecto)
* Jump Server con acceso SSH configurado (IP: 201.131.45.42)
* Espacio en disco mínimo: 20GB por VM (50GB recomendado para VM infra)
* Acceso a repositorios de paquetes Ubuntu (internet o mirror local)

### 5.2. Despliegue (Paso a Paso)

El despliegue se divide en 8 fases secuenciales que deben ejecutarse en orden:
---

## 📖 VIII. Referencias y Bibliografía

### Documentación Oficial
* [Documentación Oficial Nextcloud](https://docs.nextcloud.com/) - Guías de instalación y administración
* [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) - Configuración de proxy inverso
* [MariaDB Documentation](https://mariadb.org/documentation/) - Administración de base de datos
* [Ubuntu Server Guide](https://ubuntu.com/server/docs) - Documentación del sistema operativo base

### Recursos Técnicos Utilizados
* [NFS Server Configuration](https://ubuntu.com/server/docs/service-nfs) - Configuración de almacenamiento compartido
* [UFW - Uncomplicated Firewall](https://help.ubuntu.com/community/UFW) - Configuración de firewall
* [OpenSSL Certificate Authority](https://jamielinux.com/docs/openssl-certificate-authority/) - Creación de CA y certificados
* [SSH Tunneling Guide](https://www.ssh.com/academy/ssh/tunneling) - Túneles SSH y port forwarding

### Material de la Asignatura
* Presentaciones del curso SIS313 - Temas T1 a T6
* Laboratorios prácticos de Alta Disponibilidad y Seguridad
* Banco de Proyectos SIS313 2/2025

---

## 📄 IX. Anexos

### A. Información del Proyecto

**Institución:** Universidad Real y Pontificia de San Francisco Xavier de Chuquisaca  
**Carrera:** Ingeniería en Ciencias de la Computación / Ingeniería en Sistemas  
**Asignatura:** SIS313 - Infraestructura, Plataformas Tecnológicas y Redes  
**Docente:** Ing. Marcelo Quispe Ortega  
**Semestre:** 2/2025  
**Proyecto:** Sentinel-LAN  
**Versión:** 1.0  
**Fecha de Entrega:** Noviembre 2025  
**Repositorio:** [github.com/engel7w7/guia_proyecto_SIS313](https://github.com/engel7w7/guia_proyecto_SIS313)

---

**© 2025 - Proyecto Sentinel-LAN | SIS313 USFX**