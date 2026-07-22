# 00 — Infraestructura base

Documenta el sustrato sobre el que corre el laboratorio: hipervisor, contenedor de
ejecución y herramientas. Es el paso previo a levantar el stack OT/IT.

---

## 1. Hardware

| Elemento | Especificación |
|---|---|
| Equipo | Intel NUC (x2, idénticos) |
| RAM | 8 GB por nodo |
| Almacenamiento | 512 GB NVMe |

Los dos nodos son deliberados, no un exceso: en la fase de segmentación permiten
separar **físicamente** la zona OT de la zona IT, con la red entre ambos actuando
como conducto controlado. Esto se acerca mucho más al modelo de zonas y conductos
de IEC 62443 que dos redes virtuales dentro de una misma máquina.

En esta primera etapa solo se usa el nodo 1.

---

## 2. Hipervisor

| Componente | Versión |
|---|---|
| Proxmox VE | 9.2.2 |
| Kernel | 7.0.2-6-pve |
| Base | Debian (instalación manual + Proxmox encima) |
| Hostname del nodo | `mylab` |

### Almacenamiento: LVM-thin, no ZFS

Con 8 GB de RAM, ZFS queda descartado. Su caché de lectura (ARC) reserva por
defecto una porción significativa de la memoria del sistema, lo que en un nodo de
8 GB deja sin margen a las cargas de trabajo. LVM-thin ofrece igualmente
aprovisionamiento fino y snapshots con un consumo de memoria despreciable.

Almacenamientos configurados:

| Nombre | Tipo | Uso |
|---|---|---|
| `local` | dir | Plantillas de contenedor, ISOs, backups |
| `local-lvm` | lvmthin | Discos de contenedores y máquinas virtuales (~349 GiB) |

> `local-lvm` no aparece en la salida de `df -h`. Es el comportamiento esperado:
> un pool thin de LVM no se monta como sistema de ficheros. Para consultarlo se
> usa `pvesm status`.

---

## 3. Contenedor de ejecución: `edge-01`

### LXC en lugar de máquina virtual

Un contenedor LXC comparte kernel con el host y arranca con una fracción de la
memoria que necesitaría una VM equivalente. Con 8 GB disponibles, esta decisión es
la diferencia entre poder levantar dos "máquinas" o cuatro. Dado que todas las
cargas del laboratorio son Linux, no se pierde nada por no virtualizar el kernel.

La excepción prevista es un eventual firewall basado en BSD (OPNsense/pfSense),
que exigiría una VM completa. Se evaluará en la fase de segmentación frente a la
alternativa de `nftables` sobre un LXC Debian.

### No privilegiado

El contenedor se crea **sin privilegios**: los UID internos se mapean a un rango
sin permisos en el host, de modo que un escape del contenedor no otorga root sobre
Proxmox. Siendo este un laboratorio de seguridad, es la opción defendible por
defecto.

Ejecutar Docker en un LXC no privilegiado exige dos habilitadores:

- **`nesting=1`** — permite anidar contenedores dentro del contenedor.
- **`keyctl=1`** — expone las llamadas al keyring del kernel que el runtime de
  Docker necesita. Sin esto, los fallos son tardíos y poco descriptivos.

### Parámetros

| Parámetro | Valor |
|---|---|
| CTID | `100` |
| Hostname | `edge-01` |
| Plantilla | `debian-13-standard_13.6-1_amd64.tar.zst` |
| CPU | 2 cores |
| RAM | 2048 MB |
| Swap | 512 MB |
| Disco | 16 GB en `local-lvm` |
| Red | `vmbr0`, IP estática |
| Arranque automático | Sí |

> **Ajustar antes de reproducir:** la IP del contenedor, el gateway y el servidor
> DNS dependen de la red local. La IP debe estar fuera del rango DHCP del router.
> Se usa IP fija porque el editor de Node-RED se consulta constantemente por
> `http://<IP>:1880`; una IP variable rompe el flujo de trabajo.

### Creación

```bash
pveam update
pveam available --section system | grep -i debian
pveam download local debian-13-standard_13.6-1_amd64.tar.zst

pct create 100 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname edge-01 \
  --unprivileged 1 \
  --features nesting=1,keyctl=1 \
  --cores 2 \
  --memory 2048 \
  --swap 512 \
  --rootfs local-lvm:16 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.50/24,gw=192.168.1.1 \
  --nameserver 1.1.1.1 \
  --onboot 1 \
  --password

pct start 100
pct enter 100
```

---

## 4. Docker

Se instala desde el repositorio oficial de Docker, no desde los paquetes de
Debian, que van por detrás en versiones.

```bash
apt install -y ca-certificates curl gnupg git

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verificación

```bash
systemctl status docker --no-pager
docker run --rm hello-world
```

Versión instalada en el momento de escribir este documento: **Docker 29.6.2**.

---

## 5. Git

```bash
git config --global user.name "<nombre>"
git config --global user.email "<email de GitHub>"
git config --global init.defaultBranch main
git config --global pull.rebase false

ssh-keygen -t ed25519 -C "edge-01-lab"
cat ~/.ssh/id_ed25519.pub   # registrar en GitHub → Settings → SSH and GPG keys
ssh -T git@github.com       # verificación
```

El comentario `-C "edge-01-lab"` identifica la clave en GitHub. Con varias
máquinas registradas, permite revocar únicamente la que corresponda.

Git se instala **dentro de `edge-01`**, no en el host ni en el equipo de
escritorio: es donde residen los ficheros de configuración del stack, de modo que
se edita y se commitea en el mismo sitio sin sincronizaciones intermedias.

---

## 6. Incidencias encontradas

Se documentan porque el diagnóstico es más informativo que el síntoma.

### `ipcc_send_rec[N] failed: Unknown error -1`

Aparece al ejecutar cualquier comando `pve*` **sin ser root**. El sistema de
ficheros de configuración de Proxmox (`/etc/pve`, montado sobre FUSE) solo es
accesible para root y `www-data`; el cliente no logra comunicarse con el demonio
`pmxcfs` y devuelve este mensaje, que oculta lo que en realidad es un permiso
denegado.

Solución: `su -` (en instalaciones manuales de Debian, `sudo` puede no estar
presente).

### `REMOTE HOST IDENTIFICATION HAS CHANGED`

Tras reinstalar el host, SSH genera claves nuevas y el cliente conserva las
anteriores. El aviso es legítimo: un cambio de clave es indistinguible de un
ataque de interposición.

El procedimiento correcto es **verificar antes de borrar**. Desde la consola del
servidor:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Y comparar la huella con la mostrada en el aviso. Solo si coinciden se elimina la
entrada antigua en el cliente:

```bash
ssh-keygen -R <ip-del-host>
```

Borrar la entrada sin comparar es precisamente el hábito que un ataque de
interposición explota.

### Mensajes de `nftables` al arrancar Docker

Líneas del tipo `Deleting nftables IPv4 rules ... status 1` durante el arranque
del demonio. Son informativas: Docker intenta limpiar reglas de una ejecución
previa y no encuentra ninguna. Aparecen con más frecuencia dentro de un LXC. Si a
continuación se lee `Loading containers: done` y `Daemon has completed
initialization`, el arranque es correcto.

---

## 7. Estado

- [x] Proxmox operativo y accesible
- [x] LXC `edge-01` creado y con red
- [x] Docker funcionando en LXC no privilegiado
- [x] Git y clave SSH configurados
- [ ] Stack OT/IT desplegado — ver `docs/fase-1-pipeline-base.md`
