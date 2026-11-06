# Acme (DockerLabs) — Writeup

Dificultad: fácil

Esta máquina es de esas que te recuerdan por qué no puedes dejar ni un solo rastro de información sensible en ningún sitio. Lo que parece un despliegue estándar de Docker se convierte en un patio de recreo cuando te das cuenta de que el sistema te lo pone en bandeja desde el primer segundo. Es una buena forma de entender que, a veces, la seguridad no se rompe, simplemente se regala.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El ataque es bastante directo. Primero escaneamos para ver qué servicios están escuchando y luego nos aprovechamos de un banner que escupe las credenciales al intentar conectar por SSH. Una vez dentro como usuario normal, buscaremos un binario con permisos SUID que nos permita saltar a root sin despeinarnos.

---

## Paso 0: desplegar el laboratorio

```bash
unzip acme.zip
sudo bash auto_deploy.sh acme.tar
```

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

El escaneo nos devuelve dos puertos abiertos: el 22 para SSH y el 80 para el servidor web. Lo normal aquí sería ponerse a revisar la web, pero en este caso, el puerto 22 tiene un detalle que nos ahorra media hora de trabajo.

---

## Paso 2: acceso inicial

Como vi que el SSH estaba abierto, mi primera reacción fue intentar conectar. A veces, las configuraciones por defecto de los servicios o los mensajes de bienvenida del servidor son una mina de oro. En cuanto intentas establecer conexión, el banner del sistema suelta el usuario y la contraseña casi como si estuviera pidiendo a gritos que entres.

```bash
ssh usuario@172.17.0.2
```

Tras introducir la contraseña filtrada en el banner, ya estamos dentro del sistema. Lo primero que hice fue comprobar mis privilegios con sudo -l, pero como era de esperar, no tenía permisos.

---

## Paso 3: escalada de privilegios

Cuando un usuario normal no puede hacer gran cosa, lo siguiente en mi lista de tareas es buscar binarios con el bit SUID activo. Estos archivos se ejecutan con los permisos del propietario, que en muchos casos es root. Ejecuté un comando rápido para buscarlos por todo el sistema.

```bash
find / -perm -4000 2>/dev/null
```

Al ver /bin/bash en la lista, supe que la partida estaba ganada. Tener Bash con SUID es básicamente tener las llaves del reino. Ejecuté el binario con el flag -p para mantener los privilegios efectivos y automáticamente me encontré con la consola de root.

```bash
/bin/bash -p
```

---

## Conclusión

Esta máquina es un recordatorio de que los errores básicos de configuración son los que más duelen. Filtrar credenciales en un banner de SSH es una negligencia que te deja expuesto en menos de lo que tardas en abrir la terminal, y dejar un binario tan potente como Bash con el bit SUID activo es, básicamente, dejar la puerta blindada abierta de par en par.

Al final, la seguridad es sumar capas. Si fallas en lo básico, como el control de acceso o los permisos de tus binarios, poco importa lo demás. Es una lección rápida pero efectiva sobre cómo una mala configuración te puede costar el control total de un sistema.

---

Basado en el writeup de osfz15: https://github.com/osfz15/Dockerlab-acme/blob/main/README.md
