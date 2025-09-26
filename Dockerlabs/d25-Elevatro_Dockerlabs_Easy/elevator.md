# Elevator
## Reconocimiento

Primero, usando `nmap` escaneo los puertos. 
```
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```
El escaneo revela únicamente un servidor HTTP, por lo que el siguiente paso es visitar la web.

![Escaneo Nmap](./imgs/1.png)

La página muestra las puertas de un ascensor y un botón en el que pone "Abre el ascensor!", al pulsarlo, el ascensor se abre y salta una alerte que dice: "El misterio ha sido resuelto!", entonces se muestra un texto bajo el botón.

```
Scooby-Doo y su pandilla llegaron al Hotel Misty Towers tras recibir una llamada desesperada del gerente. "¡El ascensor número 13 está embrujado! Se mueve solo, hace ruidos extraños y los huéspedes desaparecen misteriosamente."

"¿Ascensor número 13? ¡Eso suena a problemas!" dijo Shaggy, mientras Scooby-Doo asentía temblando. Pero Velma ya había abierto su libreta: "Hay algo raro aquí. Vamos a investigar."

Cuando abrieron las puertas del ascensor, encontraron un túnel secreto que llevaba al sótano. Tras resolver acertijos, evitar trampas y escuchar risas aterradoras, descubrieron al culpable: ¡el viejo mayordomo quería asustar a los huéspedes y usar el hotel como escondite para su botín!

"¡Y habría salido con la suya si no fuera por ustedes, chicos entrometidos!" gritó, mientras Fred, Daphne y Velma sonreían. Scooby y Shaggy encontraron una pizza en el ascensor y, por supuesto, se dieron un festín.
```

Utilizando `gobuster` listo los directorios y endpoints de la web
```
❯ gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,js,txt,db,config,xml,py,bak,conf,java,jar,python -o fuzzing.txt
```
El resultado muestra un directorio que no es habitual, así que fuzzearé de nuevo pero por el en lugar de la raíz.

![Fuzzing directorios](./imgs/2.png)

El fuzzing por el directorio `/themes` revela varios endpoints interesantes, entre ellos el de `archivo.html`, que permite subir archivos.

![Fuzzing themes](./imgs/3.png)

## Explotación
Subo una revshell en php con el nombre `shell.php.jpg`, buscando explotar el bypass de doble extensión, funciona perfectamente a la primera, después accedo al archivo desde `http://IP/themes/uploads` y al solicitarlo gano una shell como `www-data`(habiéndome puesto antes en escucha con `nc -nvlp 443` evidentemente).
## Post explotación
Lo primero es sanitizar la shell
```
export SHELL=bash
export TERM=xterm
export TERM=xterm-256color
source /etc/skel7.bashrc
```

Después, con `sudo -l` veo que puedo ejecutar el binario `env` como Daphne, busco en GTFObins una manera de escalar y con una sencilla línea pivoto a dicho usuario. Entonces, para limpiar de nuevo la consola me paso con una revshell la sesión a otra terminal y sanitizo de nuevo.

![Pivot a Daphne](./imgs/4.png)

Repito el proceso con el usuario `daphne` para pivotar a `vilma`.

![Pivot a Vilma](./imgs/5.png)

Y habiendo pivotado a `vilma`, repito el proceso de nuevo por `shaggy`, `fred`, y `scooby`.

![Pivot a Shaggy, Fred y Scooby](./imgs/6.png)

Por último, y siguiendo una vez más el mismo proceso, llego a `root`.

![Root](./imgs/7.png)