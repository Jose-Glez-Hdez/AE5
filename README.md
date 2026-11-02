# AE5 — Transferencia segura de archivos

Puesta en marcha de un servicio de transferencia de archivos seguro en contenedores Docker: 
FTP con vsftpd (usuarios, permisos, modos activo y pasivo),
FTPS explícito y SFTP/SSH con clave pública.

## 1. Entorno

| Contenedor    | Imagen         | Función                                          |
|---------------|----------------|--------------------------------------------------|
| `dpl-lab`     | `ubuntu:24.04` | Servidor: vsftpd (FTP/FTPS) y OpenSSH (SSH/SFTP) |
| `dpl-cliente` | `ubuntu:24.04` | Cliente de pruebas: lftp, ftp, curl, openssh-client |

Los dos contenedores están conectados a la red Docker `dpl-net`. La IP de `dpl-lab` es
**`172.18.0.2`**. Las contraseñas se guardan en las variables `$PASS_DESP` y `$PASS_LECT` 
para no escribirlas en la documentación.

### Creación

```bash
docker network create dpl-net
docker run -dit --name dpl-lab --hostname dpl-lab --network dpl-net ubuntu:24.04 bash
docker run -dit --name dpl-cliente --hostname dpl-cliente --network dpl-net ubuntu:24.04 bash
```

```
$ docker ps --filter name=dpl- --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
NAMES         IMAGE          STATUS
dpl-cliente   ubuntu:24.04   Up 15 seconds
dpl-lab       ubuntu:24.04   Up 15 seconds
```

### Red e IP

```
$ docker network inspect dpl-net --format '{{range .Containers}}{{.Name}} -> {{.IPv4Address}}{{"\n"}}{{end}}'
dpl-cliente -> 172.18.0.3/16
dpl-lab -> 172.18.0.2/16
```

### Sistema operativo del servidor

```
$ docker exec dpl-lab bash -c "grep -E '^(NAME|VERSION)=' /etc/os-release"
NAME="Ubuntu"
VERSION="24.04.5 LTS (Noble Numbat)"
```

## 2. Instalación

Los paquetes se instalan con `apt-get` **dentro** de cada contenedor.

```bash
docker exec dpl-lab bash -c "apt-get update && apt-get install -y vsftpd openssh-server openssl iproute2 procps"
docker exec dpl-cliente bash -c "apt-get update && apt-get install -y lftp ftp openssh-client curl"
```

| Paquete | Contenedor | Para qué |
|---|---|---|
| `vsftpd` | dpl-lab | Servidor FTP/FTPS |
| `openssh-server` | dpl-lab | Servidor SSH/SFTP |
| `openssl` | dpl-lab | Generar el certificado TLS |
| `iproute2` | dpl-lab | Comando `ss` (puertos en escucha) |
| `procps` | dpl-lab | Comando `pgrep` (procesos) |
| `lftp`, `ftp` | dpl-cliente | Clientes FTP de línea de comandos |
| `curl` | dpl-cliente | Comprobación rápida de FTPS |
| `openssh-client` | dpl-cliente | `ssh`, `scp`, `sftp`, `ssh-keygen`, `ssh-copy-id` |

### Versiones instaladas

```
$ docker exec dpl-lab bash -c "dpkg -l vsftpd | tail -1"
ii  vsftpd         3.0.5-0ubuntu3.1 arm64        lightweight, efficient FTP server written for security

$ docker exec dpl-lab bash -c "dpkg -l openssh-server | tail -1"
ii  openssh-server 1:9.6p1-3ubuntu13.20 arm64        secure shell (SSH) server, for secure access from remote machines

$ docker exec dpl-cliente bash -c "lftp --version | head -1"
LFTP | Version 4.9.2 | Copyright (c) 1996-2020 Alexander V. Lukyanov

$ docker exec dpl-cliente bash -c "curl --version | head -1"
curl 8.5.0 (aarch64-unknown-linux-gnu) libcurl/8.5.0 OpenSSL/3.0.13 zlib/1.3 brotli/1.1.0 zstd/1.5.5 libidn2/2.3.7 libpsl/0.21.2 (+libidn2/2.3.7) libssh/0.10.6/openssl/zlib nghttp2/1.59.0 librtmp/2.3 OpenLDAP/2.6.10

$ docker exec dpl-cliente ssh -V
OpenSSH_9.6p1 Ubuntu-3ubuntu13.20, OpenSSL 3.0.13 30 Jan 2024
```

