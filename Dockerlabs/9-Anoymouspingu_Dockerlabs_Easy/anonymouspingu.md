# Anonymous pingu
## Reconocimiento
En primer lugar lanzo un nmap para identificar servicios, ya de primeras llama mucho la atención ese servidor FTP con un directorio que permite escritura y login anónimo.

![nmap](Ciberseguridad/CTFs/253_septiembre/11-anoymouspingu/imgs/nmap.png)

## Explotación
Ni siquiera hace falta visitar la página, basta con subir una revshell de PHP a la página y luego hacer una solicitud a `http://ip-dockerlab/upload/shell.php`. Subo el archivo al servidor FTP con `curl` y no da ningún problema, ahora sé que esta en `/upload`
```
curl -T shell.php ftp://172.17.0.2/upload/
```

Ahora, al visitar `http://172.17.0.2/upload/` me encuentro con esto:

![shell](shell.png)

Simplemente clico en el archivo y ya estoy dentro como `www-data`

![www-data](Ciberseguridad/CTFs/253_septiembre/11-anoymouspingu/imgs/www-data.png)


## Post explotación
Lo primero al entrar es sanitizar la shell con:
```
export SHELL=bash
export TERM=xterm-256color
source /etc/skel/.bashrc
```
Luego, lanzando un `sudo -l` rápidamente se ve que podemos escalar al usuario `pingu` a través del binario man, sin embargo, después de varios intentos veo que no hay manera, así que pruebo una [sanitización alternativa](https://invertebr4do.github.io/tratamiento-de-tty/#) por si era la shell y ahora si funciona.

![man](man.png)

![pingu](pingu.png)

Una vez ya soy pingu, al lanzar `sudo -l` de nuevo veo dos binarios a través de los que escalar al usuario `gladys`, y me quedo con `dpkg`, simplemente con:
```
sudo -u gladys dpkg -l
## Escribir !/bin/bash
```
Ya he escalado al usuario `gladys`

![gladys](gladys.png)

Después de un rato, no supe terminar esta parte, y me serví del [writeup de d1se0](https://dise0.gitbook.io/h4cker_b00k/ctf/dockerlabs/anonymouspingu-dockerlabs-easy#escalate-privileges) para aprender el método y finalizar.