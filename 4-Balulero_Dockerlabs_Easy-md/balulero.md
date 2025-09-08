# Info
## IP Víctima
```
172.17.0.2
```

# Writeup
## Reconocimiento
### Nmap
En primer lugar, empezamos por realizar un escaneo de puertos como de costumbre, para ver que tenemos en frente, que servicios están corriendo y por donde podemos comenzar a buscar vectores de ataque.

![Nmap Scan](./nmap-scan.png)

Nos encontramos con un servidor HTTP en el puerto 80 y nada más a primera vista, por lo que voy a echarle un vistazo a la página que allí tenemos a falta de otro lugar donde mirar.
### La página de Balú
Después de un vistazo rápido a la página veo que es una única landing, hay varios enlaces pero quitando los de las RRSS todos son internos, no hay redirecciones. A primera vista no veo nada donde empezar a probar cosas, así que voy a lanzar un fuzzing a ver si consigo encontrar alguna ruta o directorio oculto que pueda resultar interesante mientras echo un ojo el código fuente.
El fuzzing revela un archivo muy interesante, script.js, así que voy a echarle un ojo a ver que me encuentro.

![Fuzzing](./fuzzing.png)
## Explotación
### El desliz de Balú
En el archivo rápidamente salta a la lista que Balú, pese a sus conocimientos, se ha dejado un .env de backup expuesto en una ruta oculta de la página, así que vamos a tomar las credenciales de allí para ver si podemos entrar con ellas mediante SSH.

![Baluchingon](./baluchingon.png)

Efectivamente, las claves funcionan y ya estamos dentro como balu.
## Post explotación
### Chocolate
Nada más entrar hago un `sudo -l` y veo que se puede ejecutar el binario php como el usuario chocolate. Con una búsqueda rápida en [GTFObins](https://gtfobins.github.io/gtfobins/php/#sudo) somos capaces de pivotar sin complicaciones al usuario chocolate.

![Binario](./binario.png)

Ahora, lanzo un `ps aux` para ver que servicios se están ejecutando en segundo plano, y me fijo en un script de php que se lanza cada 5 segundos como root.

![PS Aux](./psaux.png)

Puesto que podemos sobrescribir el archivo usando echo y el operador `>` podemos añadir el bit SUID a bash, de manera que cuando lo ejecutemos con la flag `-p` heredará los privilegios del owner, que es root. A los cinco segundos comprobamos con `ls -la /bin/bash` y vemos que ha funcionado, solo queda lanzar `bash -p` y ya somos root.

![Root](./root.png)

