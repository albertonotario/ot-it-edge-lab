# Fase 2 — Persistencia: InfluxDB + Telegraf

Segundo incremento. Añade almacenamiento sin tocar la Fase 1, demostrando lo que
significa realmente desacoplamiento en pub/sub.

**Estado:** completada · **Tag:** `fase-2`

---

## 1. Qué se añade

Dos contenedores nuevos:

- **Telegraf** — agente de ingesta. Se suscribe a `planta/#`, recibe los JSONs de
  Node-RED, los parsea y escribe en InfluxDB. Es un puente.
- **InfluxDB** — base de datos de series temporales. Almacena las métricas con
  timestamp, permitiendo después consultarlas, analizarlas, graficarlas.

El resto de la Fase 1 (PLC, Node-RED, Mosquitto) no cambia. Siguen haciendo exactamente lo mismo.

---

## 2. La topología

```
PLC (Modbus)
    ↓ (FC3, cada 2s)
Node-RED (escala, renombra)
    ↓ (JSON, cada 2s)
Mosquitto (publica en planta/linea1/telemetria)
    ├→ mosquitto_sub (que ves en terminal)
    ├→ Telegraf (que escucha y reenvía)
    │   ↓
    └→ InfluxDB (que almacena)
```

Mosquitto no sabe ni le importa quién se suscribe. Telegraf es un suscriptor más.

---

## 3. Configuración

### Telegraf

Archivo: `telegraf/telegraf.conf`

```ini
[agent]
  interval = "10s"
  round_interval = true
  metric_buffer_limit = 10000
  flush_interval = "10s"

[[inputs.mqtt_consumer]]
  servers = ["tcp://mosquitto:1883"]
  topics = ["planta/#"]
  data_format = "json"

[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "planta"
```

Lo que hace: cada 10 segundos, recoge los mensajes MQTT de tópics que empiezan
con `planta/`, parsea el JSON y escribe en InfluxDB.

El parsing es automático. Cuando Node-RED publica:

```json
{"temperatura_c": 72.5, "presion_bar": 15.2, "rpm": 1450, "estado": 1, "piezas": 12345}
```

Telegraf lo transforma en una fila de InfluxDB:

```
measurement=mqtt_consumer temperatura_c=72.5 presion_bar=15.2 rpm=1450 ...
```

### InfluxDB

```yaml
influxdb:
  image: influxdb:1.8-alpine
  container_name: influxdb
  restart: unless-stopped
  ports:
    - "8086:8086"
  volumes:
    - influxdb_data:/var/lib/influxdb
  networks: [ot]
```

Se inicializa vacío. Telegraf crea la BD `planta` automáticamente en el primer write.

**¿Por qué 1.8 y no 2.7?** La 2.7 requiere tokens de autenticación y setup web.
Para un lab, 1.8 es suficiente: almacena datos igual, sin fricción.

---

## 4. Flujo

**Segundo 1:** PLC responde al Modbus Read con `[725, 152, 1450, 1, 12345]`.

**Segundo 2:** Node-RED lo escala, renombra y publica en MQTT:

```json
{
  "timestamp": "2026-07-24T10:03:41Z",
  "temperatura_c": 72.5,
  "presion_bar": 15.2,
  "rpm": 1450,
  "estado": 1,
  "piezas": 12345
}
```

**Segundo 3:** Telegraf recibe, parsea, escribe en InfluxDB.

**Resultado:** Los datos quedan almacenados con timestamp. Puedes consultarlos
después.

---

## 5. Verificar que funciona

¿Llegaron datos a InfluxDB?

```bash
docker exec influxdb influx -database planta -execute "SHOW MEASUREMENTS"
```

Si ves `mqtt_consumer`, los datos están ahí.

¿Cuántos puntos? (Debe haber uno cada 10 segundos.)

```bash
docker exec influxdb influx -database planta -execute "SELECT COUNT(*) FROM mqtt_consumer"
```

¿Cuáles son los últimos?

```bash
docker exec influxdb influx -database planta -execute "SELECT * FROM mqtt_consumer ORDER BY time DESC LIMIT 5"
```

---

## 6. Por qué esto importa

En la Fase 1, el dato fluye: PLC → Edge → Broker → terminal (mosquitto_sub).

Aquí, el broker tiene **dos consumidores** del mismo topic, independientes entre sí.
Telegraf no sabe que `mosquitto_sub` existe. `mosquitto_sub` no sabe que Telegraf existe.
Ambos simplemente se suscriben y leen.

Ahora imagina mañana: quieres Grafana para graficar. Lo instalas, lo conectas a InfluxDB.
Sin modificar nada de la Fase 1. Sin parar el PLC. Sin tocar Node-RED.

Eso es escalabilidad de verdad en OT.

**Comparación sin pub/sub:** PLC API → Edge → Telegraf API. Agregar un nuevo
consumidor requeriría recablear la integracion. Con Mosquitto: cuelgas y punto.

---

## 7. Limitaciones

No hay visualización. InfluxDB almacena, pero para verlo en gráficas necesitarías
Grafana.

No hay alertas. Si la temperatura sube a 85°C, nadie se entera. Eso va en la
Fase 3 (junto con endurecimiento de seguridad del broker).

No hay normalización en Telegraf. Los datos se escriben crudos. Si algún día necesitas
transformarlos (escalarlos de otra forma, enriquecer con contexto), lo harías en
el flujo de Node-RED o en una capa de procesamiento posterior.

---

## 8. Siguiente

**Fase 3 — Visualización y seguridad:** Grafana leyendo de InfluxDB, TLS en el
broker, autenticación de usuarios.
