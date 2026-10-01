# HedgeHog (DockerLabs) — Writeup

Dificultad: muy fácil

Las máquinas "muy fácil" tienen fama de ser un calentamiento, y esta no rompe la norma — pero tiene una peculiaridad que vale la pena comentar: aquí no hay ninguna vulnerabilidad web que explotar ni ningún CVE de turno. El camino de entrada es, directamente, adivinar una contraseña. Un recordatorio de que a veces la seguridad no falla por un fallo de código, sino por una política de contraseñas inexistente.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Dos servicios, dos fases. Primero SSH, resuelto por fuerza bruta una vez se consigue un nombre de usuario válido desde la web. Después, una escalada de privilegios que tampoco necesita ningún exploit: basta con mirar qué puede ejecutar el usuario con `sudo` para encontrar el camino directo a root.

---

## Paso 0: desplegar el laboratorio

```bash
unzip hedgehog.zip
sudo bash auto_deploy.sh hedgehog.tar
```

```bash
ping -c 1 172.17.0.2
```

---

## Paso 1: escaneo de puertos

```bash
nmap -sCV -p- --open --min-rate 5000 -n -vvv -Pn 172.17.0.2 -oG Escaneo
```

Dos servicios, nada exótico:

- **22/tcp** — SSH (OpenSSH 9.6p1)
- **80/tcp** — Apache

Versiones recientes, sin CVEs públicos jugosos para ninguna de las dos. Cuando el escaneo no da pistas de vulnerabilidades conocidas, toca mirar qué cuenta esa web por sí misma.

---

## Paso 2: la web — un nombre de usuario suelto

```
http://172.17.0.2
```

Enumerando el contenido de la web aparece una referencia a un usuario: **tails**. No hay nada más explotable a la vista — ni formularios, ni subida de archivos, ni parámetros sospechosos. Con solo SSH y un nombre de usuario en la mano, el camino está bastante cantado: fuerza bruta de contraseña.

---

## Paso 3: fuerza bruta contra SSH

El diccionario de siempre, `rockyou.txt`, tiene unos 14 millones de líneas — demasiado para un ataque lento, así que antes de lanzarlo conviene optimizarlo un poco. Dos trucos sencillos: invertir el orden del fichero (muchas contraseñas típicas de laboratorio aparecen hacia el final del rockyou original) y quitar los espacios en blanco sueltos que no aportan nada como contraseña real:

```bash
tac rockyou.txt >> modificado.txt
sed -i 's/ //g' modificado.txt
```

Con el diccionario ya más manejable, Hydra entra en juego:

```bash
hydra -l tails -P modificado.txt ssh://172.17.0.2/ -t 5
```

Pasado un rato, Hydra encuentra la contraseña válida para `tails`. Acceso por SSH conseguido.

```bash
ssh tails@172.17.0.2
```

---

## Paso 4: escalada de privilegios — sudo sin contraseña hacia otro usuario

Dentro, el primer movimiento de siempre:

```bash
sudo -l
```

Y ahí está la clave de la máquina: `tails` puede ejecutar comandos como el usuario **sonic**, sin que se le pida contraseña. No hace falta ningún binario raro ni ningún GTFOBins complicado — directamente:

```bash
sudo -u sonic -i
```

Una vez dentro de la sesión de `sonic`, se repite la misma comprobación por si hay otro salto disponible — y lo hay, esta vez directo a root:

```bash
sudo -u root /bin/bash
```

Root conseguido, sin exploits, solo siguiendo la cadena de permisos de `sudo` que la máquina iba dejando a la vista.

---

## Conclusión

HedgeHog es el recordatorio perfecto de que no todas las intrusiones empiezan con un exploit. Aquí todo el camino es higiene básica de seguridad mal aplicada: una contraseña débil que cae ante un diccionario optimizado, y una configuración de `sudo` que encadena privilegios de un usuario a otro sin ningún control. En sistemas reales, esto se traduce en una regla muy simple de aplicar: revisa siempre qué puede ejecutar cada usuario con `sudo -l`, porque muchas veces el camino a root ya está escrito ahí, solo hay que leerlo.
