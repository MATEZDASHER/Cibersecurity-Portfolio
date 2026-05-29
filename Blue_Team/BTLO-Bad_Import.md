![](../Assets/Attachments/Pasted%20image%2020260506105248.png)

Procedemos a arrancar la máquina y accedemos a Splunk a través de Firefox.


Pregunta 1:
**¿Qué estación de trabajo contactó primero al servidor de control del atacante? Proporciona la estación de trabajo, la IP de destino y el puerto de destino.**
![](../Assets/Attachments/Pasted%20image%2020260506105327.png)

Para identificar la primera conexión, buscamos eventos de red (Event ID 3 de Sysmon) dirigidos a una IP externa desde múltiples equipos, ya que el enunciado indica que "tres máquinas" hablaron con el servidor.

Consulta:
index=* EventCode=3 | stats count by Computer, DestinationIp, DestinationPort
![](../Assets/Attachments/Pasted%20image%2020260506105559.png)
- `index=* EventCode=3`: Busca en todos los datos eventos de tipo 3 (Conexión de red de Sysmon).
- `| stats count by...`: Crea una tabla resumiendo cuántas veces (`count`) cada computadora (`Computer`) se conectó a qué IP (`DestinationIp`) y en qué puerto (`DestinationPort`).

Se descartan las direcciones IP privadas (rangos 10.x.x.x, 172.16.x.x - 172.31.x.x, y 192.168.x.x) al corresponder a tráfico de red local. También se omiten aquellas direcciones con un número de conexiones no significativo. Ordenando por `DestinationIp`, observamos que las IPs que se repiten en tres máquinas distintas son: `117.12.214.46` y `142.11.206.73`.

![](../Assets/Attachments/Pasted%20image%2020260506111913.png)
![](../Assets/Attachments/Pasted%20image%2020260506112329.png)

Se descarta la IP `117.12.214.46` ya que, al analizar los eventos y revisar el campo `Image`, refleja la ejecución de `MsMpEng.exe` en su ruta habitual por el usuario `NT AUTHORITY\SYSTEM`.
![](../Assets/Attachments/Pasted%20image%2020260506112635.png)![](../Assets/Attachments/Pasted%20image%2020260506112643.png)
**Nota:** `MsMpEng.exe` (Microsoft Malware Protection Engine) es el proceso ejecutable principal de Windows Defender. Es un componente legítimo que protege el sistema en tiempo real.

La IP restante es `142.11.206.73`. Analizando los eventos de la estación `DEV-WKS-01.meridian.local`, el campo `Image` indica la ejecución de `C:\ProgramData\wt.exe` por el usuario `MERIDIAN\jcarter`. Resulta altamente sospechoso el `DestinationHostname` asociado: `sfrclak.com`.
![](../Assets/Attachments/Pasted%20image%2020260506114022.png)



Una investigación de los artefactos revela la siguiente información:

