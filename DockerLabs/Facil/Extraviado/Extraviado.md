
## Resumen

Es una máquina de dificultad fácil en la que accedemos mediante SSH y luego escalamos privilegios gracias a archivos ocultos y contraseñas codificadas en base64.

## Enumeración

Empiezo haciendo un ping a la máquina para saber su sistema operativo.

```bash
ping -c 1 172.17.0.2
```

El TTL de la respuesta es 64, así que sé que es Linux y procedo con nmap.

```bash
nmap -sVC -p- --min-rate 5000 172.17.0.2 -oN nmap
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 cc:d2:9b:60:14:16:27:b3:b9:f8:79:10:df:a1:f3:24 (ECDSA)
|_  256 37:a2:b2:b2:26:f2:07:d1:83:7a:ff:98:8d:91:77:37 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.58 (Ubuntu)
MAC Address: 0E:02:AA:8B:EB:09 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos abiertos:

```text
SSH/22
Apache/80
```

Como todavía no tenemos credenciales, empiezo por Apache.

## Análisis del puerto 80

Hago un curl a la página.

```bash
curl 172.17.0.2
```

Es la página por defecto de Apache, pero al final veo algo que no cuadra: dos textos en base64 con el formato `usuario : contraseña`.

```text
ZGFuaWVsYQ== : Zm9jYXJvamE=
```

Los decodifico y veo que son credenciales que probablemente pueda usar en el SSH.

```bash
echo ZGFuaWVsYQ== | base64 -d
```

```text
daniela
```

```bash
echo Zm9jYXJvamE= | base64 -d
```

```text
focaroja
```

## Análisis del puerto 22 y escalada de privilegios

Uso las credenciales para iniciar sesión por SSH.

```bash
ssh daniela@172.17.0.2
```

Una vez dentro como `daniela`, hago un `ls -a` en busca de carpetas ocultas.

```bash
ls -a
```

```text
.  ..  .bash_logout  .bashrc  .cache  .local  .profile  .secreto  Desktop
```

La carpeta `.secreto` es interesante. Dentro hay un archivo llamado `passdiego` con un texto en base64 que, al decodificarlo, nos da una contraseña que probablemente sea del usuario `diego`.

```bash
cd .secreto
ls
```

```text
passdiego
```

```bash
cat passdiego
```

```text
YmFsbGVuYW5lZ3Jh
```

```bash
base64 -d passdiego
```

```text
ballenanegra
```

Ahora inicio sesión como `diego`.

```bash
su diego
```

```text
diego@e1e694dc4401:~$ id
uid=1001(diego) gid=1001(diego) groups=1001(diego),100(users)
```

Como `diego`, empiezo a buscar más directorios ocultos.

```bash
ls -a
```

```text
.  ..  .bash_history  .bash_logout  .bashrc  .local  .passroot  .profile  pass
```

Entré en `.passroot` y leí `pass`, pero eran _rabbit holes_, así que seguí buscando.

Después de un rato, entré en `.local/share` y encontré un archivo oculto llamado `.-`.

```bash
cd ~/.local/share
ls -a
```

```text
.  .-  ..  nano
```

Al abrirlo me encuentro con un acertijo.

```bash
cat .-
```

```text
password de root

En un mundo de hielo, me muevo sin prisa,
con un pelaje que brilla, como la brisa.
No soy un rey, pero en cuentos soy fiel,
de un color inusual, como el cielo y el mar
también.
Soy amigo de los niños, en historias de
ensueño.
Quién soy, que en el frío encuentro mi dueño?
```

El acertijo nos da la contraseña de root. La contraseña es `osoazul`.

Inicio sesión como `root`.

```bash
su root
```

```text
root@e1e694dc4401:~# id
uid=0(root) gid=0(root) groups=0(root)
```

Con esto doy por completada la máquina.

## Flujo de ataque

1. **Enumeración:** con nmap sacamos los puertos abiertos `SSH/22` y `Apache/80`.
2. **Análisis del puerto 80:** con curl sacamos las credenciales de SSH de `daniela`, codificadas en base64.
3. **Acceso por SSH:** entramos como `daniela` y encontramos en la carpeta oculta `.secreto` la contraseña de `diego`, también en base64.
4. **Escalada de privilegios:** como `diego`, encontramos en una carpeta oculta un acertijo que, al resolverlo, nos da la contraseña de `root`.

 