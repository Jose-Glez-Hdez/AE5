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