## 3. Usuarios, grupo y permisos

### Usuarios y grupo

Se crea el grupo común `operaciones` y dos usuarios que pertenecen a él:
`despliegue` (escritura) y `lector` (solo lectura). Se les da shell `/bin/bash`
porque vsftpd rechaza usuarios cuya shell no aparece en `/etc/shells`.
Las contraseñas se asignan con `chpasswd` a partir de variables, para no escribirlas aquí.

```bash
docker exec dpl-lab groupadd operaciones
docker exec dpl-lab useradd -m -s /bin/bash -G operaciones despliegue
docker exec dpl-lab useradd -m -s /bin/bash -G operaciones lector
docker exec dpl-lab bash -c "echo 'despliegue:$PASS_DESP' | chpasswd"
docker exec dpl-lab bash -c "echo 'lector:$PASS_LECT' | chpasswd"
```

Resultado:

```
$ docker exec dpl-lab grep -E 'despliegue|lector' /etc/passwd
docker exec dpl-lab groups despliegue lector

despliegue:x:1001:1002::/home/despliegue:/bin/bash
lector:x:1002:1003::/home/lector:/bin/bash
despliegue : despliegue operaciones
lector : lector operaciones

$ docker exec dpl-lab groups despliegue lector
despliegue : despliegue operaciones
lector : lector operaciones
```
### Directorio de despliegue

```bash
docker exec dpl-lab mkdir -p /srv/dpl/web
docker exec dpl-lab chown despliegue:operaciones /srv/dpl/web
docker exec dpl-lab chmod 750 /srv/dpl/web
```

Resultado:
| Quién | Permiso | Efecto |
|---|---|---|
| `despliegue` (propietario) | `rwx` (7) | Lista, entra y **escribe** |
| `operaciones` (grupo, incluye a `lector`) | `r-x` (5) | Lista y entra, **no escribe** |
| Resto de usuarios | `---` (0) | Sin acceso |

```
$ docker exec dpl-lab ls -ld /srv/dpl/web
drwxr-x--- 2 despliegue operaciones 4096 Nov  2 15:15 /srv/dpl/web
```

## 4. Configuración de vsftpd

### Certificado TLS

Para FTPS vsftpd necesita un certificado y su clave privada. Se genera un certificado **autofirmado** válido un año; por eso los clientes usan
`ssl:verify-certificate no` y `--insecure`.

```bash
docker exec dpl-lab openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/vsftpd.key -out /etc/ssl/certs/vsftpd.pem -subj "/CN=dpl-lab"
```

Resultado:

```
$ docker exec dpl-lab ls -l /etc/ssl/certs/vsftpd.pem /etc/ssl/private/vsftpd.key
-rw-r--r-- 1 root root 1107 Nov  2 15:20 /etc/ssl/certs/vsftpd.pem
-rw------- 1 root root 1704 Nov  2 15:20 /etc/ssl/private/vsftpd.key

$ docker exec dpl-lab openssl x509 -in /etc/ssl/certs/vsftpd.pem -noout -subject -dates
subject=CN = dpl-lab
notBefore=Nov  2 15:20:08 2026 GMT
notAfter=Nov  2 15:20:08 2027 GMT
```

La clave privada queda con permisos `600` (solo root).

### Directivas FTP

La configuración se escribe en `vsftpd.conf`, se copia al servidor y se
asigna a root:

```bash
docker cp vsftpd.conf dpl-lab:/etc/vsftpd.conf
docker exec dpl-lab chown root:root /etc/vsftpd.conf
```

| Directiva | Valor | Por qué |
|---|---|---|
| `listen` / `listen_ipv6` | `YES` / `NO` | Servidor independiente escuchando en IPv4 |
| `anonymous_enable` | `NO` | Nadie entra sin usuario y contraseña |
| `local_enable` | `YES` | Entran los usuarios del sistema (`despliegue`, `lector`) |
| `write_enable` | `YES` | Permite subir; quién puede escribir lo deciden los permisos (750) |
| `local_umask` | `022` | Lo subido queda 644/755: legible por el grupo |
| `port_enable` / `connect_from_port_20` | `YES` | Modo activo: el servidor abre datos desde el puerto 20 |
| `pasv_enable` / `pasv_min_port` / `pasv_max_port` | `YES` / `40000` / `40100` | Modo pasivo con rango acotado  |

