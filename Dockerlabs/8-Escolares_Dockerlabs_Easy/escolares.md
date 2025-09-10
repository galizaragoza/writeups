# Info
```
172.17.0.2
```
# Writeup
### Reconocimiento
Comienzo por escanear con nmap los puertos abiertos para ver que vectores de ataque podemos aprovechar para vulnerar la máquina.
```shell
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![Nmap](./imgs/nmap.png)
	
A parte del SSH que hay en el puerto 22, no hay otra cosa que un servidor HTTP expuesto por el puerto 80, así que voy a echarle un vistazo al contenido de la página.
La página tiene varios enlaces, pero es en su mayoría estática, es la web de una universidad de ciberseguridad, y tiene una serie de rutas como `/alumnado`, `/escolares` o `/carreras`, como digo todas ellas estáticas. La sección de contacto si tiene un formulario con varios inputs de texto que podría ser un vector.
Inspeccionando el código doy con un comentario que revela una ruta con información relevante.

![Comentario](./imgs/comentario.png)

![Profesores](./imgs/profesores.png)

El último que se ve en la imagen es el que me interesa, ya que como se puede ver es el admin de wordpress. Al ir a `/wordpress` me  encuentro con una página rota y sin ningún panel de login ni enlace funcional, después de un fuzzeo dentro de `/wordpress` llego a `/wp-login.php`.
Usando `wpscan` enumero los usuarios que hay en la página.
```shell
❯ wpscan --url http://172.17.0.2/wordpress/ -e u

...

[+] luisillo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
```
### Explotación
Después de perder un poco el tiempo me doy cuenta de que no me está haciendo las redirecciones y de que no funcionaban algunos enlaces porque no había configurado `/etc/hosts` como requiere este laboratorio, así que lo corrijo añadiendo a este archivo la ip y al lado el dominio que encontramos en el código fuente y al que apuntan las redirecciones: `escolares.dl`.
Para ir cerrando frentes abiertos pruebo con el formulario de contacto pero mirando el código fuente veo que es atrezzo.

Creo que la información de la matrícula y fecha de nacimiento encontrada en `/profesores.html` puede ser útil, y lo lógico ahora sería confeccionar un diccionario custom para atacar de nuevo con `wpscan`, así que tras investigar un poco doy con un repositorio llamado [CUPP](https://github.com/Mebus/cupp/tree/master#), una herramienta escrita en python que genera archivos con combinaciones de datos como nombre, cumpleaños... Basadas en la información recopilada sobre un objetivo.
Le paso el nombre, nick, fecha de nacimiento y matrícula, y me genera un archivo de texto con el que atacar de nuevo.
```
❯ wpscan --url http://escolares.dl/wordpress/wp-login.php -U luisillo -P luis.txt
```
Esta vez conseguimos dar con la clave generada por el creador de diccionarios.

![Luis pass](./imgs/luis-pass.png)

Una vez dentro del panel de administrador de WordPress, simplemente voy al plugin File Manager, creo un archivo llamado `shell.php` y voy a revshells.com para utilizar la de PHP de Pentest Monkey.
Con esto hecho, solo hace falta ponerse en escucha con `nc -nvlp 443` y entonces hacer una solicitud a `escolares.dl/wordpress/shell.php` y ya estamos dentro como `www-data`.

![www-data](./imgs/www-data.png)

### Post explotación
Lo primero una vez hemos ganado acceso es sanitizar la shell.

![Sanitizar](./imgs/sanitizar.png)

Nada más empezar a buscar doy con un archivo en `/home` llamado secret.txt, le hago un `cat` y revela una contraseña secreta, podría servir para un login de phpmyadmin que encontré antes en un fuzzeo o quizá para pivotar a otro usuario.

![Secret pass](./imgs/secretpass.png)

Efectivamente, la clave sirve para pivotar al usuario luisillo, puesto que la máquina tiene ssh voy a utilizar las credenciales para conectarme por ahí.
Una vez autenticado como `luisillo`, pruebo en primer lugar a lanzar un `sudo -l`, y resulta que puedo ejecutar el binario `awk` como root, así que después de una búsqueda rápida en GTFObins el laboratorio queda resuelto.

![[Ciberseguridad/CTFs/253_septiembre/9-escolares/imgs/root.png]]