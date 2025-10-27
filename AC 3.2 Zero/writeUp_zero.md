# 📝 Write‑up ZERO Challenge (TryHackMe)

---

## 1. Reconocimiento

### Escaneo inicial con nmap
```bash
nmap -sC -sV -p- 10.10.X.X
```

- Puerto 22 → OpenSSH 8.4p1  
- Puerto 80 → Apache 2.4.56 (Debian)  
- Puerto 8080 → PHP 8.1.0‑dev (conocido backdoor)  

### Acceso web
```html
<h1>Zerodium</h1>
```

---

## 2. Explotación inicial
- El puerto 8080 corría **PHP 8.1.0‑dev con backdoor**.  
- Usando el exploit público: [php-8.1.0-dev-backdoor-rce](https://github.com/flast101/php-8.1.0-dev-backdoor-rce).  
- Se obtiene una shell como **root**, pero dentro de un **contenedor Docker**:  
  - Hostname: `6ad9beefaa2d`  
  - IP interna: `172.18.0.2`  
  - Confirmación: presencia de `.dockerenv` en `/`.

---

## 3. Enumeración en el contenedor
Revisando `/root/.bash_history` se encuentran las credenciales SSH para el usuario `liam`:

```bash
sshpass -p '[REDACTED]' ssh liam@127.0.0.1
```

- Usuario SSH: **liam**  
- Contraseña: **[REDACTED]**

---

## 4. Acceso al host real
Desde Kali, conexión directa al host con la IP de TryHackMe:

```bash
ssh liam@10.10.X.X
[REDACTED]
```

- Hostname real: `zero`  
- IP: `10.10.X.X`  
- Usuario válido: `liam` (uid=1000, `/home/liam`)

---

## 5. Flag de usuario
En `/home/liam/` se encuentra la flag de usuario:

```bash
cat user.txt
[REDACTED]
```

---

## Resumen de respuestas
- **Hostname (contenedor):** `6ad9beefaa2d`  
- **¿Quién eres en el contenedor?:** `root`  
- **IP del contenedor:** `172.18.0.2`  
- **Fichero oculto:** `.dockerenv`  
- **Usuario SSH:** `liam`  
- **Contraseña:** `[REDACTED]`  
- **Hostname real:** `zero`  
- **Flag de usuario:** `[REDACTED]`