Configuración activa:

```
$ docker exec dpl-lab bash -c "grep -v '^#' /etc/vsftpd.conf | grep -v '^$'"
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
pam_service_name=vsftpd
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
secure_chroot_dir=/var/run/vsftpd/empty
port_enable=YES
connect_from_port_20=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
```
### Directivas FTPS

| Directiva | Valor | Por qué |
|---|---|---|
| `ssl_enable` | `YES` | Activa TLS |
| `rsa_cert_file` / `rsa_private_key_file` | `/etc/ssl/certs/vsftpd.pem`/ `/etc/ssl/private/vsftpd.key` | Certificado y clave del servidor |
| `force_local_logins_ssl` | `YES` | La contraseña nunca viaja sin cifrar |
| `force_local_data_ssl` | `YES` | Los ficheros viajan cifrados |
| `ssl_sslv2` / `ssl_sslv3` | `NO` | Protocolos antiguos e inseguros desactivados |
| `ssl_ciphers` | `HIGH` | Solo cifrados fuertes |
| `require_ssl_reuse` | `NO` | Compatibilidad con lftp en el canal de datos |

Resultado:

```
$ docker exec dpl-lab grep -E 'anonymous_enable|local_enable|write_enable|ssl_enable' /etc/vsftpd.conf
anonymous_enable=NO
local_enable=YES
write_enable=YES
ssl_enable=YES
```

La configuración completa está en `vsftpd.conf`.

## 5. Arranque de vsftpd sin systemd

En el contenedor no existe systemd, así que no se usa `systemctl`. vsftpd se lanza en
primer plano en segundo término de Docker (`docker exec -d`).

```bash
docker exec dpl-lab mkdir -p /var/run/vsftpd/empty
docker exec -d dpl-lab /usr/sbin/vsftpd /etc/vsftpd.conf
```

Comprobación por proceso y puerto:

```
$ docker exec dpl-lab pgrep -a vsftpd
4194 /usr/sbin/vsftpd /etc/vsftpd.conf

$ docker exec dpl-lab ss -lntp | grep ':21'
LISTEN 0      32           0.0.0.0:21         0.0.0.0:*    users:(("vsftpd",pid=4194,fd=3))
```

## 6. Pruebas FTP: subida, descarga y denegaciones

El servidor exige TLS, así que lftp negocia el cifrado automáticamente; con
`set ssl:verify-certificate no` acepta el certificado autofirmado.

### Subida con `despliegue` — correcta

```
$ docker exec dpl-cliente bash -c "echo 'prueba de transferencia' > /tmp/prueba.txt"
$ docker exec dpl-cliente lftp -d -u despliegue,$PASS_DESP $IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"
---- Closing data socket
<--- 226 Transfer complete.
---> SITE UTIME 20261102171326 prueba.txt
<--- 500 Unknown SITE command.
---> SITE UTIME prueba.txt 20261102171326 20261102171326 20261102171326 UTC
<--- 500 Unknown SITE command.
---> QUIT
<--- 221 Goodbye.
---- Closing control socket
josegleez@MacBook-Air-de-Jose-2 AE5 % docker exec dpl-lab ls -l /srv/dpl/web
total 4
-rw-r--r-- 1 despliegue despliegue 24 Nov  2 17:21 prueba.txt
josegleez@MacBook-Air-de-Jose-2 AE5 % docker exec dpl-cliente bash -c "echo 'prueba de transferencia' > /tmp/prueba.txt"
docker exec dpl-cliente lftp -u despliegue,$PASS_DESP $IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"
josegleez@MacBook-Air-de-Jose-2 AE5 % docker exec dpl-cliente bash -c "echo 'prueba de transferencia' > /tmp/prueba.txt"
josegleez@MacBook-Air-de-Jose-2 AE5 % docker exec dpl-cliente lftp -d -u despliegue,$PASS_DESP $IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 331 Please specify the password.
---> PASS ********
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> TYPE I
<--- 200 Switching to Binary mode.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,154).
---- Connecting data socket to (172.18.0.2) port 40090
---- Data connection established
---> STOR prueba.txt
<--- 150 Ok to send data.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Closing data socket
<--- 226 Transfer complete.
---> SITE UTIME 20261102172453 prueba.txt
<--- 500 Unknown SITE command.
---> SITE UTIME prueba.txt 20261102172453 20261102172453 20261102172453 UTC
<--- 500 Unknown SITE command.
---> QUIT
<--- 221 Goodbye.
---- Closing control socket

$ docker exec dpl-lab ls -l /srv/dpl/web
total 4
-rw-r--r-- 1 despliegue despliegue 24 Nov  2 17:24 prueba.txt
```

