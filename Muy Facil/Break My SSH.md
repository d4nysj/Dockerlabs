# BreakMySSH (DockerLabs) — Writeup

Dificultad: muy fácil

Si el nombre de la máquina ya te avisa de que el SSH está para "romperse", no hace falta mucha imaginación sobre por dónde va a ir el ataque. Esta es probablemente la máquina más minimalista de la colección: un solo puerto abierto, cero enumeración web, cero pistas escondidas en comentarios ni archivos. Todo el reto se reduce a una pregunta: ¿alguien dejó el `root` con la puerta abierta por contraseña? Spoiler: sí.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

Cuando el escaneo de puertos devuelve un único servicio y ninguna otra superficie de ataque, la estrategia se simplifica bastante: no hay comentarios HTML que leer, no hay archivos FTP que descargar, no hay metadatos que exprimir. Solo queda un camino, y es ir directo a por credenciales. Aquí el reto no está en la sofisticación de la técnica, sino en la pura persistencia de un ataque de diccionario bien planteado.

---

## Paso 0: confirmar que hay alguien en casa

```bash
ping -c 3 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.068 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.055 ms
```

TTL 64: sistema Linux. Como en casi todas las máquinas de este estilo, el primer dato ya encaja con lo esperado.

---

## Paso 1: escaneo de puertos (y la sorpresa de lo poco que hay)

```bash
sudo nmap -p- -sS -sC --min-rate 5000 -vvv -Pn -n -oN escaneo.txt 172.17.0.2
```

| Puerto | Estado | Servicio |
|---|---|---|
| 22/tcp | abierto | SSH |

Y ya está. Un único puerto, sin web, sin FTP, sin ningún otro servicio que sirva de apoyo. Esto simplifica muchísimo el análisis: si no hay superficie donde buscar pistas indirectas (comentarios, archivos filtrados, metadatos), el único vector de entrada posible es atacar directamente el propio SSH.

Esta situación cambia bastante el enfoque respecto a otras máquinas del estilo DockerLabs: en vez de dedicar tiempo a enumeración web, todo el esfuerzo se concentra desde el minuto uno en un ataque de credenciales.

---

## Paso 2: fuerza bruta a ciegas, tanto en usuario como en contraseña

Sin ningún nombre de usuario filtrado por ningún lado, tocaba plantear el ataque de la forma más agresiva posible: probar usuarios comunes contra un diccionario de contraseñas extenso.

```bash
hydra -L top-usernames-shortlist.txt \
      -P rockyou.txt \
      ssh://172.17.0.2 -v
```

El ataque combina una lista corta de usuarios habituales (admin, root, test, etc.) contra el diccionario clásico rockyou, probando todas las combinaciones posibles hasta encontrar una válida.

```
[22][ssh] host: 172.17.0.2   login: root   password: estrella
```

Ahí está: `root:estrella`. Y aquí el verdadero problema no es tanto la contraseña en sí (que tampoco es precisamente robusta), sino el hecho de que el servidor **permite login directo como root por contraseña**. Eso normalmente implica que en la configuración de `sshd_config` la directiva `PermitRootLogin` está puesta en `yes`, algo que cualquier guía de hardening de servidores recomienda desactivar sin excepción.

---

## Paso 3: entrar, y ya está — no hay nada más que escalar

```bash
ssh root@172.17.0.2
```

```
root@c67200ea5eef:~# id
uid=0(root) gid=0(root) groups=0(root)

root@c67200ea5eef:~# whoami
root
```

Sin pasar por ningún usuario intermedio, sin ningún `sudo -l` que revisar, sin ningún binario que abusar. La máquina cae en el mismo instante en que se obtiene la contraseña — porque esa contraseña ya es la de root.

---

## Credenciales encontradas

| Usuario | Contraseña | Método |
|---|---|---|
| root | estrella | Hydra (usuarios + rockyou.txt) |

---

## Qué falló aquí (y cómo se evita)

1. **Login de root habilitado por contraseña en SSH.** Esta es, con diferencia, la peor práctica posible en cualquier servidor expuesto: convierte todo el sistema en una única credencial de distancia del compromiso total. La recomendación estándar es desactivar `PermitRootLogin` por completo, o como mínimo restringirlo a autenticación por clave pública (`PermitRootLogin prohibit-password`).
2. **Contraseña de root débil y presente en diccionarios públicos.** Cualquier contraseña que aparezca en rockyou.txt (una recopilación de contraseñas filtradas reales) debe considerarse comprometida de antemano, sin importar lo "original" que parezca a simple vista.
3. **Ausencia de mitigaciones contra fuerza bruta.** No hay rastro de `fail2ban`, límite de intentos, ni ningún mecanismo de bloqueo tras intentos fallidos repetidos. En un entorno real, un ataque de diccionario como este debería quedar frenado mucho antes de completar el diccionario.

---

## Referencias

- OpenSSH — hardening de sshd_config: https://www.ssh.com/academy/ssh/sshd_config
- Hydra (THC): https://github.com/vanhauser-thc/thc-hydra
- DockerLabs: https://dockerlabs.es/
