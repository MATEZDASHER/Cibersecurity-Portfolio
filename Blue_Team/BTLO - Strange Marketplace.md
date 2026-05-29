#  Blue Team Labs Online: Strange Marketplace - Incident Response Write-up

![](Assets/Attachments/Pasted%20image%2020260519114205.png)


##  Pregunta 1
![](Assets/Attachments/Pasted%20image%2020260519114141.png)

### Metodología de Investigación

1. **Identificación del Vector de Entrada:** El escenario de inteligencia de amenazas indicaba el posible uso de extensiones de VS Code troyanizadas. Como primer paso, procedí a montar la imagen del disco virtual (`.vhdx`) generada por la recolección de evidencias de KAPE. Al auditar el directorio de extensiones del usuario comprometido en la ruta `C:\Users\SBTuser\.vscode\extensions\`, identifiqué una carpeta anómala denominada `geforceext-load.geforceext-load-0.0.1`. Esta extensión utilizaba técnicas de engaño en su nomenclatura y mostraba una fecha de modificación reciente (27/03/2025 a las 14:25), lo que la marcó como el principal artefacto malicioso.
![](Assets/Attachments/Pasted%20image%2020260519120455.png)
![](Assets/Attachments/Pasted%20image%2020260519120511.png)

2. **Correlación de Eventos y Ejecución:** Para determinar el segundo exacto del compromiso y el mecanismo de ejecución, utilicé **Chainsaw** para parsear los registros de eventos de Windows (`.evtx`) extraídos. Ejecuté una búsqueda específica filtrando por el nombre del artefacto (`geforceext`) apuntando a los registros operativos de Sysmon.

> `chainsaw.exe search -e "geforceext" D:\C\Windows\System32\winevt\Logs\`

![](Assets/Attachments/Pasted%20image%2020260519120242.png)

3. **Confirmación del Payload:** Los resultados de Chainsaw revelaron un evento de Creación de Proceso (**Sysmon Event ID 1**). El log confirmó que el acceso inicial y la preparación de la extensión ocurrieron en la marca de tiempo UTC **`2025-03-27 14:25:51`**. Además, el campo `CommandLine` demostró que se abusó de un binario nativo de la instalación de VS Code, **`vsce-sign.exe`**, el cual fue invocado para verificar y procesar la firma del paquete malicioso (`geforceext-load.geforceext-load-0.0.1.sigzip`).
![](Assets/Attachments/Pasted%20image%2020260519120344.png)
![](Assets/Attachments/Pasted%20image%2020260519120615.png)
![](Assets/Attachments/Pasted%20image%2020260519120358.png)

**Respuesta:** `2025-03-27 14:25:51, vsce-sign.exe`
![](Assets/Attachments/Pasted%20image%2020260519114215.png)

---

## Pregunta 2
![](Assets/Attachments/Pasted%20image%2020260519120721.png)

### Metodología de Investigación

Tras identificar el vector inicial en el entorno de desarrollo de VS Code, mi siguiente objetivo fue localizar los datos sensibles comprometidos por la extensión maliciosa. Ampliando la búsqueda más allá de la carpeta del proyecto principal (`ClientA_API`) hacia el directorio raíz del espacio de trabajo del desarrollador (`DeveloperLab`), identifiqué un directorio crítico denominado `Secrets`.
![](Assets/Attachments/Pasted%20image%2020260519132944.png)

Navegando a la ruta `C:\Users\SBTuser\Projects\DeveloperLab\Secrets\`, localicé un archivo llamado `credentials.txt` alojado junto a un archivo de clave privada (`private_key.pem`). La inspección del archivo de texto reveló las credenciales almacenadas en texto plano (`username=admin` y `password=hunter2`), confirmando el alcance del robo de información.

![](Assets/Attachments/Pasted%20image%2020260519132957.png)
![](Assets/Attachments/Pasted%20image%2020260519132829.png)

---

##  Pregunta 3
![](Assets/Attachments/Pasted%20image%2020260519133013.png)

### Metodología de Investigación

1. **Análisis Estático del Payload (Identificación de C2):** Para determinar el destino de la exfiltración, analicé el código fuente de la extensión maliciosa. En el archivo `C:\Users\SBTuser\.vscode\extensions\geforceext-load.geforceext-load-0.0.1\extension.js`, identifiqué claramente la configuración de red *hardcodeada* por el atacante para establecer una conexión de *reverse shell*:
    
    * `let IP = "165.22.189.77";`
    * `let Port = 8080;`
![](Assets/Attachments/Pasted%20image%2020260519135112.png)
![](Assets/Attachments/Pasted%20image%2020260519135131.png)

2. **Análisis Dinámico de Ejecución (Identificación del LOLBin):** Conociendo la infraestructura del atacante, procedí a buscar evidencia de la exfiltración de los archivos identificados en la fase anterior (`credentials.txt`). Utilizando **Chainsaw**, realicé una búsqueda enfocada en el uso de binarios nativos del sistema (LOLBins) comúnmente abusados para transferencias de red, como `certutil.exe`.

> `chainsaw.exe search -e "certutil" D:\C\Windows\System32\winevt\Logs\`

![](Assets/Attachments/Pasted%20image%2020260519135213.png)

Los resultados de Sysmon (Event ID 1) confirmaron la ejecución maliciosa. El atacante utilizó `certutil.exe` con los parámetros `-urlcache -split -f` para interactuar con la IP del C2 previamente descubierta y exfiltrar/descargar archivos relacionados con `credentials.txt`.
![](Assets/Attachments/Pasted%20image%2020260519135248.png)
![](Assets/Attachments/Pasted%20image%2020260519135319.png)

**Respuesta:** `165[.]22[.]189[.]77:8080, certutil.exe`
![](Assets/Attachments/Pasted%20image%2020260519134330.png)

---

##  Pregunta 4
![](Assets/Attachments/Pasted%20image%2020260519135417.png)

### Metodología de Investigación

Tras identificar la exfiltración inicial y la comunicación C2, el siguiente paso fue determinar si el atacante intentó agrupar o comprimir datos adicionales ("staging") antes de su extracción. En entornos Windows, esto suele realizarse mediante comandos de PowerShell.

Utilizando **Chainsaw**, realicé una búsqueda en los registros de eventos de Windows (`.evtx`) enfocada en cmdlets de PowerShell comúnmente utilizados para la compresión de archivos, específicamente `Compress-Archive`.

> `chainsaw.exe search -e "Compress-Archive" D:\C\Windows\System32\winevt\Logs\`

![](Assets/Attachments/Pasted%20image%2020260519141635.png)

La búsqueda en el registro de `Windows PowerShell` (Event ID 800) reveló la ejecución de un comando malicioso el **2025-03-27 a las 15:18:18 UTC**. El campo `Data` del evento mostró la línea de comandos exacta utilizada por el atacante:
![](Assets/Attachments/Pasted%20image%2020260519141742.png)

> `Compress-Archive -Path "C:\Users\SBTuser\Projects\*", "C:\Users\SBTuser\DeveloperLab\*" -DestinationPath "C:\Users\SBTuser\dump.zip"`

![](Assets/Attachments/Pasted%20image%2020260519141811.png)

Este evento me permitió confirmar que el atacante utilizó el cmdlet **`Compress-Archive`** para empaquetar el contenido de los directorios de proyectos del usuario, guardando el archivo resultante (el "staged file") con el nombre **`dump.zip`** en la raíz del directorio.

**Respuesta:** `Compress-Archive, dump.zip`
![](Assets/Attachments/Pasted%20image%2020260519141237.png)

---

##  Pregunta 5
![](Assets/Attachments/Pasted%20image%2020260519141836.png)

### Metodología de Investigación

Tras identificar la comunicación inicial con el servidor de Comando y Control (C2), el objetivo fue determinar qué datos fueron empaquetados ("staged") para su posterior exfiltración y desde qué ubicación exacta se enviaron.

1. **Identificación de Objetivos (Collection):** El análisis de los registros operativos de Windows PowerShell (Event ID 800 / Event ID 4104) reveló la ejecución del comando de compresión malicioso mencionado anteriormente. El uso de `Compress-Archive` apuntando a `C:\Users\SBTuser\Projects\*` y `C:\Users\SBTuser\DeveloperLab\*` confirmó que el objetivo era el código fuente. El archivo inicial se creó en la raíz del usuario (`dump.zip`).
2. **Rastreo de la Evasión (Staging Location):** Sabiendo que los atacantes rara vez dejan los datos exfiltrados a simple vista, procedí a rastrear el ciclo de vida del archivo `dump.zip`. El análisis avanzado mediante el **Event ID 4103 de PowerShell (Module Logging)** me proporcionó la evidencia definitiva. El registro capturó la ejecución del cmdlet `Move-Item`. Al analizar el *Payload* y los *ParameterBindings* de este evento, observé que el atacante trasladó deliberadamente el archivo comprimido desde su ubicación original hacia el directorio temporal del usuario, para evadir detecciones básicas.
![](Assets/Attachments/Pasted%20image%2020260520174508.png)

Por lo tanto, concluí forensemente que la ruta absoluta final desde la cual se exfiltraron los datos fue **`C:\Users\SBTuser\AppData\Local\Temp\Dump.zip`**.

**Respuesta:** `Projects, DeveloperLab, C:\Users\SBTuser\AppData\Local\Temp\Dump.zip`
![](Assets/Attachments/Pasted%20image%2020260520131307.png)

---

##  Pregunta 6
![](Assets/Attachments/Pasted%20image%2020260520170634.png)

### Metodología de Investigación

El último paso de la investigación consistió en determinar el impacto a largo plazo (persistencia) dejado por el atacante en la máquina local. Siguiendo la inteligencia de amenazas proporcionada en el *briefing*, que indicaba la troyanización de herramientas de desarrollo, trasladé el enfoque al entorno vivo del usuario (`C:\Users\SBTuser\DeveloperLab`).

Realicé un análisis de la línea temporal (Timeline Analysis) utilizando PowerShell para identificar qué archivos fueron alterados inmediatamente después de la ventana de intrusión y exfiltración (14:25 UTC). Ejecuté el siguiente comando para aislar la actividad sospechosa:

> `Get-ChildItem -Path C:\Users\SBTuser\DeveloperLab -Recurse -File | Where-Object { $_.LastWriteTime -ge "2025-03-27 14:25:00" } | Select-Object FullName, LastWriteTime`

![](Assets/Attachments/Pasted%20image%2020260520172417.png)

El resultado reveló una anomalía temporal: el archivo **`Gruntfile.js`** (un script legítimo de automatización de tareas en proyectos JavaScript) fue modificado a las 15:57 hora local.

La inspección del código fuente de `Gruntfile.js` confirmó el compromiso. El atacante inyectó de forma encubierta una función `fetch()` en la línea 16, configurada para enviar datos silenciosamente a un **webhook de Discord**. Esta táctica garantiza que el código malicioso se ejecute de manera rutinaria cada vez que el desarrollador compile el proyecto, consolidando así el impacto y la persistencia de la amenaza. Documenté la URL de este webhook como el Indicador de Compromiso (IOC) final.
![](Assets/Attachments/Pasted%20image%2020260520172535.png)
![](Assets/Attachments/Pasted%20image%2020260520172545.png)

**Respuesta:** `Gruntfile.js, https://discord.com/api/webhook/nnvwnficnalc/Thisiswebhook`
![](Assets/Attachments/Pasted%20image%2020260520172350.png)