![](../Assets/Attachments/Pasted%20image%2020260506113558.png)
`wt.exe` es el archivo ejecutable legítimo de Windows Terminal. Sin embargo, su ejecución desde `C:\ProgramData\` es un indicador de compromiso (IoC) claro de suplantación o binario renombrado.


Adicionalmente, la comprobación del dominio `sfrclak.com` confirma su naturaleza maliciosa.
![](../Assets/Attachments/Pasted%20image%2020260506120109.png)
![](../Assets/Attachments/Pasted%20image%2020260506120118.png)

Identificada la IP del Command and Control (C2), procedemos a buscar la primera conexión temporal.

Consulta ejecutada:
index=* EventCode=3 DestinationIp="TU_IP_SOSPECHOSA_AQUI"
| sort _time

Este comando busca todas las conexiones hacia la IP maliciosa y las ordena cronológicamente de la más antigua a la más reciente. La primera ejecución nos proporciona los datos solicitados.

Respuesta:
![](../Assets/Attachments/Pasted%20image%2020260506120246.png)
DEV-WKS-01, 142.11.206.73, 8000
![](../Assets/Attachments/Pasted%20image%2020260506120315.png)

---

pregunta 2:
![](../Assets/Attachments/Pasted%20image%2020260506120508.png)

Buscamos el directorio desde el cual se lanzó el primer payload (`C:\ProgramData\wt.exe`). Filtramos por la creación del proceso (Event ID 1) ordenado por tiempo.

**Consulta ejecutada:**
index=* EventCode=1 Image="C:\\ProgramData\\wt.exe" | sort _time
![](../Assets/Attachments/Pasted%20image%2020260508142032.png)
Con esta consulta Le estamos diciendo a Splunk: _"Busca el evento de creación de proceso (EventCode=1) que corresponda exactamente con el binario malicioso wt.exe disfrazado de Powershell"_.

Esta búsqueda nos devuelve 4 eventos, pero nos centramos en el más antiguo.
![](../Assets/Attachments/Pasted%20image%2020260508142229.png)

En los eventos de creación de procesos de Sysmon (Event ID 1), el "directorio de trabajo" desde el cual se lanza el proceso queda registrado en el campo `CurrentDirectory`.
![](../Assets/Attachments/Pasted%20image%2020260508141918.png)
Esta seria la respuesta para la pregunta 2 : C:\projects\analytics-portal\node_modules\plain-crypto-js
![](../Assets/Attachments/Pasted%20image%2020260508141858.png)


Para la Pregunta 3 (El binario de Windows renombrado):
![](../Assets/Attachments/Pasted%20image%2020260506155138.png)
El atacante ejecutó `C:\ProgramData\wt.exe`. Analizando el campo `Description` y los parámetros del comando (`-NoProfile -ep Bypass -File`), se identifica claramente que el binario subyacente es PowerShell.

Respuesta pregunta 3: Windows Powershell
![](../Assets/Attachments/Pasted%20image%2020260506155100.png)

**Para la Pregunta 4 (La ruta completa del script _dropped_):** 
![](../Assets/Attachments/Pasted%20image%2020260506155243.png)
Observando el final de la línea de comandos ejecutada por el binario, se identifica la ruta del archivo `.ps1` que se pasa como argumento.
![](../Assets/Attachments/Pasted%20image%2020260506155234.png)
Respuesta pregunta 4: C:\Users\jcarter\AppData\Local\Temp\6202033.ps1

---

pregunta 5:
![](../Assets/Attachments/Pasted%20image%2020260506155444.png)
Para identificar el mecanismo de persistencia en el registro (frecuentemente mediante las claves `Run` o `RunOnce`), buscamos eventos de modificación de valores bajo el Event ID 13 de Sysmon.

Consulta ejecutada:

 index=* EventCode=13 Computer="DEV-WKS-01.meridian.local" TargetObject"*\\Run\\*"
![](../Assets/Attachments/Pasted%20image%2020260506161818.png)
- `EventCode=13`: Solo queremos eventos donde se ha establecido un _valor_ en el registro.
- `Computer=...`: Filtramos por la máquina que sabemos que está comprometida.
- `TargetObject="*\\Run\\*"`: Filtramos por la ruta del registro. Las claves de persistencia más comunes contienen la palabra "Run" (ej. `Software\Microsoft\Windows\CurrentVersion\Run`). Los asteriscos actúan como comodines.

De los resultados devueltos, destaca el archivo oculto `.run.bat`.
![](../Assets/Attachments/Pasted%20image%2020260506161838.png)

Extrayendo los datos relevantes:

- **`TargetObject`** (ruta completa del registro): `HKU\S-1-5-21-1004336348-1177238915-682003330-1001\Software\Microsoft\Windows\CurrentVersion\Run\WindowsTerminalUpdate`
- **`Details`** (el valor/payload): `C:\ProgramData\.run.bat`


 HKU\S-1-5-21-1004336348-1177238915-682003330-1001\Software\Microsoft\Windows\CurrentVersion\Run\WindowsTerminalUpdate
![](../Assets/Attachments/Pasted%20image%2020260506162446.png)


 C:\ProgramData\.run.bat 
![](../Assets/Attachments/Pasted%20image%2020260506162628.png)

Respuesta pregunta 5:
![](../Assets/Attachments/Pasted%20image%2020260506162718.png)
HKU\S-1-5-21-1004336348-1177238915-682003330-1001\Software\Microsoft\Windows\CurrentVersion\Run\WindowsTerminalUpdate,  C:\ProgramData\.run.bat 


---
pregunta 6:

(El atacante recolectó varios archivos que contenían credenciales y secretos. Proporciona el nombre de los archivos en orden alfabético, empezando por el archivo que empieza con un punto).
![](../Assets/Attachments/Pasted%20image%2020260506180542.png)


Se procede a buscar la ejecución de comandos de lectura o copia de archivos sensibles.

Tras una búsqueda inicial de palabras clave (`password`, `secret`, `credential`), identificamos un evento que utiliza `Invoke-Command`. Esto indica un movimiento lateral hacia la máquina `BUILD-01.meridian.local` con ejecución de reconocimiento (`whoami`, `hostname`).

Para localizar la exfiltración de archivos, se filtran los comandos comunes de manipulación y lectura por línea de comandos.

**Consulta ejecutada:**

index=* Computer="DEV-WKS-01.meridian.local" (CommandLine="*password*" OR CommandLine="*secret*" OR CommandLine="*.ssh*" OR CommandLine="*.aws" OR CommandLine="*credential*")
![](../Assets/Attachments/Pasted%20image%2020260506193956.png)


El análisis de los campos `CommandLine` revela la lista exacta de archivos recolectados y movidos a la carpeta `Temp`:



1. `C:\Users\jcarter\.aws\credentials`
![](../Assets/Attachments/Pasted%20image%2020260508104320.png)
2. `C:\Users\jcarter\.ssh\id_rsa`
![](../Assets/Attachments/Pasted%20image%2020260508104331.png)
3. `C:\Users\jcarter\AppData\Roaming\npm\.npmrc`
![](../Assets/Attachments/Pasted%20image%2020260508104349.png)
4. C:\Users\jcarter\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
![](../Assets/Attachments/Pasted%20image%2020260508104404.png)
5. `C:\Users\jcarter\AppData\Local\Google\Chrome\User Data\Default\Login Data`
![](../Assets/Attachments/Pasted%20image%2020260508104421.png)

Nota: Omitimos `cmdkey` y `reg query` porque esos comandos leen del sistema, no copian un archivo físico.

Aplicando el formato solicitado (orden alfabético, archivo con punto primero y excluyendo rutas completas:

Respuesta Pregunta 6: .npmrc, ConsoleHost_history.txt, credentials, id_rsa, Login Data
![](../Assets/Attachments/Pasted%20image%2020260506193834.png)

---

pregunta 7:
![](../Assets/Attachments/Pasted%20image%2020260508104434.png)
El atacante luego se movió lateralmente a la siguiente máquina. ¿Cuál es la máquina y qué cuenta se usó?

Anteriormente se detectó el uso de `Invoke-Command` hacia la máquina `BUILD-01`. Para confirmar el movimiento lateral y obtener la hora exacta, analizamos los inicios de sesión de red (Logon Type 3).

**Consulta ejecutada:**

index=* EventCode=4624 LogonType="3"
![](../Assets/Attachments/Pasted%20image%2020260508111603.png)
(Usamos LogonType=3 porque el atacante se conectó por red usando WinRM/PowerShell).
![](../Assets/Attachments/Pasted%20image%2020260508110356.png)
- **`index=*`**: Busca en todos los índices disponibles (todos los datos indexados).
- **`EventCode=4624`**: Filtra para mostrar únicamente los eventos de "Inicio de sesión exitoso" (An Inquiry was made) del registro de seguridad de Windows.
- **`LogonType="3"`**: Especifica que el tipo de inicio de sesión fue de **Red** (Network)

Se confirma el pivote hacia la máquina `BUILD-01.meridian.local` comprometiendo el usuario del dominio `svc_jenkinsdeploy`. Este evento (Event ID 4624) nos proporciona simultáneamente la fecha y hora de la intrusión.

Respuesta pregunta 7: BUILD-01, svc_jenkinsdeploy
![](../Assets/Attachments/Pasted%20image%2020260508105757.png)


Respuesta pregunta 8: 2026-04-02, 19:35:46
![](../Assets/Attachments/Pasted%20image%2020260508105856.png)

---

pregunta 9
![](../Assets/Attachments/Pasted%20image%2020260508111903.png)

Para detectar la técnica de _Timestomping_ (falsificación de fechas de creación de archivos para evasión), se analiza el Event ID 2 de Sysmon en el nuevo host comprometido.

**Consulta ejecutada:**

index=* EventCode=2  Computer=BUILD-01.meridian.local
![](../Assets/Attachments/Pasted%20image%2020260508114239.png)


Al revisar los eventos, el campo `PreviousCreationUtcTime` presenta el valor `2026-01-14 11:00:00.000`. El valor carece de milisegundos aleatorios, lo que es un indicador determinista de que la marca de tiempo fue modificada artificialmente por el atacante para encubrir los archivos `build.bat` y `app.js`.

![](../Assets/Attachments/Pasted%20image%2020260508114250.png)


![](../Assets/Attachments/Pasted%20image%2020260508114300.png)

Respuesta pregunta 9: 2026-01-14, 11:00:00
![](../Assets/Attachments/Pasted%20image%2020260508113612.png)

---
pregunta 10
![](../Assets/Attachments/Pasted%20image%2020260508115333.png)
(Las modificaciones del atacante permitieron que su código llegara a un tercer host... Identifica el hostname de este próximo host y el protocolo/puerto TCP)

Se requiere identificar las conexiones de red establecidas desde el servidor de Jenkins (`BUILD-01`) hacia el servidor de producción durante la fase de despliegue.

**Consulta ejecutada:**

index=* EventCode=3 Computer="BUILD-01.meridian.local"
| stats count by DestinationIp, DestinationHostname, DestinationPort
![](../Assets/Attachments/Pasted%20image%2020260508120056.png)

Análisis de las conexiones:
![](../Assets/Attachments/Pasted%20image%2020260508120105.png)
- `dc1.meridian.local` (123, 389, 445, 88): Tráfico legítimo del Controlador de Dominio (LDAP, Kerberos, NTP).
- `github.com` / `registry.npmjs.org`: Tráfico legítimo de Jenkins (descarga de repositorios/dependencias).
- `sfrclak.com` (8000): Comunicación con la infraestructura de C2.
- `SRV-APP-01.meridian.local` (445, 5985): Tercer host comprometido (Servidor de Aplicaciones).

De los puertos utilizados hacia el servidor de producción, el `5985` corresponde a WinRM, mientras que el `445` (SMB) es el protocolo empleado para la transferencia de archivos.

respuesta pregunta 10:
SRV-APP-01, 445
![](../Assets/Attachments/Pasted%20image%2020260508120039.png)


---

pregunta 11:
![](../Assets/Attachments/Pasted%20image%2020260508120921.png)
(Un nuevo archivo apareció en la carpeta de plugins de la aplicación customer-portal. ¿Proporciona su ruta completa?)

Para identificar el artefacto desplegado en la carpeta de plugins del servidor de producción, se busca la creación de archivos (Event ID 11).

**Consulta ejecutada:**

index=* EventCode=11 Computer="SRV-APP-01.meridian.local" TargetFilename="*plugins*"
![](../Assets/Attachments/Pasted%20image%2020260508121134.png)


El campo `TargetFilename` revela la ruta completa del archivo malicioso depositado (`telemetry.js`).
![](../Assets/Attachments/Pasted%20image%2020260508121140.png)

respuesta pregunta 11: C:\inetpub\custome-portal\plugins\telemetry.js
![](../Assets/Attachments/Pasted%20image%2020260508121126.png)

---

pregunta 12:
![](../Assets/Attachments/Pasted%20image%2020260508122043.png)

Se cruzan los datos de la IP del C2 (`142.11.206.73`) con los eventos de conexión saliente (Event ID 3) del host de producción comprometido.

**Consulta ejecutada:**
index=* EventCode=3 Computer="SRV-APP-01.meridian.local" DestinationIp=142.11.206.73 | sort -_time
![](../Assets/Attachments/Pasted%20image%2020260508122303.png)
**¿Qué hace esto?** Busca todas las conexiones de red que salieron de `SRV-APP-01` hacia la IP del atacante y las ordena cronológicamente (la más antigua arriba).

Revisando el primer evento cronológico, extraemos la imagen del proceso y el usuario bajo el que se ejecutó.

![](../Assets/Attachments/Pasted%20image%2020260508122316.png)


respuesta pregunta 12 :
C:\Program Files\nodejs\node.exe, svc_portal
![](../Assets/Attachments/Pasted%20image%2020260508122254.png)

---
pregunta 13:
![](../Assets/Attachments/Pasted%20image%2020260508123627.png)
Para analizar la escalada de privilegios en el servidor de producción mediante el abuso de "Named Pipes", se auditan los Event IDs 17 (Pipe Created) y 18 (Pipe Connected).


**Consulta ejecutada:**

index=* (EventCode=17 OR EventCode=18) Computer="SRV-APP-01.meridian.local"
| stats count by Image, PipeName
![](../Assets/Attachments/Pasted%20image%2020260508130146.png)
**¿Qué hace esto?** Agrupa todos los eventos de creación/conexión de pipes y muestra qué binarios (`Image`) interactuaron con qué pipes (`PipeName`).

Filtrando los procesos legítimos del sistema operativo, el análisis estadístico aísla la ejecución del binario anómalo `svc_helper.exe` interactuando con el pipe `\spoolss`.
![](../Assets/Attachments/Pasted%20image%2020260508130155.png)


respuesta pregunta 13 : C:\ProgramData\svc_helper.exe, spoolss
![](../Assets/Attachments/Pasted%20image%2020260508130450.png)

---
pregunta 14 :
![](../Assets/Attachments/Pasted%20image%2020260508131203.png)
La **Pregunta 14** nos pide encontrar cómo el atacante logró **persistencia** en el servidor de producción (`SRV-APP-01.meridian.local`) después de haber escalado privilegios. Nos pide el "nombre registrado" ("Registered Name").

Para localizar la técnica de persistencia post-escalada en el servidor de producción (normalmente Tareas Programadas o Servicios de Windows), se audita el registro de creación de servicios. Al no obtener resultados concluyentes monitorizando el Event ID 7045 ni comandos directos de creación de servicios, se procede a rastrear la persistencia a nivel del Registro de Windows (Event ID 13).

index=* EventCode=13 Computer="*SRV-APP-01*" TargetObject="*\\System\\CurrentControlSet\\Services\\*"
![](../Assets/Attachments/Pasted%20image%2020260508132733.png)


![](../Assets/Attachments/Pasted%20image%2020260508132748.png)

Respuesta pregunta 14: WinTelemetrySvc
![](../Assets/Attachments/Pasted%20image%2020260508132526.png)
