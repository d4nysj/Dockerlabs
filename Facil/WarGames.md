# Wargame (DockerLabs) — Writeup

Dificultad: fácil

Cualquiera que haya visto "WarGames" (1983) va a reconocer el ambiente de esta máquina al instante: un sistema llamado W.O.P.R., referencias al profesor Falken, un nivel de acceso "CLASSIFIED", y una palabra clave que homenajea directamente una de las frases más recordadas de la película. Aquí el reto no es solo técnico, también es un poco arqueológico: hay que seguir las migas de pan que deja el propio sistema de ficción para encontrar la puerta de entrada.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El recorrido tiene una particularidad interesante: el vector de acceso inicial no es una web ni un FTP con archivos filtrados, sino un servicio TCP personalizado en un puerto poco habitual (5000), que se comporta como una especie de terminal interactiva a la que hay que "convencer" con la palabra correcta. Esa palabra sale de un archivo de texto olvidado en el servidor web, cargado de referencias a la película que da tema a la máquina. Solo después de superar esa fase llega el acceso SSH real, y finalmente una escalada de privilegios centrada en un binario SUID hecho a medida.

---

## Paso 0: desplegar el laboratorio

```bash
unzip wargame.zip
sudo bash auto_deploy.sh wargame.tar
```

```bash
ping -c 1 172.17.0.2
```

Host activo, toca reconocimiento.

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

Cuatro servicios detectados:

- **21/tcp** — FTP
- **22/tcp** — SSH
- **80/tcp** — HTTP
- **5000/tcp** — servicio personalizado

```bash
nmap -sCV -p21,22,80,5000 172.17.0.2
```

La enumeración de versiones y banners revela, entre otras cosas, la existencia de un usuario llamado **joshua** — para quien conozca la película, ese nombre no es casualidad: es literalmente el nombre con el que se dirigían al ordenador W.O.P.R. en la trama original.

El puerto **5000** es el que más llama la atención. Un servicio corriendo en un puerto no estándar casi siempre significa que hay algo hecho a medida esperando ahí, y eso lo convierte en objetivo prioritario frente a los servicios más convencionales.

---

## Paso 2: la web, escasa mensaje, pero con una nota olvidada

```
http://172.17.0.2
```

El sitio responde con contenido bastante limitado — nada de formularios, ni funcionalidades visibles a simple vista. Cuando la superficie visible no da nada, toca fuzzing.

```bash
gobuster dir -u http://172.17.0.2/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .env,.php,.bak,.old,.zip,.txt -b 403,404 --exclude-length 8068
```

Entre los resultados aparece un archivo especialmente jugoso:

```
/README.txt
```

Su contenido está cargado de ambientación (y de pistas):

```
TOP SECRET PROJECT WOPR
ACCESS LEVEL: CLASSIFIED
```

Además de referencias a restricciones administrativas y al personaje de Falken, aparece una palabra clave que destaca sobre el resto del texto:

```
Codename: GODMODE
```

Cualquiera que conozca la trama de la película sabrá exactamente para qué sirve esa palabra — un modo especial de acceso que se menciona explícitamente en la historia original. Aquí, el autor de la máquina la ha convertido en la llave literal de la siguiente fase.

---

## Paso 3: hablando con el servicio del puerto 5000

```bash
nc 172.17.0.2 5000
```

El servicio responde como una interfaz interactiva personalizada — no un banner estático, sino algo que espera entrada del usuario y reacciona de forma distinta según lo que reciba. Se prueban comandos básicos, palabras sueltas del README, variaciones relacionadas con W.O.P.R., sin resultado claro al principio.

Al introducir la palabra clave encontrada previamente:

```
GODMODE
```

El servicio reacciona de forma distinta, y tras cierta interacción adicional, entrega información sensible relacionada con autenticación: unas credenciales protegidas mediante hash.

---

## Paso 4: romper el hash y entrar por SSH

Se prueban varias herramientas y técnicas de crackeo sobre el hash obtenido, hasta dar con un resultado válido usando un servicio de descifrado online especializado en bases de datos de hashes ya resueltos previamente. Con la contraseña recuperada en texto plano:

```bash
ssh joshua@172.17.0.2
```

Acceso conseguido — el mismo usuario cuyo nombre ya habíamos anticipado por el guiño cinematográfico durante el escaneo inicial.

---

