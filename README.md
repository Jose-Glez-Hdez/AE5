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