---

##  Pregunta 7
![](Assets/Attachments/Pasted%20image%2020260520172615.png)

### Metodología de Investigación

Tras descubrir que el archivo de configuración `Gruntfile.js` había sido troyanizado con un webhook de exfiltración, el último paso fue auditar el estado del control de versiones y determinar la autoría de la modificación a nivel de sistema operativo.

Dado que la carpeta del proyecto (`ClientA_API`) era un repositorio gestionado con Git, verifiqué el estado de las modificaciones locales. Es una táctica común que los atacantes eviten realizar un `git commit` de sus inyecciones maliciosas para no dejar un rastro indeleble en el historial del repositorio.

1. **Cuantificación del Impacto:** Ejecuté el comando `git diff --stat` en el directorio del proyecto para obtener un resumen de los cambios no confirmados. La salida del sistema (`1 file changed, 5 insertions(+)`) me permitió determinar matemáticamente que el código inyectado por el atacante constaba exactamente de **5 líneas** de código.
![](Assets/Attachments/Pasted%20image%2020260520173349.png)

2. **Atribución de Autoría:** Para atribuir la acción a la cuenta comprometida, interrogué al sistema sobre el contexto de seguridad activo utilizando el comando nativo `whoami`. El resultado demostró que las modificaciones se realizaron bajo el contexto del usuario **`ec2amaz-tgcpt4n\sbtuser`**, confirmando el uso de los privilegios del desarrollador para efectuar la técnica de persistencia.
![](Assets/Attachments/Pasted%20image%2020260520173406.png)

**Respuesta:** `5, ec2amaz-tgcpt4n\sbtuser`
![](Assets/Attachments/Pasted%20image%2020260520173146.png)
