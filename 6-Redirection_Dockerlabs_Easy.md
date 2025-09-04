# [Check it out on GitBook](https://app.gitbook.com/o/XpQOLQ50bhBjUPaxkpRB/s/8oAAD9KYsuqsIgDHFlYS/dockerlabs/redirection-easy-dockerlabs)
# Writeup
## Reconocimiento
### Nmap
En primer lugar empiezo por escanear con Nmap los puertos abiertos de la máquina, para ver ante que nos encontramos, que servicios tiene activos y en que puertos.

![[nmap.png]]

En principio simplemente hay un servidor HTTP en el puerto 80 de título "Laboratorio de Open Redirect". 
Al visitar la página nos encontramos con un laboratorio para practicar la vulnerabilidad de Open Redirect, así que vamos a ir resolviendo los los laboratorios uno a uno.

![[lab.png]]

## Laboratorio 1
El primer laboratorio es muy sencillo, en Firefox, simplemente al hacer hover sobre el botón vemos que lleva el siguiente enlace: `http://172.17.0.2/laboratorio1/redirect.php?url=http://google.com`, con lo cual, lo único que debemos para explotar Open Redirect aquí sería modificar este enlace con el fin de redirigir al objetivo o víctima a la URL maliciosa que tengamos preparada.
Por ejemplo, imaginemos que le envío con un correo electrónico a la víctima en cuestión con un enlace aparentemente legítimo, ya que nos estamos aprovechando de la reputación establecida de la aplicación vulnerable. Cuando el usuario clique en dicho enlace, será redirigido a DockerLabs, ya que la URL completa es tal que así: `http://172.17.0.2/laboratorio1/redirect.php?url=https://dockerlabs.es`
Si en lugar de DockerLabs, tuviese mi propio sitio web malicioso, por ejemplo `http://172.17.0.2/laboratorio1/redirect.php?url=https://dame-tu-cookie.com`, el usuario sería redirigido a esa página, abriendo la puerta a ataques de todo tipo como robo de credenciales, phishing, robo de cookies...
## Laboratorio 2
En el laboratorio 2, si probamos con un enlace igual al anterior (pero cambiando el 1 por un 2), veremos el siguiente mensaje:

![[lab2.png]]

Esto quiere decir que seguramente haya una whitelist y el único dominio al que se permite redirigir es a google.com, por lo que vamos a una serie de bypasses a ver si podemos burlar este control. Después de probar un rato con bypasses de lo más enrevesados me doy cuenta que no había probado sencillamente con `@` y es que funciona, la URL queda así `http://172.17.0.2/laboratorio2/redirect.php?url=https://www.google.com@dockerlabs.es` aunque le veo una pega mayor a este método, y es que por lo menos en Firefox salta la siguiente alerta.

![[confirmacion-lab2.png]]

He probado también en Google Chrome en la versión 138.0.7204.100 y Chromium 138.0.7204.92 y ambas han seguido la redirección sin avisos.
## Laboratorio 3
Para el último laboratorio, la forma de exponer la vulnerabilidad es inyectando un parámetro antes de `google.com`, tal que así: `172.17.0.2/laboratorio3/redirect.php?url=https://dockerlabs.google.com`, en esta ocasión no somos capaces de completar la redirección a otro sitio a pesar de exponer la vulnerabilidad, quizá con algún dominio que tenga .google.com al final ofuscado (si es que eso se puede hacer) o probando más bypasses podríamos acabar teniendo material para un ataque.
## Escalada de privilegios
Al entrar navego a la raíz de un solo `ls` salta a la vista un archivo de nombre `secret.bak`, al cual hago un cat y ahí están las credenciales para cambiar al usuario balulito.

![[balulito.png]]

Una vez soy balulito, pruebo de nuevo `sudo -l`, esta vez sale un binario que podemos usar como vector de escalada: `cp`. Siguiendo las instrucciones de GTFObins y unos consejos que encontré en un foro, hago un cat a `/etc/passwd`, creo un archivo con ese mismo nombre en el directorio en el que me encuentro y pego el contenido de passwd allí, solo que elimino la contraseña de root borrando la x que hay tras los primeros dos puntos.
Guardo el archivo y aprovechando los permisos del binario `cp`, sobreescribo el archivo passwd original con el modificado, sin contraseña en root. Después de esto ya solo queda hacer un `su` y ya estoy dentro como root.

![[Ciberseguridad/CTFs/253_septiembre/4-redirection/root.png]]
