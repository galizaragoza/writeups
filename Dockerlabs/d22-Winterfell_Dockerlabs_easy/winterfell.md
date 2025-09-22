# Winterfell
## Reconocimiento
En primer lugar, escaneo los puertos con `nmap`.
```sh
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![Escaneo de puertos](./imgs/1.png)

Un servidor HTTP de título "Juego de Tronos", dos puertos sirviendo conexiones NetBIOS y SSH.

La página es una especie de fanblog de juego de tronos (a ver si George saca ya Vientos de Invierno porque las ultimas dos temporadas de la serie...), más allá de una imagen que podría contener metadatos o esteganografía no veo nada que llame la atención.

Con gobuster busco directorios o archivos y encuentro el directorio `/dragon`.
```
❯ gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,js,txt,db,config,xml,py,bak,conf,java,jar,python -o fuzzing.txt
```
Al visitarlo, veo un archivo de texto cuyo contenido es este:
```
Estos son todos los Episodios de la primera  temporada de Juego de tronos.
Tengo la barra espaciadora estropeada por lo que dejare los nombres sin espacios, perdonad las molestias

seacercaelinvierno
elcaminoreal
lordnieve
tullidosbastardosycosasrotas
elloboyelleon
unacoronadeoro
ganasomueres
porelladodelapunta
baelor
fuegoyhielo
```
Seguramente alguno de estos títulos sea la contraseña de un usuario, pero para poder atacar por fuerza bruta primero hay que saber que usuario(s) hay.

Con enum4linux encuentro 3 usuarios:
```
❯ enum4linux -a 172.17.0.2
```

![Usuarios encontrados](./imgs/2.png)


## Explotación
Usando `crackmapexec` doy con la contraseña del usuario jon

![Contraseña encontrada](./imgs/3.png)

Después, ya con las credenciales, usando `smbmap` busco archivos que puedan revelar información sensible.
```sh
❯ smbmap -H 172.17.0.2 -u jon -p seacercaelinvierno -r jon
## Examino el contenido de la carpeta y encuentro el archivo paraJon
❯ smbmap -H 172.17.0.2 -u jon -p seacercaelinvierno -download jon/paraJon

❯ cat paraJon.txt
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   │ File: paraJon.txt
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ Jon para todos los mensajes que quieras encriptar debes de usar la herramienta oculta que te he dejado

```
Hecho un vistazo de nuevo y veo que me había dejado un archivo, vuelvo para descargarlo
```sh
❯ smbmap -H 172.17.0.2 -u jon -p seacercaelinvierno --download "jon/.mensaje.py"
```
Y examino el contenido.

![Contenido de .mensaje.py](./imgs/4.png)

Explorando un poco más los directorios doy con un archivo llamado `proteccion_del_reino`, lo descargo y leo:
```
Aria de ti depende que los caminantes blancos no consigan pasar el muro. 
Tienes que llevar a la reina Daenerys el mensaje, solo ella sabra interpretarlo. Se encuentra cifrado en un lenguaje antiguo y
dificil de entender. 
Esta es mi contraseña, se encuentra cifrada en ese lenguaje y es -> aGlqb2RlbGFuaXN0ZXI=
```

Esta encodeado en base64, lo paso por un decoder online y obtengo la contraseña para acceder mediante ssh.
## Post-explotación
Una vez dentro como jon, al correr `sudo -l` veo que puedo usar como Aria el binario `python3` en un archivo llamado `.mensaje.py`, no tengo permisos de escritura en el archivo pero si en el directorio, así que lo borro y creo de nuevo, esta vez poniendo en el una revshell de python3, con la que me doy una sesión como Aria en otra terminal

![Escalada a Aria](./imgs/5.png)

Hago sudo `sudo -l` de nuevo y veo que puedo usar `cat` y `ls` como daenerys, así que exploro el directorio de dicho user y haciendo uso de `cat` para leer un mensaje que contiene la contraseña de daenerys.

![Contraseña de daenerys](./imgs/6.png)

Como daenerys, puedo usar bash como root en el archivo `.shell.sh`, sobreescribo el contenido de ese archivo con una instrucción que añadirá el bit SUID al binario `bash`, eso hará que este binario herede los privilegios de su dueño si se ejecuta con el flag `-p`, es decir, cuando luego hago `bash -p`, ya tengo una shell como root.

![Shell como root](./imgs/7.png)