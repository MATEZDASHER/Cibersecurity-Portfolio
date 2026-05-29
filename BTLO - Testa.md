# Blue Team Labs Online: Testa - Incident Response Write-up


![](Assets/Attachments/Pasted%20image%2020260527093752.png)

## Pregunta 1
![](Assets/Attachments/Pasted%20image%2020260527111056.png)
_¿Cuál es la dirección IP de origen no autorizada que aparece en el segmento OT?_

#### **Metodología de Análisis**
Para resolver este punto, realicé un análisis cruzado entre el tráfico de red capturado (`ot-forensic-inc.pcap`) y la arquitectura de red aprobada en el manual de operaciones de Jurassic Resourcing.

1. **Revisión de Documentación:** Primero, consulté la sección de _Quick Reference_ y el diagrama de red del manual para identificar el esquema de direccionamiento legítimo del segmento OT (`172.20.0.0/24`).
2. **Identificación de Línea Base:** Confirmé que solo existen tres dispositivos autorizados en este segmento: el controlador de carga del barco (`.2`), el TLCS de la terminal (`.3`) y la estación de trabajo de ingeniería (`.4`).
3. **Análisis de Tráfico:** Abrí el archivo de captura en Wireshark y navegué a **Statistics > Endpoints > IPv4** para listar todos los hosts que estaban generando o recibiendo tráfico.

#### **Evidencia Recopilada**
**Evidencia A: Arquitectura Aprobada** _(Any other -> Unauthorised)._
![](Assets/Attachments/Pasted%20image%2020260527111747.png)

**Evidencia B: Tráfico Anómalo** _(IP `172.20.0.5`)._
![](Assets/Attachments/Pasted%20image%2020260527111826.png)

* **`172.20.0.1`**: Esta IP no está en la tabla, pero por convención de redes (y la cantidad de tráfico que tiene), determiné que actúa como el **Default Gateway** (la interfaz del IT/OT DMZ Jump Host que se ve en el diagrama del punto 3 del pdf).
![](Assets/Attachments/Pasted%20image%2020260527112132.png)

Al contrastar la Evidencia A con la Evidencia B, observé tráfico proveniente de la IP `172.20.0.5`, la cual no figura en el inventario de activos legítimos.

#### **Conclusión y Respuesta**
Basado en la regla estricta de denegación por defecto del entorno OT, concluí que el dispositivo intruso es la dirección IP **`172.20.0.5`**.

**Respuesta:** `172.20.0.5`
![](Assets/Attachments/Pasted%20image%2020260527111101.png)

---

## Pregunta 2
![](Assets/Attachments/Pasted%20image%2020260527112227.png)
_¿Qué códigos de función Modbus utiliza el host no autorizado y cuáles son operaciones de escritura?_

#### **Metodología de Análisis**
Para determinar las acciones exactas que el intruso (`172.20.0.5`) intentó ejecutar contra los controladores industriales, procedí a aislar e inspeccionar su tráfico.

1. **Decodificación de Puertos No Estándar:** Inicialmente, el tráfico hacia los puertos `5020` y `5021` figuraba como TCP genérico. Dado que el manual de arquitectura indica que estos son los puertos Modbus de la terminal, utilicé la función _Decode As..._ en Wireshark para forzar la interpretación de dichos puertos como `MBTCP` (Modbus TCP).
2. **Filtrado de Tráfico Malicioso:** Apliqué el filtro `ip.src == 172.20.0.5 && mbtcp` para visualizar exclusivamente los comandos inyectados por el atacante.
3. **Extracción y Clasificación:** Analicé los paquetes resultantes y extraje los _Function Codes_. Posteriormente, los crucé con la tabla de referencia del manual de operaciones para determinar su naturaleza (Lectura o Escritura).
4. **Validación Exhaustiva:** Apliqué un filtro de exclusión (`ip.src == 172.20.0.5 && mbtcp && modbus.func_code != 1 && modbus.func_code != 4 && modbus.func_code != 5 && modbus.func_code != 16`) para garantizar que no existiera ningún otro código oculto en la captura. El resultado fue cero, confirmando mi lista definitiva.

