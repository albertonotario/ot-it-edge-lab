# Fase 1 — Tubería base: PLC → Edge → MQTT

Primer incremento del laboratorio. Establece el flujo de datos completo desde un
autómata simulado hasta un broker de mensajería, atravesando una capa Edge que
transforma el dato crudo en información con semántica.

**Estado:** completada · **Tag:** `fase-1`

---

## 1. Objetivo

Demostrar el patrón fundamental de la convergencia OT/IT: **el dato asciende desde
la planta hacia sistemas de información sin que el control descienda**.

Sobre el modelo Purdue (ISA-95):

| Nivel | Componente | Servicio |
|---|---|---|
| 1–2 | Control y supervisión | `modbus-plc` |
| 3 | Operaciones de planta (Edge) | `node-red` |
| 3.5 → 4 | Frontera hacia IT | `mosquitto` |

La segmentación real de esa frontera (IDMZ) corresponde a la fase 3. En esta fase
todos los servicios comparten una única red Docker.

---

## 2. Componentes

| Servicio | Imagen | Función |
|---|---|---|
| `modbus-plc` | `oitc/modbus-server` 2.3.0 | Esclavo Modbus TCP con registros predefinidos |
| `node-red` | `nodered/node-red` | Adquisición, escalado y publicación |
| `mosquitto` | `eclipse-mosquitto:2` | Broker MQTT |

