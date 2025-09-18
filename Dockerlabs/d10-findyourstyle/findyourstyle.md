# Find Your Style
## Reconocimiento
### Nmap
En primer lugar, escaneo con `nmap` los puertos abiertos buscando información que pueda ayudar a vulnerar la máquina
```bash
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![Nmap](./imgs/nmap.png)

Solo hay un puerto y se trata de un servidor HTTP hecho en Drupal 8, un CMS escrito en PHP, la página contiene un `robots.txt` que deshabilita 22 entradas. Con `whatweb` saco más información sobre la página, efectivamente se trata de una web hecha en Drupal, así que lo siguiente va a ser buscar en metasploit.

![Whatweb](./imgs/whatweb.png)

## Explotación
Con `msfdb run` accedo a la consola y veo que el exploit 0 podría ser buena opción, así que lanzo `use 0` para poder rellenar la información necesaria para correr el exploit.

![MSFDB](./imgs/msfdb.png)

Con `set [OPCIÓN] [VALOR]` voy fijando los datos que el exploit requiere para correr y una vez todo está en orden lanzo `run`.

![Options](./imgs/options.png)

En unos segundos ya estoy dentro de la máquina, al hacer un `cat /etc/passwd` me fijo en el user `ballenita`, al que debo tratar de pivotar. Navego a `/var/www/html` y hago uso de la función `upload` de meterpreter para subir una reverse shell, a la cual hago una solicitud desde el navegador visitando la URL `http://IP/shell.php`, me pongo en escucha y me doy una shell que pueda sanitizar.

## Post explotación
Lo primero como siempre es sanitizar la shell. En primer lugar lo hice con 
```
export SHELL=bash
export TERM=xterm
export TERM=xterm-256color
source /etc/skel/.bashrc
```
Pero me dió problemas en un paso posterior, así que utilicé este método
```
script /dev/null -c bash
## Aquí presionamos CTRL+Z
stty raw -echo; fg
>reset
>xterm
export TERM=xterm
export SHELL=bash
```
Después pasé un rato investigando hasta llegar a un archivo en `/var/www/html/sites/default` que contiene un archivo de configuración en el cual había credenciales del usuario ballenita, las cuales sirvieron para cambiar con `su ballenita`. 

Para esta parte tuve que apoyarme un poco en el [writeup de d1se0](https://dise0.gitbook.io/h4cker_b00k/ctf/dockerlabs/findyoustyle-dockerlabs-easy), ya que no estaba seguro de como usar los binarios en los que el usuario ballenita tiene permisos SUDO, aunque es bastante evidente.
La idea es hacer un `sudo ls` de la raíz, ahí hay un archivo con la passwd que podremos leer gracias a `grep`, una vez tenemos esa credencial, solo queda 