### Descarga con `lector` — correcta

```
$ docker exec dpl-cliente lftp -d -u lector,$PASS_LECT $IP_DPL_LAB -e "set ssl:verify-certificate no; cd /srv/dpl/web; get prueba.txt -o /tmp/descargado.txt; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER lector
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 331 Please specify the password.
---> PASS ********
<--- 230 Login successful.
---> PWD
<--- 257 "/home/lector" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> TYPE I
<--- 200 Switching to Binary mode.
---> SIZE prueba.txt
<--- 213 24
---> MDTM prueba.txt
<--- 213 20261102172458
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,68).
---- Connecting data socket to (172.18.0.2) port 40004
---- Data connection established
---> RETR prueba.txt
<--- 150 Opening BINARY mode data connection for prueba.txt (24 bytes).
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Got EOF on data connection
---- Closing data socket
get: prueba.txt: /tmp/descargado.txt: file already exists and xfer:clobber is unset
<--- 226 Transfer complete.
---> QUIT
gnutls_record_recv: The TLS connection was non-properly terminated. Assuming EOF.
<--- 221 Goodbye.
---- Closing control socket

$ docker exec dpl-cliente cat /tmp/descargado.txt
prueba de transferencia
```

### Subida con `lector` — denegada

```
$ docker exec dpl-cliente lftp -d -u lector,$PASS_LECT $IP_DPL_LAB -e "set net:max-retries 1; set net:reconnect-interval-base 1; set ssl:verify-certificate no; cd /srv/dpl/web; put /tmp/prueba.txt; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER lector
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 331 Please specify the password.
---> PASS ********
<--- 230 Login successful.
---> PWD
<--- 257 "/home/lector" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---- CWD path to be sent is `/srv/dpl/web'
---> CWD /srv/dpl/web
<--- 250 Directory successfully changed.
---> TYPE I
<--- 200 Switching to Binary mode.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,75).
---- Connecting data socket to (172.18.0.2) port 40011
---- Data connection established
---> STOR prueba.txt
<--- 553 Could not create file.
put: /tmp/prueba.txt: Access failed: 553 Could not create file. (prueba.txt)
---- Closing data socket
---> QUIT
<--- 221 Goodbye.
---- Closing control socket
```

`lector` pertenece al grupo `operaciones`, que solo tiene `r-x` sobre `/srv/dpl/web`:
el servidor responde **553** y no se crea el fichero.

### Acceso anónimo — denegado

```
$ docker exec dpl-cliente lftp -d -u anonymous, $IP_DPL_LAB -e "set net:max-retries 1; set net:reconnect-interval-base 1; ls; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER anonymous
ERROR: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
**** Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Closing control socket
ls: Fatal error: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)

