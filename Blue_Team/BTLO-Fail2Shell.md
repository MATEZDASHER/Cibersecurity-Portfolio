![](../Assets/Attachments/Pasted%20image%2020260512095952.png)

Arrancamos la máquina y, una vez dentro, un `README` nos indica que Volatility 2 y 3 están instalados en `/home/btlo/tools`. Accedemos a WSL desde cmd ejecutando `wsl` (o usando Ubuntu) y ya estamos listos para empezar con el análisis.
![](../Assets/Attachments/Pasted%20image%2020260512100033.png)

![](../Assets/Attachments/Pasted%20image%2020260512100231.png)

### **Pregunta 1**

**Tras el movimiento lateral, determine la dirección IP de origen desde la cual el atacante se autenticó con éxito en nuestro servidor comprometido.**
![](../Assets/Attachments/Pasted%20image%2020260512100336.png)


Para identificar el origen del acceso, busqué las variables de entorno de los procesos almacenados en la captura de memoria. Al establecer una sesión SSH, el sistema guarda variables como `SSH_CONNECTION` con los datos de red. Utilicé el plugin `linux.envars` de Volatility 3 para extraer esto.

python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/Fail2Shell/mem.mem linux.envars.Envars | grep -iE "SSH_CONNECTION|SSH_CLIENT"

Evidencia: 
![](../Assets/Attachments/Pasted%20image%2020260512103013.png)

La salida muestra que los procesos con PID 1492 y 1493 contienen la variable `SSH_CONNECTION`. El valor sigue el formato `[IP Origen] [Puerto Origen] [IP Destino] [Puerto Destino]`. Esto confirma una conexión establecida hacia el puerto 22 (SSH) del servidor local, originada desde la IP 192.168.1.100.


**Respuesta:** 192.168.1.100
![](../Assets/Attachments/Pasted%20image%2020260512103042.png)


---

Pregunta 2:
![](../Assets/Attachments/Pasted%20image%2020260512103106.png)
Observamos tráfico de red que contenía un archivo ZIP llamado 'good.zip', indicando una probable exfiltración de datos. Después de eso, el atacante estableció un mecanismo de persistencia antes de terminar su sesión. Identifique la ruta completa del sistema de archivos de este artefacto de persistencia que otorga al atacante acceso al servidor.


Como los plugins típicos de historial de comandos no mostraron el mecanismo exacto de persistencia, revisé los procesos activos en memoria, centrándome en el usuario comprometido (`shared`). Con el plugin `linux.psaux` pude ver la lista de procesos junto con los argumentos y rutas de ejecución completas.

python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/Fail2Shell/mem.mem linux.psaux.PsAux
![](../Assets/Attachments/Pasted%20image%2020260513110048.png)


**Evidencia:**
![](../Assets/Attachments/Pasted%20image%2020260513102756.png)
La salida revela un proceso en ejecución (PID 1493) bajo el contexto del usuario `shared`. Sin embargo, el binario que está ejecutando está en `/home/ubuntu/.local/shared`. Alojar un ejecutable dentro de un directorio oculto (`.local`) en el _home_ de otro usuario es una técnica clásica para esconderse. Esta anomalía confirma que se trata del payload de persistencia.

Respuesta: /home/ubuntu/.local/shared
![](../Assets/Attachments/Pasted%20image%2020260513102546.png)

----

Pregunta 3:
![](../Assets/Attachments/Pasted%20image%2020260513102938.png)
Determine el nombre del conocido módulo de Linux explotado por el atacante para mantener un acceso persistente.

Tras descartar la presencia de rootkits a nivel de kernel (Loadable Kernel Modules / `.ko`) mediante la revisión de procesos e invocaciones, la investigación se enfocó en técnicas de persistencia a nivel de usuario. Al haber identificado el ejecutable malicioso (`/home/ubuntu/.local/shared`) en la fase anterior, era imperativo encontrar el mecanismo que actuaba como "gatillo" para lanzarlo. En sistemas Linux, el framework PAM (Pluggable Authentication Modules) es un vector frecuente para este propósito.

Después de descartar rootkits a nivel de kernel, me enfoqué en técnicas de persistencia a nivel de usuario. Ya teníamos localizado el ejecutable malicioso, por lo que tocaba buscar qué actuaba como "gatillo" para lanzarlo.

