# Borazuwarah (DockerLabs) — Writeup

Dificultad: muy fácil

Esta máquina es un ejemplo perfecto de por qué la esteganografía y los metadatos de imagen dan para escribir un libro entero de "cosas que la gente olvida borrar". Aquí ni siquiera hace falta abrir un editor hexadecimal: basta con preguntarle educadamente a la imagen quién es su dueño.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto se resuelve en tres actos bien diferenciados: encontrar un usuario escondido en un sitio poco habitual (los metadatos de una imagen), fuerza bruta contra SSH para dar con la contraseña, y una escalada de privilegios que ni siquiera necesita clave. Nada de vulnerabilidades exóticas — puro ejercicio de observación y paciencia.

---

## Paso 0: confirmar que el objetivo está en pie

```bash
ping -c 2 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.070 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.068 ms
```

TTL 64, así que estamos ante un sistema Linux. Rutina de cualquier reconocimiento.

---

## Paso 1: qué hay abierto en la máquina

```bash
sudo nmap -p- -sS -sCV --min-rate 5000 -n -Pn -vvv -oN escaneo.txt 172.17.0.2
```

| Puerto | Estado | Servicio | Versión |
|---|---|---|---|
| 22/tcp | abierto | SSH | OpenSSH 9.2p1 (Debian) |
| 80/tcp | abierto | Apache | httpd 2.4.59 (Debian) |

Otra vez la fórmula clásica de las máquinas fáciles: solo dos servicios, y uno de ellos (el web) sirviendo de vía de entrada para la información inicial.

---

## Paso 2: una web que solo enseña una imagen

```bash
curl -s http://172.17.0.2
```

```html
<html><body><img src='imagen.jpeg'></body></html>
```

Todo el contenido del sitio se reduce a una etiqueta `<img>`. Nada de texto, nada de formularios, nada de comentarios jugosos esta vez. La pista, si la hay, tiene que estar dentro del propio archivo de imagen.

```bash
wget http://172.17.0.2/imagen.jpeg
```

---

## Paso 3: preguntarle a la imagen quién es (spoiler: responde)

Antes de asumir que hay que hacer esteganografía avanzada con `steghide` o buscar bytes escondidos al final del archivo, conviene empezar por lo más simple: revisar los metadatos EXIF. Muchas veces ahí está la respuesta, sin necesidad de descifrar nada.

```bash
exiftool imagen.jpeg
```

```
Description  : ---------- User: borazuwarah ----------
Title        : ---------- Password:  ----------
```

El campo `Description` entrega directamente un nombre de usuario: **borazuwarah**. El campo `Title`, en cambio, deja el espacio de la contraseña vacío — es casi una broma del autor del laboratorio, como diciendo "toma el usuario, la contraseña te la curras tú". Confirmado: hay que ir a por fuerza bruta.

---

## Paso 4: fuerza bruta contra SSH con el usuario ya en mano

```bash
hydra -l borazuwarah -P rockyou.txt -v -t 64 ssh://172.17.0.2 -o hydra_result.txt
```

```
[22][ssh] host: 172.17.0.2   login: borazuwarah   password: 123456
```

`123456`. Probablemente la contraseña más reciclada de la historia de internet, y aquí cumple su papel una vez más. Con el usuario ya confirmado por los metadatos, el ataque de diccionario tarda lo justo.

```bash
ssh borazuwarah@172.17.0.2
```

Sesión abierta, toca ver qué privilegios tenemos.

---

## Paso 5: escalar sin ni siquiera necesitar la contraseña de nuevo

```bash
sudo -l
```

```
User borazuwarah may run the following commands on 6e8dc307494a:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/bash
```

Aquí hay dos líneas interesantes, y merece la pena distinguirlas bien:

- La primera, `(ALL : ALL) ALL`, permite ejecutar cualquier comando como root — pero pidiendo contraseña, así que no es el camino más rápido.
- La segunda, `NOPASSWD: /bin/bash`, permite abrir una shell de bash como root **sin pedir absolutamente nada**. Es la vía de menor resistencia y, de hecho, la más directa posible: ni siquiera hace falta encontrar un binario intermedio que abra una shell — el propio sudo ya nos regala una.

```bash
sudo /bin/bash
```

```
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
```

Root conseguido sin escribir una sola contraseña más.

---

## Credenciales encontradas

| Usuario | Contraseña | Método |
|---|---|---|
| borazuwarah | 123456 | Metadatos EXIF + Hydra + rockyou |
| root | — | sudo NOPASSWD sobre /bin/bash |

---

## Qué falló aquí (y cómo se evita)

1. **Metadatos EXIF sin limpiar antes de publicar la imagen.** Cualquier información añadida "para uso interno" en los metadatos de un archivo público (usuario, ruta, autor, incluso coordenadas GPS en fotos reales) queda accesible con una simple herramienta de línea de comandos. Antes de subir una imagen a un servidor público, siempre conviene limpiar sus metadatos.
2. **Contraseñas débiles y reutilizadas.** `123456` sigue apareciendo en prácticamente todas las filtraciones de contraseñas conocidas. Ningún usuario, ni siquiera en un entorno de pruebas, debería tener una contraseña de diccionario tan directa.
3. **Regla de sudo con NOPASSWD sobre una shell completa.** Conceder `NOPASSWD` sobre `/bin/bash` es, literalmente, entregar una llave maestra de root sin ninguna fricción. Cualquier regla de sudo debería revisarse binario por binario contra GTFOBins antes de aplicarla, y evitar por completo el NOPASSWD sobre shells o intérpretes.

---

## Referencias

- ExifTool: https://exiftool.org/
- Hydra (THC): https://github.com/vanhauser-thc/thc-hydra
- GTFOBins — bash: https://gtfobins.github.io/gtfobins/bash/
- DockerLabs: https://dockerlabs.es/