#### **Evidencia Recopilada**
**Evidencia A: Forzado de Protocolo Modbus (Decode As)** _(el puerto 5020/5021 se asigna a MBTCP)._
![](Assets/Attachments/Pasted%20image%2020260527115654.png)

**Evidencia B: Tráfico del Atacante** _(filtro `ip.src == 172.20.0.5 && mbtcp` aplicado)._
![](Assets/Attachments/Pasted%20image%2020260527115745.png)

**Evidencia C: Clasificación según el Manual** _(captura de pantalla de la tabla del punto 8.2 del manual donde se explica el propósito de cada código)._
![](Assets/Attachments/Pasted%20image%2020260527115831.png)

Al analizar las evidencias, identifiqué los siguientes códigos:
- **FC01** (Read Coils) -> Operación de Lectura (R)
- **FC04** (Read Input Registers) -> Operación de Lectura (R)
- **FC05** (Write Single Coil) -> Operación de Escritura (W)
- **FC16** (Write Multiple Registers) -> Operación de Escritura (W)

#### **Conclusión y Respuesta**
El atacante realizó labores de reconocimiento (Lectura) e intentó alterar la lógica del sistema (Escritura). Aplicando el formato técnico requerido, la cadena de comandos identificada es:

**Respuesta:** `FC01(R),FC04(R),FC05(W),FC16(W)`
![](Assets/Attachments/Pasted%20image%2020260527115335.png)

---

## Pregunta 3
![](Assets/Attachments/Pasted%20image%2020260527120049.png)
_¿Qué valor intentó escribir el host no autorizado en el registro de retención (holding register) 40000 de la Terminal TLCS?_

#### **Metodología de Análisis**
Para identificar el valor específico inyectado por el atacante en el controlador industrial, aislé el paquete de escritura correspondiente y decodifiqué su carga útil (_payload_) a nivel de bytes.

1. **Filtrado de Escrituras Específicas:** Apliqué un filtro avanzado en Wireshark para aislar el tráfico proveniente del atacante (`172.20.0.5`) hacia la Terminal TLCS (`172.20.0.3`), enfocándome exclusivamente en el protocolo Modbus TCP y la función de escritura múltiple de registros (**FC16**).
    - _Filtro utilizado:_ `ip.src == 172.20.0.5 && ip.dst == 172.20.0.3 && mbtcp && modbus.func_code == 16`
2. **Análisis del Desfase de Direccionamiento (Offset):** En el protocolo Modbus, el "Holding Register 40000" se transmite en la red utilizando la dirección base `0x0000` (el número 4 inicial es solo una convención lógica para definir el tipo de registro). Por lo tanto, busqué el paquete que apuntara a la dirección de inicio `0`.
3. **Análisis del Volcado Hexadecimal (Hex Dump):** Debido a limitaciones en el disector automático de Wireshark para interpretar la estructura interna del paquete (alerta de _Expert Info_), procedí a realizar una lectura manual de la trama de datos directamente desde el volcado hexadecimal de la capa de aplicación:
    - La secuencia identificada en la línea de datos fue: `01 10 00 00 00 02 04 23 28 00 00`
    - **`01`**: Unit ID.
    - **`10`**: Function Code (16 en decimal: Write Multiple Registers).
    - **`00 00`**: Starting Address (Registro lógico 40000).
    - **`00 02`**: Word Count (Escritura en dos registros contiguos).
    - **`04`**: Byte Count (4 bytes de datos adjuntos).
    - **`23 28`**: Primeros dos bytes de datos cargados, pertenecientes al Registro 40000.
4. **Conversión de Datos:** Extraje el valor en crudo `2328` (en base 16 / hexadecimal) y lo convertí a sistema decimal para obtener el parámetro real enviado al proceso físico.

#### **Evidencia Recopilada**
**Evidencia A: Paquete de Escritura Aislado** _(Captura de pantalla de Wireshark con el filtro aplicado, mostrando el paquete 3172)._
![](Assets/Attachments/Pasted%20image%2020260527122645.png)