$ docker exec dpl-cliente lftp -d -u anonymous, $IP_DPL_LAB -e "set net:max-retries 1; set net:reconnect-interval-base 1; set ssl:verify-certificate no; ls; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER anonymous
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 530 Anonymous sessions may not use encryption.
---> QUIT
ls: Login failed: 530 Anonymous sessions may not use encryption.
<--- 221 Goodbye.
---- Closing control socket
```

Con `anonymous_enable=NO` el servidor responde **530 Login incorrect**. En las pruebas que deben fallar se añade `set net:max-retries 1; set net:reconnect-interval-base 1`
para que lftp muestre el error y termine, en lugar de reintentar indefinidamente en silencio.

**Conclusión:** los permisos 750 y la desactivación del anónimo funcionan como se esperaba.

## 7. Modo activo y modo pasivo

### Activo

```
$ docker exec dpl-cliente lftp -d -e "debug 3; set ftp:passive-mode off; ..."
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 230 Login successful.
<--- 250 Directory successfully changed.
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
-rw-r--r--    1 1001     1002           24 Nov 02 17:24 prueba.txt
```

### Pasivo

```
$ docker exec dpl-cliente lftp -d -e "debug 3; set ftp:passive-mode on; ..."
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 230 Login successful.
<--- 250 Directory successfully changed.
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
-rw-r--r--    1 1001     1002           24 Nov 02 17:24 prueba.txt
```

### Quién abre el canal de datos

| Modo | Orden del cliente | Quién abre la conexión de datos | Puertos |
|---|---|---|---|
| Activo | `PORT` / `EPRT` (indica IP y puerto del cliente) | El **servidor**, hacia el cliente | origen 20 del servidor → puerto del cliente |
| Pasivo | `PASV` / `EPSV` (el servidor responde un puerto) | El **cliente**, hacia el servidor | cliente → 40000-40100 del servidor |

## 8. Despliegue del build de Vite

Se genera el build de producción de la aplicación de la UT4 y se sube completo a
`/srv/dpl/web/dist` con `mirror -R`.

```bash
docker cp dist dpl-cliente:/tmp/dist
docker exec dpl-cliente lftp -u despliegue,$PASS_DESP $IP_DPL_LAB \
  -e "set ssl:verify-certificate no; mirror -R --verbose /tmp/dist /srv/dpl/web/dist; exit"
