# Obsession (DockerLabs) — Writeup

Máquina de dificultad muy fácil sobre Ubuntu Linux. El nombre le viene al pelo: el protagonista de este pwn es un usuario obsesionado con una chica, y esa obsesión es literalmente la pista que nos lleva hasta él.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Antes de lanzar ninguna herramienta, conviene tener claro el objetivo: encontrar un usuario válido, conseguir credenciales, entrar por SSH y escalar a root. En máquinas "muy fácil" normalmente hay una cadena de 3-4 fallos encadenados, ninguno especialmente sofisticado por separado. Aquí el patrón se repite: cada servicio filtra un poquito de información, y juntando las piezas se llega al final.

---

## Paso 0: comprobar que el bicho respira

```bash
ping -c 2 172.17.0.2
```

Respuesta con TTL 64, señal típica de un host Linux (los Windows suelen responder con 128). Nada nuevo bajo el sol, pero es el primer check de cualquier auditoría.

---

## Paso 1: mapear servicios

Escaneo completo de puertos, sin resolución DNS y sin ping previo (ya sabemos que responde):

```bash
sudo nmap -p- -sS -sC -sV --min-rate 5000 -n -Pn -vvv -oN nmap_full.txt 172.17.0.2
```

Resultado:

- **21/tcp** — vsftpd 3.0.5
- **22/tcp** — OpenSSH 9.6p1 (Ubuntu)
- **80/tcp** — Apache 2.4.58 (Ubuntu)

Tres servicios, superficie de ataque pequeña. Lo primero que llama la atención en el output de Nmap con `-sC` es que el FTP **admite login anónimo**. Ese suele ser el hilo del que empezar a tirar.

---

## Paso 2: tirar del FTP anónimo

```bash
ftp 172.17.0.2
```

Usuario: `anonymous`, contraseña vacía (solo pulsar Enter). Dentro hay dos archivos de texto descargables con `get`.

El primero es, literalmente, la captura de una conversación de chat entre dos personas. Uno de ellos, identificado como **russoski**, presume de haber grabado un vídeo a una chica llamada Nágore y menciona que lo tiene guardado "en una ruta segura" de su ordenador. Con esto ya tenemos un nombre de usuario candidato.

El segundo archivo es una lista de tareas pendientes del propio russoski. Entre ellas hay una que destaca sobre las demás: menciona que tiene "ciertos permisos habilitados que no son del todo seguros" en su equipo y que debería revisarlos. Aquí el autor de la máquina nos está regalando, sin quererlo el propio personaje, la pista de la escalada de privilegios. Conviene apuntarlo mentalmente para más adelante.

---

## Paso 3: mirar qué esconde el servidor web

```bash
curl -s http://172.17.0.2 | grep -i "<!--"
```

En el código fuente aparece un comentario del desarrollador confesando que reutiliza el mismo usuario en todos sus servicios "para no olvidarlo". Es la confirmación que necesitábamos: el nombre visto en el chat de FTP es también el usuario de SSH.

Ahora toca buscar rutas ocultas:

```bash
ffuf -w SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-small.txt \
     -u http://172.17.0.2/FUZZ -recursion -ac -v
```

Aparecen dos directorios con código 200: `/important` y `/backup`.

- `/important` contiene un archivo de texto con el manifiesto clásico de "La conciencia de un hacker". Curioso, pero irrelevante para el ataque.
- `/backup` contiene un archivo `backup.txt` con una única línea: el nombre de usuario en texto plano, junto con una nota del propio administrador diciendo que "hay que cambiarlo pronto". Spoiler: no lo cambió a tiempo.

Con esto el usuario queda confirmado por partida triple (chat, comentario HTML y backup).

---

## Paso 4: fuerza bruta contra SSH

Con usuario confirmado, toca probar contraseñas. Diccionario de cabecera de cualquier CTF:

```bash
hydra -l russoski -P rockyou.txt -V ssh://172.17.0.2
```

Hydra encuentra la contraseña en cuestión de minutos: `iloveme`. Dado el contexto del chat sobre la chica, la contraseña no es precisamente una sorpresa — más bien un chiste que se escribe solo.

```bash
ssh russoski@172.17.0.2
```

Sesión abierta como usuario de bajo privilegio.

---

## Paso 5: escalar a root

Primer comando de rigor tras entrar en cualquier máquina:

```bash
sudo -l
```

La salida confirma exactamente lo que la lista de tareas del FTP anticipaba: el usuario puede ejecutar `/usr/bin/vim` como root sin necesidad de contraseña (`NOPASSWD`).

Vim, como muchos editores e intérpretes, permite abrir una shell del sistema desde dentro de su propio entorno. Si el proceso que lo lanza tiene privilegios de root, la shell resultante también los tiene. Es una técnica catalogada en GTFOBins, el repositorio de referencia para este tipo de "escapes" desde binarios legítimos.

```bash
sudo vim
```

Una vez dentro del editor, en modo normal:

```
:shell
```

Y con eso, una shell de root sin pedir ni una contraseña:

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

Máquina resuelta.

---

## El detalle final: el vídeo

Como colofón anecdótico, una vez con acceso root merece la pena revisar el directorio home del propio root. Ahí aparece un archivo con el nombre de la chica mencionada en el chat inicial, conteniendo un enlace de YouTube y un comentario emocionado del usuario sobre haber "terminado" el vídeo. La "ruta segura" de la que hablaba al principio no era más que el home de root — la ironía de creer que algo está protegido solo porque está "escondido" en el sitio equivocado.

---

## Credenciales encontradas

| Usuario | Contraseña | Origen |
|---|---|---|
| russoski | iloveme | Hydra contra SSH |
| root | — | Escape de sudo vim |

---

## Qué falló aquí (y cómo se evita)

1. **FTP con acceso anónimo habilitado.** Si un servicio no necesita estar abierto al público, no debería estarlo. Y si necesita estarlo, desde luego no debería tener archivos con nombres de usuario ni información personal dentro.
2. **Comentarios de depuración olvidados en el HTML de producción.** Cualquier información operativa (usuarios, rutas, credenciales, arquitectura interna) no debe viajar en el código fuente que ve el cliente.
3. **Reutilización del mismo usuario en todos los servicios.** Convierte cualquier fuga aislada en un mapa completo del sistema.
4. **Directorio de backups accesible sin autenticación.** Un backup expuesto en la web equivale a regalar las llaves de casa.
5. **Regla de sudo demasiado permisiva sobre un binario con capacidad de shell.** Nunca debe concederse `NOPASSWD` sobre editores, intérpretes o cualquier binario listado en GTFOBins sin restringir explícitamente esa capacidad.

---

## Referencias

- GTFOBins — vim: https://gtfobins.github.io/gtfobins/vim/
- DockerLabs: https://dockerlabs.es/