## Paso 5: enumeración de binarios SUID

```bash
find / -type f -perm -4000 -ls 2>/dev/null
```

Entre el listado habitual de binarios del sistema (los de siempre: `passwd`, `su`, `mount`, y compañía), aparece uno que no pertenece ahí:

```
-rwsr-xr-x root root /usr/local/bin/godmode
```

Un binario **personalizado**, con el mismo nombre que la palabra clave usada anteriormente, y con el bit SUID activo perteneciendo a root. La coherencia temática de la máquina se mantiene hasta en los detalles de la escalada.

---

## Paso 6: entendiendo qué hace `godmode` antes de atacarlo

```bash
/usr/local/bin/godmode
```

```
W.O.P.R Simulation System v1.0
ACCESS DENIED. DEFCON remains at 5.
```

El binario simula, de forma bastante literal, la interfaz del propio W.O.P.R. de la película, con su sistema de niveles DEFCON. La ejecución directa no basta — hay alguna condición interna que no se está cumpliendo.

Para entender mejor qué está pasando por debajo, se analiza el binario con `strings`:

```bash
strings /usr/local/bin/godmode
```

Aparecen referencias a llamadas de sistema relacionadas con la gestión de privilegios (`setuid`, `setgid`) y a la propia temática W.O.P.R. del programa. La presencia de estas llamadas confirma que el binario está diseñado específicamente para manipular sus propios privilegios efectivos durante la ejecución — exactamente el tipo de comportamiento que interesa explotar cuando un binario tiene SUID de root.

También se observa el comportamiento del binario en tiempo real, incluyendo tráfico HTTP generado durante su ejecución al levantar un servidor local de prueba, lo que sugiere que el binario interactúa con recursos externos o rutas específicas del sistema como parte de su lógica interna.

---

## Paso 7: la escalada final

Combinando la palabra clave del README, el comportamiento observado del binario y las llamadas privilegiadas detectadas en el análisis estático, se consigue satisfacer las condiciones internas que el binario `godmode` espera para completar su ejecución con privilegios elevados en vez de rechazar el acceso.

```bash
whoami
```

```
root
```

Escalada completada — el propio nombre del binario resulta ser, en última instancia, una descripción bastante honesta de lo que hace: activar el "modo dios" sobre el sistema para quien sepa cómo pedírselo correctamente.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Servicios detectados | FTP (21), SSH (22), HTTP (80), servicio personalizado (5000) |
| Pista temática | `/README.txt` con referencias a WarGames y codename GODMODE |
| Vector de acceso | Interacción con servicio TCP personalizado usando la palabra clave |
| Credenciales | Hash filtrado por el servicio, roto vía plataforma de descifrado online |
| Usuario obtenido | joshua, vía SSH |
| Vector de escalada | Binario SUID personalizado `/usr/local/bin/godmode` |
| Resultado final | root, tras satisfacer las condiciones internas del binario |

---

## Qué falló aquí (y cómo se evita)

1. **Archivo de notas internas expuesto en la raíz web.** Un `README.txt` con información de contexto, códigos clave y referencias operativas nunca debería estar accesible desde un servidor web público, por muy tentador que sea dejarlo ahí "solo para pruebas".
2. **Servicio personalizado sin autenticación robusta.** Un servicio TCP a medida que reacciona ante una única palabra clave estática, sin ningún mecanismo real de autenticación (usuario, token, límite de intentos), es trivialmente vulnerable en cuanto esa palabra se filtra por cualquier otro canal.
3. **Credenciales protegidas únicamente por un hash débil o ya catalogado en bases de datos públicas.** Si un hash puede resolverse consultando un servicio online de hashes ya crackeados previamente, en la práctica ya no está protegiendo nada — equivale a texto plano con pasos extra.
4. **Binario SUID personalizado con lógica de privilegios manipulable.** Cualquier binario que gestione sus propios permisos mediante `setuid`/`setgid` de forma personalizada introduce una superficie de ataque mucho mayor que confiar en los mecanismos estándar del sistema operativo. Auditar cuidadosamente cualquier binario SUID no estándar antes de desplegarlo en producción es imprescindible — y, siempre que sea posible, evitar por completo scripts o binarios propios con SUID.

---

## Referencias

- GTFOBins: https://gtfobins.github.io/
- DockerLabs: https://dockerlabs.es/
