# Los 40 ladrones
## Reconocimiento
En primer lugar escaneo los puertos de la máquina para sacar información de que vías pueden ser vectores de ataque
```shell
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![1.png](./imgs/1.png)

Solo hay un servidor HTTP que pueda resultar interesante, al visitarla me encuentro con la página default de Apache2. Echando un vistazo al código fuente no encuentro nada que pueda resultar útil, decido fuzzear en busca de algún directorio que pueda resultar útil.
```shell
❯ gobuster dir -u 'http://172.17.0.2/' -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,js,txt,db,config,xml,py,bak,conf,java,jar,python -o fuzzing.txt
```
El fuzzeo revela un directorio llamadp `/qdefense.txt`, que contiene un texto:

> Recuerda llama antes de entrar , no seas como toctoc el maleducado
> 7000 8000 9000
> busca y llama +54 2933574639

Después de pensar durante un rato consulto el writeup de otro compañero ([D1se0](https://dise0.gitbook.io/h4cker_b00k/ctf/dockerlabs/los-40-ladrones-dockerlabs-easy)) y descubro un concepto que no conocía: *port knocking*
Es una forma de abrir puertos cerrados introduciendo cierta secuencia, en este caso: 7000 8000 9000
## Explotación
Hago knock a la máquina introduciendo la secuencia que había en el texto
```sh
❯ knock 172.17.0.2 7000 8000 9000
```
Al lanzar de nuevo un escaneo, veo que se ha abierto en el puerto 22, sirviendo SSH.

![2.png](./imgs/2.png)

Pruebo a entrar con las credenciales `toctoc:+54 2933574639` y `toctoc:2933574639` pero no da resultado. Pruebo a atacar mediante diccionario con hydra y da resultados

![3.png](./imgs/3.png)
## Post explotación
La escalada de privilegios resulta bastante sencilla, nada más entrar, al lanzar `sudo -l` veo que puedo usar `bash` como root, así que simplemente con hacer `sudo /opt/bash` gano privilegios root.

![4.png](./imgs/4.png)