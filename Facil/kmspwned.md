# Kmspwned (DockerLabs) — Writeup

Dificultad: fácil

El nombre ya avisa de por dónde van a ir los tiros: una web que se presenta como herramienta de comprobación de dominios, con una base de datos detrás que no desinfecta una sola entrada de usuario. Es el ejemplo de manual de inyección SQL clásica — y precisamente por eso es una máquina estupenda para quien todavía no ha automatizado del todo el proceso de "columna por columna, tabla por tabla" y quiere hacerlo a mano para entenderlo de verdad.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El recorrido tiene dos fases muy diferenciadas. La primera es web pura: registro de usuario, un formulario que verifica extensiones de dominio, y una inyección SQL de manual que se puede llevar a mano, paso a paso, hasta sacar las credenciales de la base de datos. La segunda fase cambia completamente de terreno: una vez dentro por SSH, la escalada de privilegios no depende de ningún binario exótico ni de un CVE con nombre propio, sino de algo mucho más común en sistemas reales mal configurados — un script de backup que corre como root cada minuto y que el usuario normal puede modificar a su antojo.

---

## Paso 0: desplegar el laboratorio

```bash
unzip kmspwned.zip
sudo bash auto_deploy.sh kmspwned.tar
```

```bash
ping -c 1 172.17.0.2
```

Host arriba, toca reconocimiento.

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

Dos servicios a la vista:

- **22/tcp** — SSH
- **80/tcp** — HTTP

```bash
nmap -sCV -p22,80 172.17.0.2
```

Nada fuera de lo normal en los banners. Con solo dos puertos abiertos y uno de ellos siendo la web, el camino de entrada está cantado: hay que mirar qué hace esa aplicación.

---

## Paso 2: la web — un verificador de dominios con cuenta de usuario

```
http://172.17.0.2
```

La aplicación pide registrar una cuenta antes de dejarte usar nada, así que el primer paso es crear un usuario cualquiera e iniciar sesión. Una vez dentro aparece la funcionalidad principal: un "verificador de extensiones de dominio" — le escribes un nombre y te dice si esa extensión está disponible o no. Es exactamente el tipo de formulario que suele construirse con una consulta SQL directa por detrás, sin pensar en sanitizar la entrada.

---

## Paso 3: confirmando la inyección SQL

Lo primero, lo de siempre: meter una comilla simple (`'`) en el campo del verificador y ver qué pasa.

```
'
```

La aplicación responde con un error de base de datos. Eso confirma que el parámetro llega sin filtrar a una consulta SQL. El siguiente paso es comprobar si se puede manipular la lógica de la consulta:

```sql
' OR '1'='1'-- -
```

Sin error esta vez, y el comportamiento cambia — la inyección es válida y se puede empezar a construir sobre ella con `UNION SELECT`.

---

## Paso 4: averiguando el número de columnas

```sql
' UNION SELECT NULL,NULL,NULL-- -
```

Con tres `NULL` la consulta no da error; añadiendo un cuarto sí lo da. Conclusión: la consulta original selecciona **3 columnas**. A partir de aquí ya se puede usar `UNION SELECT` para sacar literalmente cualquier cosa que la base de datos quiera contarnos.

---

## Paso 5: tablas, columnas y la cuenta que importa

Primero, qué tablas existen:

```sql
' UNION SELECT NULL,NULL,table_name FROM information_schema.tables-- -
```

Entre el listado aparece una que llama la atención por su nombre: `sc_usuarios`. Ahí tiene que estar lo interesante.

```sql
' UNION SELECT NULL,NULL,column_name FROM information_schema.columns WHERE table_name='sc_usuarios'-- -
```

Columnas `id`, `usuario` y `password` — justo lo que hace falta para una fuga de credenciales completa:

```sql
' UNION SELECT id,usuario,password FROM sc_usuarios-- -
```

La consulta devuelve la tabla entera. Entre los usuarios hay uno que destaca: **carlos**, con una contraseña guardada como hash MD5.

---

## Paso 6: de hash a shell

El hash de `carlos` cae sin complicaciones en cualquier servicio de lookup de MD5 (son hashes sin sal, así que si el valor ya está indexado en alguna base pública, se resuelve al momento). La contraseña en claro resulta ser `password1` — un recordatorio de que ni la mejor inyección SQL del mundo sirve de mucho si detrás hay contraseñas de diccionario.

```bash
ssh carlos@172.17.0.2
```

Dentro.

---

## Paso 7: escalada de privilegios — el cron que te regala root

Una vez con shell, lo primero es mirar qué tareas programadas hay corriendo en el sistema:

```bash
cat /etc/crontab
```

Ahí aparece un script, `/opt/backup.sh`, ejecutándose **cada minuto como root**. El reflejo inmediato cuando se ve algo así es comprobar los permisos:

```bash
ls -la /opt/
```

Y ahí está el fallo: `carlos` tiene permiso de **lectura, escritura y ejecución** sobre ese script. Si algo que corre como root se puede modificar como usuario normal, la máquina ya es tuya — solo hay que decidir qué hacer con ese minuto de margen.

```bash
nano /opt/backup.sh
```

Se añade una línea sencilla:

```bash
chmod u+s /bin/bash
```

Se guarda, y se espera a que el cron dispare la siguiente ejecución (como mucho, 60 segundos). Pasado ese minuto:

```bash
ls -la /bin/bash
bash -p
```

El bit SUID ya está puesto sobre `/bin/bash`, así que `bash -p` arranca una shell que conserva privilegios de root en lugar de soltarlos. Root conseguido.

---

## Conclusión

Dos lecciones muy distintas en una sola máquina fácil. La primera, de toda la vida: cualquier campo que acepte texto libre y lo meta en una consulta SQL sin parametrizar es una puerta abierta, y con paciencia (columnas, tablas, columnas de la tabla, datos) se puede vaciar una base de datos entera a mano sin herramientas automáticas. La segunda es la que más se ve en sistemas reales mal configurados fuera de los laboratorios: un cron que corre como root ejecutando un script que un usuario sin privilegios puede escribir es, en la práctica, una invitación a convertirse en root cuando quieras. Vale la pena acostumbrarse a mirar `/etc/crontab` y los permisos de lo que hay ahí dentro nada más conseguir una shell — es de los primeros sitios donde merece la pena mirar.