```

Resultado de la subida:

```
$ docker exec dpl-cliente lftp -u despliegue,$PASS_DESP $IP_DPL_LAB -d -e "set ssl:verify-certificate no; mirror -R --verbose /tmp/dist /srv/dpl/web/dist; exit"
---- Resolving host address...
---- IPv6 is not supported or configured
---- 1 address found: 172.18.0.2
---- Connecting to 172.18.0.2 (172.18.0.2) port 21
<--- 220 (vsFTPd 3.0.5)
---> FEAT
<--- 211-Features:
<---  AUTH TLS
<---  EPRT
<---  EPSV
<---  MDTM
<---  PASV
<---  PBSZ
<---  PROT
<---  REST STREAM
<---  SIZE
<---  TVFS
<--- 211 End
---> AUTH TLS
<--- 234 Proceed with negotiation.
Loaded 120 CAs
Loaded 0 CRLs
---> USER despliegue
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
<--- 331 Please specify the password.
---> PASS ********
<--- 230 Login successful.
---> PWD
<--- 257 "/home/despliegue" is the current directory
---> PBSZ 0
<--- 200 PBSZ set to 0.
---> PROT P
<--- 200 PROT now Private.
---> MKD /srv
<--- 550 Create directory operation failed.
---> MKD /srv/dpl
<--- 550 Create directory operation failed.
---> MKD /srv/dpl/web
<--- 550 Create directory operation failed.
---> MKD /srv/dpl/web/dist
<--- 257 "/srv/dpl/web/dist" created
---- CWD path to be sent is `/srv/dpl/web/dist'
---> CWD /srv/dpl/web/dist
<--- 250 Directory successfully changed.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,77).
---- Connecting data socket to (172.18.0.2) port 40013
---- Data connection established
---> LIST
<--- 150 Here comes the directory listing.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,119).
---- Connecting data socket to (172.18.0.2) port 40055
---- Data connection established
---> NLST
<--- 150 Here comes the directory listing.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> TYPE I
<--- 200 Switching to Binary mode.
Transferring file `index.html'
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,145).
---- Connecting data socket to (172.18.0.2) port 40081
---- Data connection established
---> STOR index.html
<--- 150 Ok to send data.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Closing data socket
<--- 226 Transfer complete.
---> SITE UTIME 20260907221513 index.html
<--- 500 Unknown SITE command.
---> SITE UTIME index.html 20260907221513 20260907221513 20260907221513 UTC
<--- 500 Unknown SITE command.
---> MKD assets
Making directory `assets'
<--- 257 "/srv/dpl/web/dist/assets" created
---- CWD path to be sent is `/srv/dpl/web/dist/assets'
---> CWD /srv/dpl/web/dist/assets
<--- 250 Directory successfully changed.
---> TYPE A
<--- 200 Switching to ASCII mode.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,143).
---- Connecting data socket to (172.18.0.2) port 40079
---- Data connection established
---> LIST
<--- 150 Here comes the directory listing.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,117).
---- Connecting data socket to (172.18.0.2) port 40053
---- Data connection established
---> NLST
<--- 150 Here comes the directory listing.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Got EOF on data connection
---- Closing data socket
<--- 226 Directory send OK.
---> TYPE I
Transferring file `assets/dpl-app.js'
<--- 200 Switching to Binary mode.
---> PASV
<--- 227 Entering Passive Mode (172,18,0,2,156,83).
---- Connecting data socket to (172.18.0.2) port 40019
---- Data connection established
---> STOR dpl-app.js
<--- 150 Ok to send data.
Loaded 120 CAs
Loaded 0 CRLs
WARNING: Certificate verification: The certificate is NOT trusted. The certificate issuer is unknown.  (98:A3:E3:32:82:8F:57:79:AB:FA:1F:B5:65:A5:33:F4:F9:64:6C:F1)
---- Closing data socket
<--- 226 Transfer complete.
---> SITE CHMOD 644 ./dpl-app.js
<--- 200 SITE CHMOD command ok.
---- CWD path to be sent is `/srv/dpl/web/dist'
---> CWD /srv/dpl/web/dist
<--- 250 Directory successfully changed.
---> SITE CHMOD 644 ./index.html
<--- 200 SITE CHMOD command ok.
---> SITE CHMOD 755 ./assets
<--- 200 SITE CHMOD command ok.
---> QUIT
<--- 221 Goodbye.
---- Closing control socket
```

Verificación en el servidor:

```
$ docker exec dpl-lab ls -lR /srv/dpl/web/dist
/srv/dpl/web/dist:
total 8
drwxr-xr-x 2 despliegue despliegue 4096 Nov  2 18:11 assets
-rw-r--r-- 1 despliegue despliegue  735 Nov  2 18:11 index.html
```

Con `local_umask=022` los ficheros quedan `644` y los directorios `755`, así que
`lector` puede leerlos, pero no modificarlos.

## 9. FTPS explícito y comprobación con curl

FTPS **explícito**: el cliente conecta al puerto 21 y pide cifrado con `AUTH TLS`.

### lftp con TLS obligatorio

```
$ docker exec dpl-cliente lftp -u despliegue,$PASS_DESP $IP_DPL_LAB -e "set ftp:ssl-force true; set ssl:verify-certificate no; cd /srv/dpl/web; ls; exit"
drwxr-xr-x    3 1001     1002         4096 Nov 02 18:11 dist
-rw-r--r--    1 1001     1002           24 Nov 02 17:24 prueba.txt
```

### curl: banner y listado

```
$ docker exec dpl-cliente bash -c "curl -sv --ssl-reqd --insecure ... | grep 220"
< 220 (vsFTPd 3.0.5)
$ docker exec dpl-cliente curl -s --ssl-reqd --insecure -u despliegue:$PASS_DESP ftp://$IP_DPL_LAB/
```

### FTP sin cifrar rechazado

```
docker exec dpl-cliente bash -c "curl -sv -u despliegue:$PASS_DESP ftp://$IP_DPL_LAB/ 2>&1 | grep -E '^< 5'"
< 530 Non-anonymous sessions must use encryption.
```

- `ftp:ssl-force true` / `--ssl-reqd`: obligan al cliente a usar TLS.
- `ssl:verify-certificate no` / `--insecure`: necesarios porque el certificado es autofirmado.
- El FTP en claro se rechaza por `force_local_logins_ssl=YES`.

## 10. SSH/SFTP con clave pública

### Arranque de sshd

`/run/sshd` es necesario para la separación de privilegios de sshd y no existe en el
contenedor; `ssh-keygen -A` genera las claves de host que falten.

```bash
docker exec dpl-lab mkdir -p /run/sshd
docker exec dpl-lab ssh-keygen -A
docker exec -d dpl-lab /usr/sbin/sshd -D
```

```
$ docker exec dpl-lab pgrep -a sshd
4408 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups

