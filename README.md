# Puesta en marcha

Instrucciones para desplegar el laboratorio en un entorno nuevo.

---

## 1. Requisitos previos

### Infraestructura

- **Hipervisor:** Proxmox VE (o similar con capacidad para LXC/contenedores)
- **Contenedor Linux:** Debian 13 con Docker 29+ instalado
- **Red:** acceso a internet para descargar imágenes Docker
- **Recurso:** 2 cores, 2 GB RAM, 16 GB almacenamiento (mínimo)

El laboratorio está diseñado para correr en un **LXC no privilegiado con nesting
activado**. Si usas una VM Linux completa o Docker Desktop, el comando `docker
compose up -d` seguirá funcionando, pero la configuración de aislamiento será
distinta.

### En tu máquina de desarrollo

- `git` instalado y configurado (`user.name` y `user.email`)
- Clave SSH registrada en GitHub (recomendado) o token PAT para HTTPS
- Navegador web para acceder a Node-RED en `http://<IP-del-contenedor>:1880`

---

## 2. Clonar el repositorio

```bash
cd /opt
git clone git@github.com:<tu-usuario>/project-cerberus.git
cd project-cerberus
git checkout fase-1
```

Sustituye `<tu-usuario>` por tu nombre de usuario en GitHub. Si usas HTTPS en lugar de SSH:

```bash
git clone https://github.com/<tu-usuario>/project-cerberus.git
```

---

## 3. Preparar permisos

Los ficheros del repositorio se descargan con propietario `root`. Node-RED corre
como UID `1000:1000` (usuario sin privilegios dentro del contenedor). Sin este
paso, el contenedor fallará con permisos al crear `node_modules/`:

```bash
chown -R 1000:1000 node-red/
```

---

## 4. Levantar el stack

```bash
docker compose up -d
docker compose ps
```

Los tres servicios deben estar en estado `Up`:

```
CONTAINER ID   IMAGE                       STATUS           PORTS
...            oitc/modbus-server:latest   Up 1 minute      0.0.0.0:5020->5020/tcp
...            eclipse-mosquitto:2         Up 1 minute      0.0.0.0:1883->1883/tcp
...            nodered/node-red:latest     Up 1 minute      0.0.0.0:1880->1880/tcp
```

Si Node-RED está en `Restarting`, ver [troubleshooting](#troubleshooting).

---

## 5. Instalar dependencias de Node-RED

Node-RED necesita descargar `node-red-contrib-modbus` (declarado en
`node-red/package.json`). El contenedor lo hace automáticamente en el primer
arranque, pero es más predecible hacerlo explícitamente:

```bash
docker exec -it -w /data node-red npm install
```

Espera a que termine. Cuando veas algo como:

```
added 55 packages, and audited 56 packages in 3s
```

El proceso ha terminado correctamente.

---

## 6. Verificar que funciona

### 6.1 Acceso al editor de Node-RED

Abre en el navegador: `http://<IP-del-contenedor>:1880`

Deberías ver el lienzo con tres nodos encadenados:
- **`Reading 1st line`** (naranja) — Modbus Read
- **`Scale and naming`** (beige) — function
- Un nodo sin visible (ya que no hay debug en el flujo actual)

Si los nodos aparecen en gris con errores, Node-RED no terminó de instalar las
dependencias. Vuelve al paso anterior.

### 6.2 Datos fluyendo por MQTT

Desde una terminal del contenedor, suscríbete al broker:

```bash
docker exec -it mosquitto mosquitto_sub -t 'planta/#' -v
```

Deberías ver aparecer una línea cada 2 segundos con el JSON de telemetría:

```
planta/linea1/telemetria {"timestamp":"2026-07-22T12:45:00.123Z","temperatura_c":72.5,"presion_bar":15.2,"rpm":1450,"estado":1,"piezas":12345}
planta/linea1/telemetria {"timestamp":"2026-07-22T12:45:02.124Z","temperatura_c":72.5,"presion_bar":15.2,"rpm":1450,"estado":1,"piezas":12345}
```

Si no ves nada después de 5 segundos, el flujo de Node-RED no está publicando.
Verifica los logs: `docker logs node-red`.

### 6.3 Lectura directa del PLC

Para confirmar que el simulador responde a Modbus:

```bash
docker run --rm --network host oitc/modbus-client \
  -s 127.0.0.1 -p 5020 -t 3 -r 0 -l 5
```

Esperado:

```
register
40000     0x02D5     725    725  ...  72.5°C (temperatura ÷10)
40001     0x0098     152    152  ...  15.2 bar (presión ÷10)
40002     0x05AA    1450   1450  ...  RPM motor
40003     0x0001       1      1  ...  Estado (1=marcha)
40004     0x3039   12345  12345  ...  Contador de piezas
```

---

## 7. Parar y reanudar

Para detener sin eliminar:

```bash
docker compose stop
```

Para reanudar:

```bash
docker compose start
```

Para parar y eliminar contenedores (pero mantiene volúmenes y configuraciones):

```bash
docker compose down
```

Para una limpieza completa (elimina también volúmenes):

```bash
docker compose down -v
```

Después de un `down`, necesitarás volver a hacer `npm install`:

```bash
docker compose up -d
docker exec -it -w /data node-red npm install
docker restart node-red
```

---

## 8. Troubleshooting

### Node-RED en `Restarting` infinito

**Síntoma:** `docker compose ps` muestra `node-red` en estado `Restarting`.

**Causas posibles:**

1. **Permisos en `node-red/`:** El directorio pertenece a root.
   ```bash
   chown -R 1000:1000 node-red/
   docker compose restart node-red
   ```

2. **Dependencias no instaladas:** `node-red/node_modules` está vacío.
   ```bash
   docker exec -it -w /data node-red npm install
   docker restart node-red
   ```

3. **Puerto ya en uso:** Algo más escucha en 1880.
   ```bash
   lsof -i :1880
   ```

**Verificar logs detallados:**

```bash
docker logs -f node-red
```

Busca líneas con `[error]` o `Failed to start`.

---

### Modbus no responde

**Síntoma:** El cliente Modbus devuelve conexión rechazada.

**Causas posibles:**

1. **PLC no está corriendo:**
   ```bash
   docker compose ps modbus-plc
   ```

2. **Puerto bloqueado o mal configurado:** Verifica que el puerto 5020 esté
   publicado:
   ```bash
   docker port modbus-plc
   ```

3. **Configuración JSON inválida:** Revisa `modbus/server_config.json`.

**Diagnosticar:**

```bash
docker logs modbus-plc
```

---

### MQTT no recibe mensajes

**Síntoma:** `mosquitto_sub` se queda esperando sin recibir nada.

**Causas posibles:**

1. **Node-RED no está publicando:** Verifica que el nodo `mqtt out` esté
   conectado.
   ```bash
   docker logs node-red | grep -i mqtt
   ```

2. **Topic incorrecto:** El flujo publica en `planta/linea1/telemetria`. Si
   suscribiste a otro topic, no verás nada.

3. **Red de Docker:** Los servicios no se ven entre sí.
   ```bash
   docker network inspect project-cerberus_ot
   ```

**Diagnosticar:**

Verifica que Mosquitto está escuchando:

```bash
docker logs mosquitto
```

---

## 9. Próximas fases

Este estado corresponde a **Fase 1: Tubería base**. Ver
[docs/fase-1-pipeline-base.md](docs/fase-1-pipeline-base.md) para detalles
técnicos.

Las siguientes fases están documentadas en [`docs/`](docs/):

- **Fase 2:** Valores dinámicos, InfluxDB y Grafana
- **Fase 3:** Segmentación de red, autenticación y cifrado
