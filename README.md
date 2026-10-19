 La practica se divide basicamente en dos; una primera con Apache donde crearemos los dos virtual host con todo lo que esto conlleva.Y una segunda donde con Nginx crearemos un proxy inverso para controlar el trafico de la red.En mi caso los dominios elegidos para la parte de apache son empre-nimbus.com, este es el dominio para la marca de empresa y clientes-nimbus.com para los clientes de nuestra empresa.
 En cuanto al MPM he utilizado el mpm_event que para ser honesto la forma de implantarlo ha sido puramente gracias a la ia por que era el tema con el que mas perdido estaba.
 En esta carpeta encontraras tres "zonas" una primera con el docker compose, el readme y las carpetas de apache y html.
 en el docker compose enconctraras lo siguiente:
 services:
  backend-apache:
    build: ./apache
    container_name: practica1_nimbus
    ports:
      - "8011:8011"
    volumes:
      - ./html/privado:/var/www/privado
      - ./html/clientes:/var/www/clientes
    restart: always


Lo que nos dira los servicios en este caso apache, el contenedor y su nombre que es el de practica1_nimbus, los puertos, que yo he elegido el 8011 por aplicar la regla de originalidad, encontraremos los volumenes que es en que carpetas esta alojado cada servidor y marcamos que se inicie siempre.

Si nos centramos en la carpeta de apache encontramos una carpeta llamada sites-available con las configuraciones de apache(000-default.conf), la del servidor cliente y la del privado; tambien cuenta con la configuracion de puertos y el Dockerfile el cual cuenta con la version de docker, el puerto al que escucha y la desactivacion de prefork que es el mpm default y activacion de el mpm_event que es el que actua como Nginx. Esto ha sido gran parte realizado por la ia por que era lo que menos idea tenia.

En la carpeta de html contamos con el html del servidor clientes, el index de privado(el de empresa) y el archivo que tiene que estar oculto.

Realizando los curls nos da el html de todo pero cuando intentamos acceder al nimbus3000.json nos da error 403 que s lo que nos piden.



