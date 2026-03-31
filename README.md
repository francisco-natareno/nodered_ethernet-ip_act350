# Comunicación EtherNet/IP entre Node-Red y Transmisor Mettler Toledo ACT350

## 1. Arquitectura de Comunicación

El módulo ACT350 implementa el protocolo SAI (Standard Automation Interface) de Mettler Toledo sobre EtherNet/IP. La comunicación se basa en el intercambio cíclico de bloques de datos entre un "Scanner" (Node-Red actuando como controlador) y un "Adapter" (el ACT350).

```
┌─────────────────┐       EtherNet/IP (Class 1 - Cíclico)       ┌──────────────────┐
│                 │  ──── Assembly 100 (Output: 16 bytes) ────► │                  │
│    Node-RED     │                                             │      ACT350      │
│   (Scanner)     │  ◄─── Assembly 101 (Input: 16 bytes)  ───── │     (Adapter)    │
│                 │                                             │                  │
│  IP: x.x.x.x    │       EtherNet/IP (Class 3 - Acíclico)      │  IP: 172.16.0.8  │
│                 │  ◄──────── Explicit Messaging ──────────►   │                  │
└─────────────────┘                                             └──────────────────┘
```

### Conceptos clave de EtherNet/IP

- **Scanner**: El dispositivo que inicia la conexión I/O (Node-Red). Equivale al "IO Controller" en PROFINET.
- **Adapter**: El dispositivo de campo (ACT350). Responde a las solicitudes del Scanner.
- **Assembly**: Bloque de datos agrupados que se intercambian cíclicamente. Se identifican por número de instancia.
- **RPI (Requested Packet Interval)**: Frecuencia de actualización en milisegundos. El ACT350 soporta de 1 ms a 100 ms.
- **Class 1 Messaging**: Datos cíclicos (I/O implícito) — peso, estado, comandos.
- **Class 3 Messaging**: Datos acíclicos (mensajes explícitos) — configuración, calibración, diagnósticos.

---

## 2. Mapa de Memoria: Formato 2 Bloques (16 bytes)

El ACT350 soporta dos formatos de conexión. El formato de 2 bloques (Connection1) es el recomendado para aplicaciones industriales porque incluye tanto el bloque de medición como el bloque de estado.

### 2.1 Datos de ENTRADA (T→O: del ACT350 hacia Node-Red) — Assembly 101

Estos son los datos que **leeremos** del dispositivo. Esta es la organización de sus 16 bytes:

| Offset (bytes) | Tamaño |    Tipo    |     Nombre SAI       |            Descripción               |
|:--------------:|:------:|:----------:|:--------------------:|:------------------------------------:|
|      0–3       |   4    | FLOAT32 LE |  MB Measuring Value  | Valor de peso (según comando activo) |
|      4–5       |   2    |  UINT16 LE |   MB Device Status   | Bits de estado del dispositivo       |
|      6–7       |   2    |  INT16 LE  |      MB Response     | Respuesta al comando enviado         |
|      8–9       |   2    |  UINT16 LE |   SB Status Group 1  | Alarmas RedAlert                     |
|     10–11      |   2    |  UINT16 LE |   SB Status Group 2  | Estado de báscula / Alarmas          |
|     12–13      |   2    |  UINT16 LE |   SB Status Group 3  | Estado I/O discretas                 |
|     14–15      |   2    |  INT16 LE  |      SB Response     | Respuesta al comando de status       |

### 2.2 Datos de SALIDA (O→T: de Node-Red hacia el ACT350) — Assembly 100

Estos son los datos que **escribimos** hacia el dispositivo. También son 16 bytes:

| Offset (bytes) | Tamaño |    Tipo    |     Nombre SAI       |              Descripción                    |
|:--------------:|:------:|:----------:|:--------------------:|:-------------------------------------------:|
|      0–3       |   4    | FLOAT32 LE |   MB Command Value   | Valor flotante opcional para el comando     |
|      4–5       |   2    |  UINT16 LE |   MB Channel Mask    | Máscara de canal (0 para dispositivo único) |
|      6–7       |   2    |  INT16 LE  |      MB Command      | Comando del bloque de medición              |
|      8–9       |   2    |  INT16 LE  |     SB Reserved 1    | Reservado (escribir 0)                      |
|     10–11      |   2    |  INT16 LE  |     SB Reserved 2    | Reservado (escribir 0)                      |
|     12–13      |   2    |  INT16 LE  |     SB Reserved 3    | Reservado (escribir 0)                      |
|     14–15      |   2    |  INT16 LE  |      SB Command      | Comando del bloque de status                |

### 2.3 Byte Order