$ docker exec dpl-lab ss -lntp | grep ':22'
LISTEN 0      128          0.0.0.0:22         0.0.0.0:*    users:(("sshd",pid=4408,fd=3))  
LISTEN 0      128             [::]:22            [::]:*    users:(("sshd",pid=4408,fd=4))  
```

### Clave pública ed25519

Se genera un par de claves **ed25519** en elcliente y se instala la pública en `despliegue` 
con `ssh-copy-id`. Es la única vez que se usa la contraseña por SSH. La clave no lleva frase 
de paso para poder automatizar despliegues; en producción se protegería con frase de paso y `ssh-agent`.

```
$ docker exec dpl-cliente ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
docker exec dpl-cliente ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
Generating public/private ed25519 key pair.
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:sD62eTp5HO/eM9FR+lLC26RD9CkV9s93YmcQeBSVmbk root@dpl-cliente
The key's randomart image is:
+--[ED25519 256]--+
|             o+=B|
|            . +Bo|
|      .      +o++|
|       o     .*E=|
|      . S    o+XB|
|     .  .   ..*+=|
|      +o o   . o |
|     .o++ ..o    |
|      += oo .o   |
+----[SHA256]-----+

$ docker exec -it dpl-cliente ssh-copy-id despliegue@$IP_DPL_LAB
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
despliegue@172.18.0.2's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'despliegue@172.18.0.2'"
and check to make sure that only the key(s) you wanted were added.


What's next:
    Try Docker Debug for seamless, persistent debugging tools in any container or image → docker debug dpl-cliente
    Learn more at https://docs.docker.com/go/debug-cli/

$ docker exec dpl-lab ls -l /home/despliegue/.ssh/authorized_keys
-rw------- 1 despliegue despliegue 98 Nov  2 18:32 /home/despliegue/.ssh/authorized_keys
```
### Endurecimiento de sshd

```bash
docker exec dpl-lab sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
docker exec dpl-lab sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
docker exec dpl-lab sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
docker exec dpl-lab sed -i 's/^#\?KbdInteractiveAuthentication.*/KbdInteractiveAuthentication no/' /etc/ssh/sshd_config
docker exec dpl-lab sshd -t
docker exec dpl-lab pkill sshd
docker exec -d dpl-lab /usr/sbin/sshd -D
```

| Directiva | Valor | Por qué |
|---|---|---|
| `PasswordAuthentication` | `no` | Elimina los ataques de fuerza bruta contra contraseñas |
| `KbdInteractiveAuthentication` | `no` | Cierra la otra vía para introducir contraseña |
| `PubkeyAuthentication` | `yes` | Solo se entra con clave |
| `PermitRootLogin` | `no` | Nadie entra directamente como administrador |

Configuración efectiva:

```
$ docker exec dpl-lab grep -E '^PasswordAuthentication|^PermitRootLogin|^PubkeyAuthentication|^KbdInteractiveAuthentication' /etc/ssh/sshd_config
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no

$ docker exec dpl-lab bash -c "sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication) '"
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
```















## 13. Problemas encontrados y solución

### Problema 1 — lftp sin respuesta en las pruebas FTP

- **Síntoma:** primero `cd: Not connected` / `put: /tmp/prueba.txt: Not connected`; después,
  las pruebas de `lector` y `anonymous` se quedaban más de 10 minutos sin mostrar ningún error.
- **Causa:**
  - `Not connected`: al escribir la IP a mano se usó `172.18.0.3`, que es la IP de `dpl-cliente`,
     no la de `dpl-lab` (`172.18.0.2`). lftp intentaba conectar con el propio cliente, donde
     no hay servidor FTP, y por defecto reintenta hasta 1000 veces con esperas crecientes
     sin mostrar nada en modo no interactivo.
- **Solución:**
  - Obtener la IP del servidor con `docker inspect` y usar variables de sesión:
    `export IP_DPL_LAB=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' dpl-lab)`
  - Detener los procesos colgados: `docker exec dpl-cliente pkill lftp`.
  - En las pruebas que deben fallar, limitar los reintentos:
    `set net:max-retries 1; set net:reconnect-interval-base 1`.