**Evidencia B: Análisis Manual del Payload Hexadecimal** _(Bytes `23 28` resaltados en la matriz hexadecimal para demostrar el origen del dato)._
![](Assets/Attachments/Pasted%20image%2020260527122710.png)

**Evidencia C: Impacto en el Proceso Físico según el Manual** _(Tabla "3.2 System Components / Holding Registers" del manual, donde se observa que el registro 40000 corresponde a la variable `BATCH_TARGET` con un rango normal de operación de 1 a 900 m³)._
![](Assets/Attachments/Pasted%20image%2020260527122729.png)

Al realizar la conversión matemática del valor hexadecimal obtenido en la **Evidencia B**:
$$\text{0x2328} = (2 \times 16^3) + (3 \times 16^2) + (2 \times 16^1) + (8 \times 16^0) = 8192 + 768 + 32 + 8 = 9000$$

#### **Conclusión y Respuesta**
El host no autorizado intentó forzar un parámetro crítico en el sistema manipulando la variable `BATCH_TARGET`. Al saltarse los límites establecidos en la interfaz de operación legítima, el atacante inyectó el valor bruto de 9000. 

_Nota de análisis:_ Este valor supera por un factor de 10 el límite máximo permitido de 900 m³ estipulado en el manual de operaciones, lo que confirma una clara intención de sabotaje físico o desbordamiento en el control de almacenamiento de crudo.

**Respuesta:** `9000`
![](Assets/Attachments/Pasted%20image%2020260527121719.png)

---

## Pregunta 4
![](Assets/Attachments/Pasted%20image%2020260527122811.png)
_¿Cuántas válvulas de los tanques de carga (cargo tank valves) abrió el atacante?_

#### **Metodología de Análisis**
Para determinar el alcance físico del ataque contra la infraestructura del barco, audité las operaciones de escritura discreta (encendido/apagado) dirigidas al controlador de carga de la embarcación.

1. **Aislamiento de Comandos Discretos:** Apliqué un filtro en Wireshark para capturar exclusivamente las peticiones de inyección de estado (_Write Single Coil_, **FC05**) originadas por la IP no autorizada (`172.20.0.5`) hacia el _Ship Cargo Controller_ (`172.20.0.2`).
    - _Filtro utilizado:_ `ip.src == 172.20.0.5 && ip.dst == 172.20.0.2 && mbtcp && modbus.func_code == 5`
2. **Decodificación de Carga Útil (Hexadecimal):** Analicé los paquetes interceptados a nivel de _hex dump_ para identificar:
    - **Dirección destino:** La bobina (_Coil_) específica que se estaba manipulando.
    - **Instrucción de control:** El valor inyectado, donde `FF 00` indica apertura (ON) y `00 00` indica cierre (OFF).
3. **Correlación de Contexto Físico:** Extraje la lista completa de bobinas atacadas y la contrasté con las tablas de referencia del manual de operaciones (_Quick Reference_ y sección _2.4 Valves_) para distinguir qué comandos afectaban realmente a los tanques de carga y cuáles a otros sistemas del barco.

#### **Evidencia Recopilada**
**Evidencia A: Intercepción de Tráfico de Red** _(Captura de Wireshark con el filtro aplicado, mostrando los 5 paquetes resultantes, y la estructura `05 00 XX FF 00` en el hex dump)._
![](Assets/Attachments/Pasted%20image%2020260527123926.png)
![](Assets/Attachments/Pasted%20image%2020260527123948.png)

Del análisis de la Evidencia A, detecté 5 comandos individuales de apertura (`FF 00`) dirigidos a las siguientes direcciones lógicas (Coils):
- Bobina `01`
- Bobina `03`
- Bobina `04`
- Bobina `05`
- Bobina `06`