Para EtherNet/IP, el ACT350 utiliza **Little Endian** (nativo de Rockwell/Allen-Bradley). Esto significa que el byte menos significativo se almacena primero en memoria. En Node-Red con la librería `@serafintech/node-red-contrib-eip-io`, se configura `bigEndian: false`.

---

## 3. Decodificación de Bits de Estado (MB Device Status — Word 4–5)

El word de estado del dispositivo (bytes 4-5 del Assembly de entrada) contiene 16 bits individuales:

| Bit  |          Nombre          |              Significado cuando ACTIVO (1)                        |
|:----:|:------------------------:|:-----------------------------------------------------------------:|
|  0   |      Sequence Bit 0      | Toggle de secuencia (para sincronización de comandos)             |
|  1   |      Sequence Bit 1      | Toggle de secuencia                                               |
|  2   |        Heartbeat         | Alterna cada ~1 segundo (confirma que el dispositivo está activo) |
|  3   |         Data OK          | Los datos son válidos. Si es 0 → error crítico                    |
|  4   |     Alarm Condition      | Hay una condición de alarma activa                                |
|  5   |     Center of Zero       | El peso bruto está en ±¼ de intervalo alrededor de cero           |
|  6   |          Motion          | Lectura de peso NO estable (hay movimiento)                       |
|  7   |         Net Mode         | Se está reportando peso neto en lugar de bruto                    |
|  8   |   Alternate Weight Unit  | Se está usando una unidad de peso alternativa                     |
| 9–15 |    Device Specific Bits  | Bits específicos del dispositivo (comparadores, etc.)             |

---

## 4. Tabla de Comandos Principales (MB Command)

Los comandos se escriben en el byte offset 6 – 7 del Assembly de salida:

### 4.1 Comandos de Reporte (solicitan datos continuos)

| Comando |          Descripción                | Valor Float requerido |
|:-------:|:-----------------------------------:|:---------------------:|
|    0    | Peso bruto redondeado (por defecto) |          NO           |
|    1    | Peso bruto redondeado               |          NO           |
|    2    | Peso de tara redondeado             |          NO           |
|    3    | Peso neto redondeado                |          NO           |
|    5    | Peso bruto resolución interna       |          NO           |
|    6    | Peso de tara resolución interna     |          NO           |
|    7    | Peso neto resolución interna        |          NO           |

### 4.2 Comandos de Operación (ejecutan una acción)

| Comando |        Descripción       |
|:-------:|:------------------------:|
|   400   |   Tara cuando estable    |
|   401   |   Cero cuando estable    |
|   402   |      Limpiar tara        |
|   403   |     Tara inmediata       |
|   404   |     Cero inmediato       |

### 4.3 Comandos de Escritura (escriben un valor)

| Comando |         Descripción           |    Valor Float requerido   |
|:-------:|:-----------------------------:|:--------------------------:|
|   201   | Preset tara (unidad display)  |    Peso de tara deseado    |
|   240   | Escribir límite comparador 1  |      Valor del límite      |
|   242   | Escribir límite comparador 2  |      Valor del límite      |
|   290   |   Escribir modo de pesaje     | 0=Universal, 2=Filtro fijo |

### 4.4 Comandos del Sistema

| Comando |               Descripción                        |
|:-------:|:------------------------------------------------:|
|  2000   | NOOP – Sin operación (limpiar comando anterior)  |
|  2004   |           Cancelar operación en curso            |
|  2047   |       (Solo respuesta) Comando en proceso        |
|  2046   |         (Solo respuesta) Paso exitoso            |

---

## 5. Configuración en Node-Red

### 5.1 Librerías Requeridas

Instalar desde la paleta de Node-Red o desde la línea de comandos:

```bash
npm install @serafintech/node-red-contrib-eip-io
npm install node-red-contrib-buffer-parser
```

- **@serafintech/node-red-contrib-eip-io**: Proporciona los nodos `eip-io scanner`, `eip-io connection`, `eip-io in` y `eip-io out` para comunicación EtherNet/IP Class 1 (implícita).
- **node-red-contrib-buffer-parser**: Facilita la interpretación de buffers binarios a tipos de datos legibles (float, int, bits).

### 5.2 Configuración del Scanner

El nodo `eip-io scanner` representa al controlador EtherNet/IP. Configurar:

- **Name**: `EIP-Scanner` (identificador descriptivo)
- **IP Address**: Dejar vacío (se configura en la conexión individual)

### 5.3 Configuración de la Conexión

El nodo `eip-io connection` define los parámetros de la conexión I/O al dispositivo específico. Los valores clave extraídos del archivo EDS son los siguientes:

|    Parámetro    |             Valor            |                      Explicación                   |
|:---------------:|:----------------------------:|:--------------------------------------------------:|
|    IP Address   |           172.16.0.8         | Dirección IP del ACT350 en la red                  |
|      RPI        |              50              | Intervalo de actualización en ms (rango: 1–100 ms) |
| Assembly Config |              128             | Instancia de assembly de configuración             |
|   Size Config   |              0              | Tamaño en bytes del assembly de configuración      |
|  Assembly Input |              101             | Instancia del assembly de entrada (T→O)            |
|    Size Input   |              16              | Tamaño en bytes de los datos de entrada            |
| Assembly Output |              100             | Instancia del assembly de salida (O→T)             |
|    Size Output  |              16              | Tamaño en bytes de los datos de salida             |
|    Connection   |         Connection1          | Formato 2 bloques (medición + estado)              |
|     EDS File    | (contenido del archivo .eds) | Describe las capacidades del dispositivo           |

### 5.4 Carga del Archivo EDS

El contenido del archivo EDS (Electronic Data Sheet) se debe copiar y pegar en la configuración de la conexión. Este archivo le indica a la librería las capacidades del dispositivo: los assemblies disponibles, sus tamaños y los formatos de conexión soportados.

---

## 6. Explicación Detallada del Código

### 6.1 Lectura: Nodo `eip-io in`

```
[eip-io in] → [buffer-parser] → [function: decodeResponse] → [debug]
```

El nodo `eip-io in` se configura para leer los 128 bits (16 bytes) completos del Assembly 101 como un `Buffer` sin procesar. Los parámetros clave son:

- `byteOffset: 0` — Empieza a leer desde el byte 0
- `bitSize: 128` — Lee los 16 bytes completos (16 × 8 = 128 bits)
- `bigEndian: false` — Little Endian (convención de EtherNet/IP)
- `dataType: Buffer` — Retorna el buffer crudo para procesamiento manual
- `updateRate: 1` — Emite un mensaje cada vez que hay datos nuevos

El `buffer-parser` divide el buffer en campos con nombre: `measureValue` (float32), `deviceStatus` (uint16), `mbResponse` (int16), y los tres grupos de status (uint16 cada uno).

La función `decodeResponse` extrae cada bit individual usando operaciones bitwise:

```javascript
// Ejemplo: extraer el bit 6 (motion) del deviceStatus
const motion = (deviceStatus >> 6) & 1;
// >> 6 desplaza 6 posiciones a la derecha
// & 1 aísla solo el bit menos significativo
```

### 6.2 Escritura: Nodo `eip-io out`

```
[inject con msg.comando] → [function: preparePacket] → [eip-io out]
```

La función `preparePacket` recibe un objeto `msg.comando` y prepara el paquete de datos que se enviará al Assembly 100 del dispositivo. Cada campo requerido en el mensaje SAI se envía por separado a nodos eip-io out individuales, que tienen configurado un Byte Offset / Data Type específico según el campo correspondiente. Esto es así ya que la librería `@serafintech/node-red-contrib-eip-io` utiliza comunicación `Implícita` para el envío de datos hacia el dispositivo de campo.

```javascript
const cmd = msg.comando || {};
const codigo  = cmd.codigo    || 0;
const valor   = cmd.valor     || 0.0;
const sbCmd   = cmd.sbComando || 0;

// Salida 1: FLOAT32 Command Value → offset 0
const msg1 = { payload: valor };

// Salida 2: UINT16 Channel Mask → offset 4 (siempre 0 para single device)
const msg2 = { payload: 0 };

// Salida 3: INT16 MB Command → offset 6
const msg3 = { payload: codigo };

// Salida 4: INT16 SB Command → offset 14
const msg4 = { payload: sbCmd };
```

### 6.3 Secuencia de Comandos

El protocolo SAI exige una secuencia específica para enviar comandos:

1. **Enviar el comando**: Escribir el código deseado en el MB Command (bytes 6-7).
2. **Verificar la respuesta**: Leer el MB Response (bytes 6-7 del assembly de entrada).
   - Si el response iguala al comando enviado → el comando fue aceptado.
   - Si el response tiene bit 15 = 1 → ocurrió un error (ver bits 0-10 para el tipo de error).
   - Si el response = 2047 → el comando está siendo procesado.
3. **Enviar NOOP (2000)**: Antes de enviar el mismo comando dos veces seguidas, se debe enviar un NOOP intermedio.
4. **Verificar sequence bits**: Los bits 0-1 del Device Status cambian con cada comando procesado, confirmando que el dispositivo recibió y actuó sobre el comando.

### 6.4 Verificación de Respuesta a Comando