Para que el malware nos dé acceso sin tener que depender del `cron` o servicios evidentes, el atacante abusó de `pam_exec.so`. Es un módulo legítimo de Linux que sirve para ejecutar scripts externos durante el ciclo de autenticación de un usuario. Al meterlo en los archivos de `/etc/pam.d/`, el sistema ejecuta silenciosamente la puerta trasera en cada inicio de sesión.

**Respuesta:** pam_exec.so

![](../Assets/Attachments/Pasted%20image%2020260513102947.png)

---
Pregunta 4:
![](../Assets/Attachments/Pasted%20image%2020260513103202.png)
Proporcione el número de inodo del archivo que contiene la configuración de persistencia del atacante.

Sabiendo que usó `pam_exec.so` y que el acceso se hizo por SSH, la configuración de persistencia debía estar en `/etc/pam.d/sshd`. Para sacar el número de inodo directamente desde la RAM, usé `linux.pagecache.Files` filtrando por el directorio PAM.



python3 vol.py -f /mnt/c/Users/BTLOTest/Desktop/Artefacts/Fail2Shell/mem.mem linux.pagecache.Files | grep -iE "pam\.d|shared"

![](../Assets/Attachments/Pasted%20image%2020260513105656.png)

Evidencia:
![](../Assets/Attachments/Pasted%20image%2020260513105741.png)


Respuesta: 6293323
![](../Assets/Attachments/Pasted%20image%2020260513105611.png)

---
Pregunta 5:
![](../Assets/Attachments/Pasted%20image%2020260513110107.png)
Identifique la aplicación del servidor C2 desplegada por el atacante durante el compromiso.


Al no encontrar firmas de frameworks C2 típicos (Cobalt Strike, Sliver, etc.), pasé a revisar las cabeceras HTTP cacheadas en RAM buscando alguna anomalía en los User-Agents.


strings /mnt/c/Users/BTLOTest/Desktop/Artefacts/Fail2Shell/mem.mem | grep -i "^User-Agent:" | sort | uniq -c | sort -nr | head -n 15

![](../Assets/Attachments/Pasted%20image%2020260513115220.png)

Evidencia:

![](../Assets/Attachments/Pasted%20image%2020260513115232.png)

Entre los User-Agents normales del sistema (wget, curl, snapd), encontré un cliente automatizado en Python que usaba `discord.py`. Tener un bot de Discord en este servidor canta muchísimo. Los atacantes lo usan a menudo como C2 porque el tráfico va por HTTPS normal y se salta los bloqueos de red tradicionales.

**Respuesta:** Discord
![](../Assets/Attachments/Pasted%20image%2020260513115146.png)


---
Pregunta 6:
![](../Assets/Attachments/Pasted%20image%2020260513115504.png)
Localizar la fecha de creación del bot de Discord.

Sabiendo que el C2 era Discord, busqué el token de autenticación del bot en la memoria, ya que en su propia estructura lleva incrustada la fecha de creación. Usé una expresión regular para cazar el formato típico de estos tokens.

strings mem.mem | grep -oE "[a-zA-Z0-9_-]{24}\.[a-zA-Z0-9_-]{6}\.[a-zA-Z0-9_-]{27,39}" | head -n 5

![](../Assets/Attachments/Pasted%20image%2020260513122504.png)

Evidencia:
![](../Assets/Attachments/Pasted%20image%2020260513122524.png)

El escaneo saca el token activo del atacante. El primer segmento del token es el Snowflake ID en Base64 (`MTI4MDI2MDk0NzI4OTcwNjYxMA`).

- Al decodificarlo en CyberChef obtenemos el ID: `1280260947289706610`.
- Desplazando esto 22 bits a la derecha (`>> 22`) sacamos el timestamp interno: `305237995932`.
- Le sumamos la Discord Epoch (`1420070400000`) y nos da `1725308395932`.
- Convirtiendo ese valor Unix (1725308395) a fecha, obtenemos la respuesta en UTC.

**Respuesta:** 2024-09-02 20:19:55
![](../Assets/Attachments/Pasted%20image%2020260513122722.png)


---
Pregunta 7:
![](../Assets/Attachments/Pasted%20image%2020260513122838.png)
Determina la dirección IP externa y el puerto de escucha del servidor de comando y control (C2).

Como `linux.netstat` no dio resultados útiles para este volcado, pero sabíamos que el bot de Discord (PID 1493) tenía sockets activos, busqué por patrones en crudo. La API de Discord va enrutada por Cloudflare (rango 162.159.0.0/16) y se comunica obligatoriamente por HTTPS. Filtré la memoria en busca de esas IPs.


