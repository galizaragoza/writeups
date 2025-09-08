# Info
## IP Víctima
```
172.17.0.2
```
## Sacado del dict a Basic Auth
```

```
```

```
# Writeup
## Reconocimiento
### Nmap
En primer lugar lanzo un escaneo de puertos para identificar los servicios expuestos y empezar a ver por dónde se puede vulnerar esta máquina
```zsh
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![nmap scan](./nmap.png)

### Tomcat 9.0.88
El resultado muestra únicamente un servidor HTTP en el puerto 8080, lo cual reduce la superficie de ataque al contenido que haya en la propia página, así que a ver que contiene.
Al visitar la página veo una confirmación de instalación de Apache Tomcat versión 9.0.88 (como se ve puede ver en el escaneo),  veo un panel de login de administrador pero antes de probar nada investigo un poco sobre la versión de Tomcat, y rápidamente me topo con el [CVE-2025-24813](https://nvd.nist.gov/vuln/detail/CVE-2025-24813), una vulnerabilidad que afecta a varias versiones de Apache Tomcat, incluida la que tengo delante.
La vulnerabilidad permite lectura y escritura de archivos dentro de la máquina, lo que supone la ejecución de código remoto. Sin embargo, después de investigar un poco más y probar con un exploit de GitHub hecho por [Mattb709](https://github.com/Mattb709/CVE-2025-24813-PoC-Apache-Tomcat-RCE) veo que la página no es vulnerable porque no se cumple el requisito básico para explotar, siendo este que la página permita peticiones PUT.

![Failed Exploit](./failedExploit.png)

### Basic Auth
Tras fallar en el intento de entrar con el exploit de esta versión investigo durante un rato maneras de atacar a este panel de login poco convencional. Esto lo digo porque la autenticación es un panel de login que aparece como alerta del navegador. Nunca había visto esto pero con investigar un poco descubro que se trata de un HTTP Basic Auth, básicamente, encripta en Base64 la combinación `username:password` y la pasa en los Request Headers como `Authorization: Basic YWRtaW46YWRtaW4=`(ese string equivale a admin:admin). 

![Login](./login.png)

Me encuentro con un [script de ffuf](https://github.com/ffuf/ffuf-scripts/blob/master/README.md) que codifica todas las combinaciones de listas que le pases en Base64 y luego fuzzea en busca de la combinación correcta, en primer lugar pruebo con listas cortas para ver si funciona y el script va bien, aunque ninguna combinación llega a funcionar. Pruebo entonces con el rockyou.txt y una lista de 10 millones de usernames, así que tocará ser pacientes.
```shell
./ffuf_basicauth.sh usernames.txt passwords.txt |ffuf -w -:AUTH -u http://172.17.0.2:8080/manager/html -H "Authorization: Basic AUTH" -fc 403,401 -c
```

![Tropecientos](./tropecientos.png)

### El falso negativo
Me parecía muy raro la cantidad de combinaciones porque iba a tardar muchísimo rato así que lanzo un nmap de nuevo para asegurarme, y resulta que me encuentro con el siguiente resultado:

![Nmap2](./nmap2.png)

### tomcat.txt
Al parecer he sido víctima de mi primer falso negativo, a no ser que haya cometido un error del que no me doy cuenta, iba a borrar todo y retomarlo desde aquí pero voy a dejarlo para que el write-up sea fiel al proceso mental y de investigación.
Con la nueva información de que la máquina permite login anónimo mediante FTP, al entrar veo que hay un archivo llamado `tomcat.txt`, como no puedo hacerle un `cat` me lo descargo con `wget` en la terminal de mi máquina y leemos:

![FTP](./ftp.png)

## Explotación
Encuentro una lista de credenciales default de Tomcat y me la descargo, entonces, con este one-liner (me lo ha dicho DeepSeek, no me escondo, tengo que pulir mi bash) codificamos todo en base64 manteniendo el formato para atacar al panel de nuevo.
```shell
while IFS= read -r line; do echo -n "$line" | base64 | tr -d '\n'; echo ""; done < credentials.txt > encoded_credentials.txt
```

![FFUF](./ffuf.png)

Da con el texto codificado, lo paso por un decoder online y ya sé las credenciales: tomcat y s3cr3t.
### .war
Una vez dentro del panel de admin, veo que hay un campo que permite la subida de archivos `.war`, con una búsqueda en www.revshells.com consigo un one-liner que genera un archivo `shell.war`, solo queda subirlo y acceder a http://172.17.0.2:8080/shell, en esta ocasión no es necesario escalar privilegios, ya que nada más entrar veremos con un `whoami` que ya somos root.  

![WAR](./war.png)

![Root](./root.png)