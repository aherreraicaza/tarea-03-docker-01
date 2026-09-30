# Práctica Docker

He usado alpine:3.22 para hacer la práctica. Pongo esa versión en vez de
latest porque así no cambia la imagen de un paso a otro.

Primero pongo "docker pull alpine:3.22". Ya tenía la imagen descargada, así que
me sale que está actualizada.

![captura](capturas/01-descargar-imagen.png)

Esto solo descarga la imagen, no crea el contenedor. Después miro "docker images"
y ahí sale Alpine ocupando 8,3 MB. Las otras imágenes ya las tenía de antes.

![captura](capturas/02-comprobar-imagen.png)

Pongo "docker create alpine:3.22 sh" y me devuelve un id.

![captura](capturas/03-crear-sin-nombre.png)

Lo miro con "docker ps -a" y luego con "docker ps".

![captura](capturas/04-estado-created.png)

Sale en estado Created, o sea que está creado pero no arrancado. Por eso aparece
en "docker ps -a" y no en "docker ps", que solo muestra los que están funcionando.

Como no le puse nombre, Docker le ha puesto friendly_brahmagupta. Estos nombres
son aleatorios; para elegir uno hay que poner --name.

Ahora pongo "docker run -dit --name dam_alp1 alpine:3.22 sh". Esta vez sí lo
arranca, y con "docker ps" veo que está funcionando.

![captura](capturas/05-arrancar-dam_alp1.png)

La -d es para dejarlo en segundo plano, la -i mantiene abierta la entrada
y la -t le da una terminal. Así la shell no se cierra al arrancarlo.

Entro con "docker exec -it dam_alp1 sh". Dentro pongo "ip -4 addr show eth0"
y después "ping -c 3 google.com".

![captura](capturas/06-ip-y-ping-google.png)

Me sale la IP 172.17.0.2. El ping a Google va bien: llegan los tres paquetes
y hay un 0% de pérdida. Tiene salida a internet.

Creo el segundo igual, con "docker run -dit --name dam_alp2 alpine:3.22 sh".
Miro su IP con "docker exec dam_alp2 ip -4 addr show eth0" y sale 172.17.0.3.
El primero tiene 172.17.0.2.

Por IP funciona con "docker exec dam_alp1 ping -c 2 172.17.0.3". Pero al probar
con el nombre dam_alp2 da error.

![captura](capturas/07a-pings-bridge-defecto.png)

Sale el error ping: bad address 'dam_alp2'. En la red bridge que viene por defecto,
Docker no busca la IP a partir del nombre del contenedor. Lo compruebo con
"docker exec dam_alp1 nslookup dam_alp2": pregunta a 8.8.8.8 y contesta
NXDOMAIN, porque ese DNS no conoce nuestros contenedores.

Para que vaya por nombre creo una red con
"docker network create --driver bridge dam_red_dam". Conecto los dos
con "docker network connect dam_red_dam dam_alp1" y
"docker network connect dam_red_dam dam_alp2", y repito el ping.

![captura](capturas/07b-pings-red-usuario.png)

Ahora sí va por el nombre. En nslookup sale el DNS de Docker, 127.0.0.11,
que ya sabe que dam_alp2 tiene la IP 172.18.0.3.

En la nueva red tienen las IP 172.18.0.2 y 172.18.0.3. Pruebo también por
esa IP y al revés, desde dam_alp2 a dam_alp1, y funciona en los dos sentidos.

![captura](capturas/07c-pings-red-usuario-ip.png)

Uso "docker stats --no-stream" para verlo. Con --no-stream muestra los datos
una vez, sin quedarse actualizándolos.

![captura](capturas/08-memoria-stats.png)

Me salen 536 KiB en dam_alp1 y 528 KiB en dam_alp2. Consumen muy poco porque
solo está abierta la shell. La CPU sale al 0%.

Entro otra vez con "docker exec -it dam_alp1 sh", escribo "exit" y después
miro "docker ps".

![captura](capturas/09-exit-desligado.png)

Siguen funcionando los dos. El exit ha cerrado la shell que abrí con
docker exec, pero no el proceso principal del contenedor.

Pruebo también con otro contenedor sin la -d:
"docker run -it --name dam_alp_attached alpine:3.22 sh". Aquí la shell es el
proceso principal. Pongo "echo $$" y sale 1.

![captura](capturas/10-exit-adherido.png)

En este caso, al poner "exit" sí se para el contenedor, porque estoy cerrando
su proceso principal. Ya no sale en "docker ps". Lo miro con
"docker ps -a --filter name=dam_alp_attached" y aparece como Exited (0).
Para arrancarlo otra vez se puede usar "docker start dam_alp_attached".

Pongo "docker system df" para ver cuánto espacio ocupa Docker.

![captura](capturas/11a-disco-system-df.png)

Y con "docker system df -v" veo lo que ocupa cada imagen y cada contenedor.

![captura](capturas/11b-disco-system-dfv.png)

Las imágenes ocupan 248,6 MB en total, pero no todo es de esta práctica.
También están las imágenes que tenía antes. Alpine ocupa 8,3 MB.

La imagen es la base para crear los contenedores. Los cuatro de Alpine usan la
misma, así que esos 8,3 MB no se guardan cuatro veces. Cada contenedor guarda
aparte los archivos que se añaden o cambian dentro.

Con "docker ps -a --size", dam_alp1 sale con 47 B y dam_alp_attached con
13 B. Es el historial que ha guardado la shell. Los otros dos de Alpine salen
con 0 B. Donde pone virtual 8.3MB está contando también la imagen compartida.

Todos los contenedores juntos ocupan 1,153 kB, contando los que tenía de antes.
Vamos, que en esta práctica lo que más ocupa es la imagen; dentro de los
contenedores apenas hemos guardado nada.

![captura](capturas/11c-imagenes-vs-contenedores.png)