strings mem.mem | grep -oE "162\.159\.[0-9]{1,3}\.[0-9]{1,3}" | sort | uniq -c | sort -nr | head -n 5
![](../Assets/Attachments/Pasted%20image%2020260513125456.png)

**Evidencia:**
![133](../Assets/Attachments/Pasted%20image%2020260513125512.png)

La IP `162.159.137.232` es la que más se repite con diferencia, lo cual cuadra perfectamente con un bot haciendo _polling_ continuo a los servidores de Discord para ver si hay comandos nuevos. Al ser hacia la API oficial, el puerto de escucha remoto es obligatoriamente el 443.

**Respuesta:** 162.159.137.232:443
![](../Assets/Attachments/Pasted%20image%2020260513125550.png)

---

Pregunta 8:
![](../Assets/Attachments/Pasted%20image%2020260513125705.png)
Identificar la hora en la que el atacante validó el acceso persistente

Después de comprobar que los tiempos de modificación en PAM databan de 2022 y descartar Fail2Ban, revisé el Page Cache del Kernel mediante Volatility. Esto me permitió ver la interacción directa del SO con el binario malicioso (`/home/ubuntu/.local/shared`) y recuperar sus metadatos MAC en memoria.

python3 vol.py -f mem.mem linux.pagecache.Files | grep -iE "shared|/etc/pam.d/sshd"
![](../Assets/Attachments/Pasted%20image%2020260514091106.png)

Evidencia:
![](../Assets/Attachments/Pasted%20image%2020260514091140.png)

El volcado revela que el archivo se subió al sistema a las `05:38:10 UTC` (M-Time) y se le dieron permisos de ejecución a las `05:39:21 UTC` (C-Time). El Access Time (A-Time) es `05:40:00 UTC`, el momento exacto de su primera ejecución. El hecho de que se ejecutara clavado en el segundo `00` es el indicador claro de que se lanzó mediante una tarea programada (`cron`). Así validó la persistencia.

**Respuesta:** 2025-07-01 05:40:00
![](../Assets/Attachments/Pasted%20image%2020260514091009.png)



---
Pregunta 9:

![](../Assets/Attachments/Pasted%20image%2020260513180429.png)
Identifica la hora de eliminación (UTC) del directorio de registros como parte de las actividades antiforense del atacante.

Como el plugin `linux.bash.Bash` no arrojó resultados sobre comandos destructivos, supuse que los ejecutó directamente mandando mensajes al bot de Discord, sin abrir una terminal. Filtrando la memoria en crudo por comandos habituales como `rm -r /var/log`, encontré las estructuras JSON de la API.


strings mem.mem | grep -B 2 -E "rm -rf /var/log|rm -r /var/log"
![](../Assets/Attachments/Pasted%20image%2020260513180728.png)

Evidencia:
![](../Assets/Attachments/Pasted%20image%2020260513180754.png)

Se observa un JSON donde el atacante (usuario "m4shl3") manda la instrucción `!cmd rm -r /var/log` al bot. El propio JSON guarda el _timestamp_ generado por los servidores de Discord en formato UTC (`09:17:08`), y un segundo después el bot le confirma que el comando se ha ejecutado con éxito.


**Respuesta:** 2025-07-01 09:17:08
![](../Assets/Attachments/Pasted%20image%2020260513180416.png)

---
Pregunta 10:
![449](../Assets/Attachments/Pasted%20image%2020260513190252.png)
Recuperar el último comando ejecutado por el atacante en el sistema comprometido.

Aprovechando que los comandos C2 se quedaban guardados en texto plano en la memoria, extraje y ordené cronológicamente todos los comandos enviados al bot de Discord que usaban el prefijo `!cmd`.

strings /mnt/c/Users/BTLOTest/Desktop/Artefacts/Fail2Shell/mem.mem | grep '"content":"!cmd'
![](../Assets/Attachments/Pasted%20image%2020260513191044.png)

**Evidencia:**
![](../Assets/Attachments/Pasted%20image%2020260513191108.png)

La extracción muestra una secuencia clara: tras borrar los logs (`rm -r /var/log` y `rm /root/.bash_history`), la última instrucción registrada antes del volcado de RAM fue a las `09:23:17 UTC`. El atacante estaba enumerando directorios y revisando permisos dentro de la carpeta personal del usuario legítimo `allam`.

**Respuesta:** ls -lha /home/allam/
![](../Assets/Attachments/Pasted%20image%2020260513190300.png)