Para implementar una lógica robusta de envío de comandos con verificación:

```javascript
// En el nodo function de decodificación, agregar:
const respValue = resp & 0x07FF;
const respError = (resp >> 15) & 1;

if (respError) {
    const errorCode = respValue;
    const errorMessages = {
        1: 'Comando inválido',
        2: 'Timeout',
        4: 'Comando desconocido',
        8: 'Valor de comando inválido',
        16: 'Comando abortado',
        32: 'Paso fallido',
        64: 'Comando de test fallido'
    };
    msg.error = errorMessages[errorCode] || `Error desconocido: ${errorCode}`;
}
```

---

## 7. Buenas Prácticas para Implementación Industrial

### 7.1 Monitoreo de Comunicación

Siempre verificar el bit **Heartbeat** (bit 2 del Device Status). Este bit alterna cada segundo. Si deja de alternar, significa que la comunicación con el módulo de pesaje se ha perdido.

```javascript
// Ejemplo de detección de pérdida de heartbeat
let lastHeartbeat = null;
let lastChangeTime = Date.now();

// En cada lectura:
if (heartbeat !== lastHeartbeat) {
    lastHeartbeat = heartbeat;
    lastChangeTime = Date.now();
}
if (Date.now() - lastChangeTime > 3000) {
    // Alarma: sin cambio de heartbeat por más de 3 segundos
}
```

### 7.2 Verificación de Datos Válidos

Nunca usar el valor de peso sin verificar que el bit **Data OK** (bit 3) está en 1. Cuando Data OK = 0, el dispositivo tiene un error crítico y el valor reportado no es confiable.

### 7.3 Espera de Estabilidad

Para operaciones de pesaje de precisión, siempre verificar que el bit **Motion** (bit 6) esté en 0 antes de registrar el peso. Los comandos 400 (tara cuando estable) y 401 (cero cuando estable) incluyen esta verificación internamente, pero si se necesita lógica personalizada se debe monitorear este bit.

### 7.4 Manejo del NOOP

Cuando se necesita enviar el mismo comando repetidamente (por ejemplo, tara), se debe intercalar un comando NOOP (2000) entre cada envío:

```
Tara (400) → verificar respuesta → NOOP (2000) → verificar respuesta → Tara (400)
```

### 7.5 Selección de RPI

Para aplicaciones de pesaje industrial típicas, un RPI de 50 ms es adecuado. Para aplicaciones de alta velocidad (check-weighers, llenado) se puede reducir hasta 10 ms, pero esto aumenta la carga de red. El ACT350 soporta un mínimo de 1 ms (Param60 en el EDS: 1000-100000 µs).

### 7.6 Formato de Conexión: 1 Bloque vs 2 Bloques

- **1 Bloque** (Connection2, Assembly 102/103, 8 bytes): Solo incluye el bloque de medición. Útil cuando solo se necesita el peso y no se requieren alarmas ni I/O.
- **2 Bloques** (Connection1, Assembly 100/101, 16 bytes): Incluye medición y status. Recomendado para cualquier aplicación industrial compleja ya que proporciona acceso a alarmas, estado de I/O y diagnósticos.

---

## 8. Referencia Rápida de Errores de Respuesta

| Bit 15 | Valor (bits 0-10) |                Significado                     |
|:------:|:-----------------:|:----------------------------------------------:|
|   1    |        1          | Comando inválido (conocido pero no ejecutable) |
|   1    |        2          | Timeout (requiere estabilidad no obtenida)     |
|   1    |        4          | Comando desconocido (no soportado)             |
|   1    |        8          | Valor inválido (fuera de rango)                |
|   1    |        16         | Comando abortado                               |
|   1    |        32         | Paso fallido (en secuencia multi-paso)         |
|   1    |        64         | Comando de test fallido                        |

---

## 9. Notas Importantes

1. **El ACT350 solo permite 1 conexión Class 1 simultánea** (MaxIOConnections = 1 en el EDS). Si Node-Red pierde la conexión, debe reconectarse antes de que otro cliente lo haga.

2. **El archivo EDS debe cargarse en la configuración** del nodo `eip-io connection` para que la librería reconozca los assemblies disponibles y sus tamaños, así como también los formatos de conexión.

3. **Los datos acíclicos (Class 3)** para calibración, ajuste y diagnósticos avanzados utilizan CIP Generic Messages con los parámetros Class/Instance/Attribute documentados en las secciones 11-13 del manual SAI. La librería `@serafintech/node-red-contrib-eip-io` está diseñada principalmente para datos cíclicos.

4. **Siempre enviar un comando NOOP (2000)** al iniciar la comunicación para limpiar cualquier estado residual del bloque de comando.