Nodo adicional en Node-RED: `node-red-contrib-modbus` **5.45.2** (ver
[incidencias](#5-incidencias-y-hallazgos)).

---

## 3. Mapa de registros

Holding registers, leídos con **FC3** (*Read Holding Registers*).

| Dirección | Clave en el JSON | Significado | Valor | Interpretación |
|---|---|---|---|---|
| 0 | `"1"` | Temperatura ×10 | 725 | 72,5 °C |
| 1 | `"2"` | Presión ×10 | 152 | 15,2 bar |
| 2 | `"3"` | RPM motor | 1450 | 1450 rpm |
| 3 | `"4"` | Estado | 1 | 0=parado, 1=marcha, 2=fallo |
| 4 | `"5"` | Contador de piezas | 12345 | 12345 |

### Sobre el escalado ×10

Modbus transporta enteros de 16 bits sin tipo ni unidad asociada. Para transmitir
un decimal, el valor se multiplica por diez en origen y se divide en destino. Ese
conocimiento **no viaja en el protocolo**: reside en la documentación del
fabricante y, en este laboratorio, queda codificado explícitamente en el nodo de
transformación.

Es el contraste directo con OPC-UA, donde el modelo de información acompaña al
dato (tipo, unidad, rango de ingeniería) y esta transformación resultaría
innecesaria.

---

## 4. Flujo Node-RED

Tres nodos encadenados. El export vive en [`node-red/flows.json`](../node-red/flows.json).

### 4.1 `Modbus Read`

| Parámetro | Valor |
|---|---|
| Server → Host | `modbus-plc` |
| Server → Port | `5020` |
| Unit-Id | `1` |
| FC | 3 (Read Holding Registers) |
| Address / Quantity | `0` / `5` |
| Poll Rate | 2 segundos |

El host es el **nombre del servicio**, no una IP: la red `ot` de Docker
proporciona resolución DNS interna, de modo que el flujo sigue siendo válido
aunque cambien las direcciones de los contenedores.

> **Polling.** Modbus es estrictamente petición-respuesta: el esclavo nunca inicia
> comunicación. Conocer el estado del proceso exige preguntar cíclicamente. La
> elección de la frecuencia es un compromiso real en planta — demasiado lenta
> pierde eventos, demasiado rápida carga un autómata que además está controlando
> el proceso.

### 4.2 `function` — escalado y nomenclatura

```javascript
const r = msg.payload;

msg.payload = {
    timestamp: new Date().toISOString(),
    temperatura_c: r[0] / 10,
    presion_bar:   r[1] / 10,
    rpm:           r[2],
    estado:        r[3],
    piezas:        r[4]
};

msg.topic = "planta/linea1/telemetria";

return msg;
```

Es el nodo conceptualmente central del flujo: aquí el dato adquiere significado.
Entra un array anónimo `[725, 152, 1450, 1, 12345]` y sale un objeto
autodescriptivo.

### 4.3 `mqtt out`

| Parámetro | Valor |
|---|---|
| Server | `mosquitto:1883` |
| Topic | *(vacío)* |
| QoS | 0 |
| Retain | No |

El topic se deja vacío deliberadamente para que el nodo utilice el `msg.topic`
recibido. El destino viaja con el dato, lo que permitirá que varias líneas
publiquen en topics distintos reutilizando el mismo nodo de salida.

**QoS 0** ("enviar y olvidar") es adecuado para telemetría continua: perder una
muestra carece de consecuencias cuando llega otra dos segundos después. Un
contador acumulado o una alarma exigirían QoS 1 o 2.

---

## 5. Incidencias y hallazgos

### 5.1 Direccionamiento Modbus

La primera lectura devolvió los valores desplazados una posición. El simulador
interpreta las claves del JSON como **referencias 1-based**: la dirección de
protocolo equivale a la clave menos uno, con independencia del valor del
parámetro `zeroMode` (se probaron ambos).

El origen de la ambigüedad es histórico. El modelo de datos de Modbus numera los
elementos desde 1 y los agrupa por prefijo (`4xxxx` para holding registers),
mientras que el campo de dirección de la trama empieza en 0. Fabricantes y
herramientas eligen una convención u otra sin declararlo.

> **Regla práctica:** el direccionamiento se verifica siempre de forma empírica
> con un valor conocido en el primer registro. Nunca se asume que la
> documentación y el dispositivo coinciden.

### 5.2 Dependencia rota en npm

La instalación de `node-red-contrib-modbus` 5.50.0 desde el gestor de paletas
falla con error 404 sobre `@openp4nr/modbus-serial@8.4.0`, una dependencia
transitiva bajo un scope propio del mantenedor que el registro público no sirve.

Solución aplicada — fijar la versión anterior:

```bash
docker exec -it -w /data node-red npm install node-red-contrib-modbus@5.45.2
docker restart node-red
```

Es un caso de manual de **riesgo de cadena de suministro de software**: un
despliegue reproducible ayer deja de serlo hoy sin que nadie haya modificado
nada. En entornos OT, donde los sistemas permanecen en servicio durante una
década y se parchean con poca frecuencia, esta fragilidad tiene consecuencias
reales. Es uno de los frentes que cubre IEC 62443 en la gestión de componentes.

---

## 6. Verificación

Lectura directa del esclavo, sin pasar por el Edge:

```bash
docker run --rm --network host oitc/modbus-client \
  -s 127.0.0.1 -p 5020 -t 3 -r 0 -l 5
```

Extremo a extremo, suscribiéndose al broker:

```bash
docker exec -it mosquitto mosquitto_sub -t 'planta/#' -v
```

Salida esperada, una línea cada dos segundos:

```
planta/linea1/telemetria {"timestamp":"...","temperatura_c":72.5,"presion_bar":15.2,"rpm":1450,"estado":1,"piezas":12345}
```

---

## 7. Deuda técnica reconocida

Se documenta de forma explícita porque son decisiones conscientes, no descuidos.

| Elemento | Situación | Fase de resolución |
|---|---|---|
| Broker sin autenticación | `allow_anonymous true` | 3 |
| Broker sin cifrado | Escucha en 1883, sin TLS | 3 |
| Sin ACL por topic | Cualquier cliente publica en cualquier topic | 3 |
| Puerto 5020 publicado en el host | La planta es alcanzable desde la LAN | 3 |
| Etiquetas `:latest` en el compose | Reproducibilidad aparente | Pendiente |
| Valores estáticos | El simulador no varía | 2 |

El endurecimiento del broker se reserva deliberadamente para la fase 3: permite
documentar el contraste entre una configuración por defecto y una configuración
endurecida, en lugar de presentar únicamente el resultado final.

---

## 8. Siguiente fase

**Fase 2 — Persistencia y visualización.** Valores dinámicos en el simulador,
almacenamiento en InfluxDB y representación en Grafana. Ambos consumidores se
suscriben al broker sin modificar el PLC ni el Edge: es la ventaja del
desacoplamiento que aporta MQTT.
