# Vacaciones (DockerLabs) — Writeup

Dificultad: muy fácil

Esta máquina tiene gracia por lo que representa más que por lo técnico: una demostración perfecta de que compartir una contraseña "solo por si acaso" nunca es tan buena idea como parece en el momento. Juan se fue de vacaciones y decidió dejar su clave apuntada por si Camilo la necesitaba. Spoiler: no hacía falta ser Camilo para leerla.

IP de laboratorio: `172.17.0.2`

---

## Idea general del reto

El objetivo aquí es doble: encontrar credenciales válidas para entrar al sistema, y después escalar privilegios hasta root. Como suele pasar en las máquinas de nivel introductorio, la superficie de ataque es mínima (dos servicios) y la dificultad real está en saber leer entre líneas la información que el propio sistema expone sin querer.

---

## Reconocimiento inicial

Escaneo de puertos completo, con detección de versiones y scripts por defecto:

```bash
sudo nmap -p- -sV -sC -T3 -vvv -Pn 172.17.0.2 -oN escaneo.txt
```

El resultado deja solo dos servicios expuestos:

```text
22/tcp   ssh    OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
80/tcp   http   Apache httpd 2.4.29 (Ubuntu)
```

Con SSH y HTTP como único frente de ataque, la lógica manda empezar por el 80: es el único servicio que permite interacción sin credenciales previas.

---

## La web que no decía nada... a simple vista

Al entrar en `http://172.17.0.2` la página se muestra completamente en blanco. Ni un título, ni un mensaje, ni una imagen rota. Nada.

Pero "nada visible" no es lo mismo que "nada en el código fuente". Interceptando la petición con Burp Suite y revisando el HTML de la respuesta aparece un comentario que el desarrollador olvidó quitar:

```html
<!-- De: Juan Para: Camilo, te he dejado un correo es importante..-->
```

Con esa única línea obtenemos dos datos de golpe:

1. Dos posibles usuarios del sistema: **Juan** y **Camilo**.
2. Una pista muy directa: existe un correo en algún lado con información relevante.

Este es el tipo de comentario que jamás debería llegar a producción — es básicamente el desarrollador dejando una nota post-it pegada en la pantalla, visible para cualquiera que abra las herramientas de desarrollador del navegador.

---

## Enumeración web adicional (que no llevó a nada)

Antes de lanzarse directamente sobre SSH, se hizo un fuzzing de directorios sobre el servidor web para no dejar cabos sueltos:

```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt
```

Se descubrieron varios directorios, pero todos resultaron ser carpetas vacías sin ningún archivo interesante dentro. Un callejón sin salida — a veces la enumeración exhaustiva sirve principalmente para descartar caminos, no para encontrarlos.

---

## Fuerza bruta sobre SSH: primer intento fallido, segundo con premio

Con dos nombres de usuario en la mano, tocaba probar contraseñas contra el servicio SSH:

```bash
hydra -l juan -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Contra el usuario **Juan**, ningún resultado. El diccionario se agota sin encontrar coincidencia.

Se repite el mismo ataque contra **Camilo**:

```bash
hydra -l camilo -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2
```

Esta vez sí: Hydra da con una contraseña válida y se consigue acceso al sistema como Camilo. El primer usuario del comentario HTML no era el eslabón débil — el segundo sí.

---

## Siguiendo la pista del correo

Recordando el comentario HTML ("te he dejado un correo es importante"), lo lógico era revisar si el sistema guarda correos locales, algo típico en instalaciones Unix clásicas mediante el mecanismo de mail spool:

```bash
ls -la /var/mail
```

Dentro aparece una entrada correspondiente al usuario Camilo. Al abrir el archivo asociado (`correo.txt`), el contenido es exactamente lo que el comentario prometía: un mensaje de Juan avisando que se iba de vacaciones y dejando su contraseña por escrito, "por si Camilo la necesitaba mientras tanto".

Con esa contraseña en mano, el acceso a la cuenta de **Juan** queda resuelto sin necesidad de fuerza bruta ni adivinanzas.

```bash
ssh juan@172.17.0.2
```

---

## Escalada a root: Ruby al rescate (del atacante)

Ya autenticado como Juan, el primer comando de cualquier escalada de privilegios:

```bash
sudo -l
```

La salida muestra que Juan puede ejecutar **Ruby como root sin contraseña**. Ruby, como intérprete de propósito general, puede lanzar procesos del sistema operativo directamente desde una línea de código. Si el intérprete corre con privilegios de root, cualquier proceso que genere hereda esos mismos privilegios.

```bash
sudo ruby -e 'exec "/bin/sh"'
```

Ese `exec` reemplaza el proceso de Ruby por una shell del sistema, y como Ruby se ejecutó vía sudo, la shell resultante es directamente root:

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

Sistema totalmente comprometido.

---

## Impacto

Esta máquina combina dos fallos que, por separado, ya serían graves, y juntos garantizan compromiso total del sistema:

- **Filtración de credenciales por correo interno.** Guardar una contraseña en texto plano dentro de un mensaje de correo, aunque sea "solo para un compañero", asume que ese buzón nunca será accedido por nadie más. En este caso, cualquier usuario con acceso de lectura al mail spool podía leerlo.
- **Configuración de sudo permisiva sobre un intérprete de scripting.** Otorgar `NOPASSWD` sobre binarios como Ruby, Python, Perl o similares equivale, en la práctica, a otorgar una shell de root directa, ya que todos ellos permiten ejecutar comandos del sistema operativo de forma nativa.

La lección de fondo es sencilla: ninguna credencial debería viajar en texto plano por ningún canal, y ningún permiso de sudo debería concederse sin comprobar antes si el binario en cuestión aparece en GTFOBins.

---

## Referencias

- GTFOBins — ruby: https://gtfobins.github.io/gtfobins/ruby/
- DockerLabs: https://dockerlabs.es/
