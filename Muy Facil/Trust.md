# Trust (DockerLabs) — Writeup

Dificultad: muy fácil

El nombre de esta máquina es casi una ironía de guion: "Trust" (confianza), y resulta que la propia web te dice sin querer en quién no deberías confiar demasiado con las contraseñas. Aquí el protagonista es Mario, y su gran error no fue reutilizar usuario en todos lados como en otros retos — fue simplemente elegir una contraseña que cualquier diccionario de fuerza bruta decente encuentra en segundos.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El camino de esta máquina es corto pero encadenado: hay que encontrar una ruta oculta en el servidor web que filtra un nombre de usuario, reventar la contraseña de ese usuario por diccionario, y una vez dentro, aprovechar un permiso de sudo mal configurado sobre un editor de texto. La particularidad interesante aquí, frente a otros retos similares, es que la regla de sudo **sí pide contraseña** — pero como ya tenemos la de Mario de un paso anterior, el "problema" prácticamente no existe.

---

## Paso 0: el clásico primer contacto

```bash
ping -c 2 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.043 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.133 ms
```

TTL 64, sistema Linux confirmado. Arrancamos con lo de siempre.

---

## Paso 1: qué hay expuesto

```bash
sudo nmap -p- -sS -sC --min-rate 5000 -n -vvv -Pn 172.17.0.2
```

| Puerto | Estado | Servicio |
|---|---|---|
| 22/tcp | abierto | SSH |
| 80/tcp | abierto | HTTP (Apache2, Debian) |

Dos servicios, y como ya viene siendo costumbre en este tipo de retos: el puerto 80 es la puerta de entrada informativa, y el 22 es el objetivo final del acceso.

---

## Paso 2: la web por defecto no dice nada... hasta que rascas un poco

Al visitar `http://172.17.0.2` solo aparece la página de bienvenida por defecto de Apache2 — la típica plantilla que viene de fábrica y que no debería seguir ahí en ningún servidor que se tome en serio a sí mismo. Como no hay nada visible a simple vista, toca fuzzing de directorios.

```bash
ffuf -u http://172.17.0.2/FUZZ \
  -w DirBuster-2007_directory-list-2.3-small.txt \
  -v -fs 10701 -r -e .php,.html
```

El detalle importante aquí es el flag `-fs 10701`: filtra por tamaño de respuesta para excluir automáticamente todas las páginas que devuelven exactamente el mismo tamaño que la plantilla por defecto de Apache. Sin ese filtro, el fuzzing se llenaría de falsos positivos — cada ruta inexistente respondería con la misma página genérica y sería indistinguible de un hallazgo real.

```
[Status: 200, Size: 927]
  → http://172.17.0.2/secret.php
```

Un archivo con tamaño distinto al resto: claramente algo que sí existe de verdad.

---

## Paso 3: secret.php no guarda muy bien el secreto

Al visitar la ruta, la página muestra el siguiente mensaje:

> "Hola Mario, esta web no se puede hackear."

Una frase con más ironía de la que su autor probablemente pretendía, porque acaba de regalarnos justo lo que necesitábamos: un nombre de usuario, **mario**, filtrado directamente en el contenido visible de una página que se supone "secreta".

---

## Paso 4: fuerza bruta contra SSH con el usuario ya confirmado

```bash
hydra -l mario -P rockyou.txt ssh://172.17.0.2 -v
```

```
[22][ssh] host: 172.17.0.2   login: mario   password: chocolate
```

`chocolate`. Una contraseña con más personalidad que seguridad — y como tantas otras palabras cotidianas, aparece también en rockyou.txt, así que el ataque de diccionario no necesita mucho tiempo para dar con ella.

```bash
ssh mario@172.17.0.2
```

Sesión abierta como usuario de bajo privilegio.

---

## Paso 5: escalada, con un pequeño matiz respecto a otros retos

```bash
sudo -l
```

```
User mario may run the following commands on c528297bca0c:
    (ALL) /usr/bin/vim
```

Aquí conviene fijarse en un detalle que distingue esta máquina de otras similares: la regla **no** lleva `NOPASSWD`. Es decir, sudo va a pedir la contraseña de mario antes de ejecutar vim como root. En la práctica esto apenas supone un obstáculo, porque ya la conseguimos en el paso anterior con Hydra — pero conceptualmente es una configuración algo menos catastrófica que un `NOPASSWD` directo, ya que al menos exige que el atacante tenga la contraseña del usuario en cuestión.

```bash
sudo vim
```

Al introducir la contraseña de mario (`chocolate`), se abre el editor con privilegios de root. Desde el modo normal de vim:

```
:shell
```

Vim, como cualquier editor con capacidad de invocar un intérprete de comandos, abre una shell que hereda los privilegios del proceso padre — en este caso, root.

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
whoami
# root
```

Root conseguido.

---

## Credenciales encontradas

| Usuario | Contraseña | Método |
|---|---|---|
| mario | chocolate | secret.php (usuario) + Hydra (contraseña) |
| root | — | sudo vim → :shell (con contraseña de mario) |

---

## Qué falló aquí (y cómo se evita)

1. **Filtración de nombre de usuario en contenido web visible.** No hacía falta ni un comentario HTML ni metadatos escondidos: el propio texto de la página lo decía sin rodeos. Cualquier información sobre usuarios del sistema, aunque sea a modo de broma o mensaje de bienvenida, no debería aparecer nunca en contenido accesible públicamente.
2. **Contraseña de diccionario para una cuenta con privilegios de sudo.** `chocolate` es una palabra común que cualquier ataque de fuerza bruta con rockyou.txt encuentra sin esfuerzo. El impacto de una contraseña débil se multiplica cuando esa cuenta tiene, además, permisos de sudo sobre binarios peligrosos.
3. **Permiso de sudo sobre un editor con capacidad de shell.** Aunque en este caso la regla exige contraseña (a diferencia de configuraciones NOPASSWD vistas en otras máquinas), el problema de fondo sigue siendo el mismo: cualquier binario listado en GTFOBins con capacidad de abrir una shell, si se concede vía sudo, equivale en la práctica a una vía de escalada a root. La única defensa real es no otorgar ese permiso en absoluto, o restringirlo con wrappers que impidan invocar comandos externos.

---

## Referencias

- GTFOBins — vim: https://gtfobins.github.io/gtfobins/vim/
- SecLists: https://github.com/danielmiessler/SecLists
- THC Hydra: https://github.com/vanhauser-thc/thc-hydra
- DockerLabs: https://dockerlabs.es/