**Evidencia B: Mapeo de Variables Industriales** _(El manual muestra que "Which tank valves are open?" apunta a las "Ship Coils 00003-00006", y la tabla 2.4 define las LV-C1 a LV-C4 como "Cargo tank loading valves")._
![](Assets/Attachments/Pasted%20image%2020260527124021.png)
![](Assets/Attachments/Pasted%20image%2020260527124057.png)

Al realizar la correlación cruzada, determiné que:
- La orden dirigida a la bobina `01` **no** corresponde a una válvula de tanque de carga (presumiblemente afecta a la válvula de succión XV-101).
- Las órdenes dirigidas a las bobinas `03, 04, 05` y `06` corresponden exactamente a las cuatro válvulas de los tanques de carga (LV-C1 a LV-C4).

#### **Conclusión y Respuesta**
Aunque el atacante logró inyectar con éxito 5 comandos de apertura en la red OT, mi análisis contextual de la ingeniería de la planta confirma que solo un subconjunto de estos comandos afectó al objetivo evaluado. El número exacto de válvulas de tanques de carga que el atacante logró abrir es 4.

**Respuesta:** `4`
![](Assets/Attachments/Pasted%20image%2020260527123835.png)

---

## Pregunta 5
![](Assets/Attachments/Pasted%20image%2020260527130103.png)
_¿A qué hora exacta hace la transición de 0 a 1 la bobina SHORE_STOP_ACTIVE del lado del barco, deteniendo las operaciones de la instalación? (Formato: hh:mm:ss:xxxxxx)_

#### **Metodología de Análisis**
Para localizar el momento exacto en que el sistema automatizado asertó la parada de emergencia sin depender de marcas de tiempo o rangos de paquetes predefinidos, procedí a buscar la firma hexadecimal exacta del cambio de estado dentro del tráfico de red crudo.

1. **Definición del Patrón de Búsqueda:** Según el manual de operaciones, la terminal lee el estado de la bobina 0 (`SHORE_STOP_ACTIVE`) mediante consultas constantes (_polling_). Cuando esta bobina pasa a estado de alarma (True/1), la respuesta Modbus TCP (_Read Coils Response_) finaliza con la secuencia de bytes correspondiente a la activación: `01 01 01 01`.
2. **Filtrado Base de Comunicaciones:** Apliqué un filtro inicial para aislar las respuestas enviadas desde el barco (`172.20.0.2`) a través del puerto de control de carga `5021`.
    - _Filtro utilizado:_ `tcp.port == 5021 && ip.src == 172.20.0.2`
3. **Búsqueda Avanzada de Payload:** Utilizando la herramienta de búsqueda de Wireshark (`Ctrl + F`), configuré el motor para escanear Valores Hexadecimales (_Hex value_) directamente dentro de los bytes de los paquetes (_Packet bytes_). Introduje la firma objetivo: `01010101`.
4. **Extracción del Timestamp:** La búsqueda automatizada localizó instantáneamente la primera ocurrencia de esta cadena. Posteriormente, ajusté la vista temporal de Wireshark (_Time of Day / Microseconds_) para obtener la marca de tiempo absoluta del evento.

#### **Evidencia Recopilada**
**Evidencia A: Ejecución de la Búsqueda Hexadecimal** _(Wireshark mostrando la configuración "Hex value" y la cadena "01010101", resaltando el salto automático al paquete 4479)._
![](Assets/Attachments/Pasted%20image%2020260527132937.png)

**Evidencia B: Confirmación en el Volcado de Datos (Hex Dump)** _(Panel inferior del paquete 4479, confirmando inequívocamente que el bit de la bobina 0 ha transicionado a 1)._
![](Assets/Attachments/Pasted%20image%2020260527132954.png)

#### **Conclusión y Respuesta**
La búsqueda en bruto sobre el _payload_ TCP me permitió identificar el **Paquete 4479** como la primera transmisión en la que el barco notificó a la terminal que la condición de emergencia había sido asertada. Adaptando la marca de tiempo de dicho paquete al formato requerido por el reporte, la hora exacta del evento fue:

**Respuesta:** `12:52:37:255230`
![](Assets/Attachments/Pasted%20image%2020260527130039.png)