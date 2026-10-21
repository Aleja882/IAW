<<<<<<< HEAD
# Guía Práctica y Memoria del Proyecto — Fase 3

**Asignatura:** Implantación de Aplicaciones Web (IAW)

**Proyecto:** Montaje de Nginx como Proxy Inverso + Apache en el Backend

**Entorno:** Docker y Docker Compose

## 1. ¿Cómo está montado todo esto? (Arquitectura)

En lugar de tener un solo servidor haciendo de todo, hemos separado el trabajo en dos contenedores Docker independientes que se comunican entre sí por una red interna:

1. **`jinx-proxy` (`practica_nginx`):** Es la puerta de entrada. Es el único servidor que da la cara a Internet (escuchando en los puertos `80` para HTTP y `443` para HTTPS). Se encarga de la seguridad, el certificado SSL, la redirección a HTTPS y de frenar el tráfico abusivo antes de que llegue dentro.

2. **`apache-mapache` (`practica_nimbus`):** Es el servidor web interno (escucha en el puerto `80` dentro de la red Docker). Está escondido detrás de Nginx y se encarga de guardar y servir las páginas web reales según el dominio que pida el usuario.

### El archivo `docker-compose.yml`

```yamL
services:
  apache-mapache:
    image: ubuntu/apache2:latest
    container_name: practica_nimbus
    ports:
      - "8011:80"
    volumes:
      - ./html/clientes:/var/www/clientes
      - ./html/privado:/var/www/privado
      - ./apache/config-servicios:/etc/apache2/sites-available
    restart: always

  jinx-proxy:
=======
🌐 NimbusDocs — Portal Documental
Seguro Multi-Marca

Fase 1 — Apache Los dos Virtual Hosts, MPM, protección de recursos (bloque II)
Fase 2 — Nginx Proxy inverso, TLS, cabeceras de seguridad, rate limiting (bloques III y IV)
Fase 3 — Cierre Diagnóstico provocado sobre el conjunto y defensa oral (bloques V y VI)

Los dos dominios ficticios elegidos (uno por marca) y por qué los habéis nombrado así.
  Los dos dominios en mi caso han sido:
  clientes-nimbus.com para los clientes y empre-nimbus.com para los usuarios de la empresa, he utilizado estos dominios por que eran muy facil de recordar y son bastante caracteristicos para los servicios qeu doy.
El MPM de Apache elegido y la justificación técnica de esa elección (repasad UD2,
bloque de MPM).
  El mpm no lo he tenido que cambiar, esto se debe a que la imagen no utiliza el mpm_prefork que tiene por defecto apache tiene el mpm_event que es mas optimo.

Regla de originalidad
Ningún nombre de dominio, puerto, fichero sensible o parámetro de configuración puede
coincidir literalmente con los usados en el guion de clase o en las diapositivas. Se audita con
un simple grep sobre vuestro repositorio.

Los puertos que he utilizado para las paginas son el 8011, el nombre del fichero sensible es nimbus3000.json, esto por que la practica me recordo a harry potter los dominios son los anteriormente dichos y los nombre de los servicios son apache-mapache para el servicio de apache y jinx-proxy a modo de guino a league of legends.

II. Fase 1 — Backend Apache Multi-Marca (UD2)
• Demostrad con curl, usando la cabecera Host explícita, que cada dominio devuelve el
contenido correcto

asir@INF1-05:~/IAW/practica1$ curl -k -H "Host: clientes-nimbus.com" https://localhost/
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Clientes de Nimbus</title>
</head>
<body>
    <h1>Gracias por contar con nosotros</h1>
    <p>Espero que disfruten de nuestros servicios, cualquier duda contacten al 12345678 y un profesional les atendera</p>
</body>

asir@INF1-05:~/IAW/practica1$ tica1$ curl -k -H "Host: empre.com" https://localhost/
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Clientes de Nimbus</title>
</head>
<body>
    <h1>Gracias por contar con nosotros</h1>
    <p>Espero que disfruten de nuestros servicios, cualquier duda contacten al 12345678 y un profesional les atendera</p>
</body>


• La marca "interna" (la que decidáis que no debe ser pública) tiene que proteger un
fichero sensible simulado — con nombre propio, no wp-config.php — y desactivar el
listado de directorios en toda su ruta.
asir@INF1-05:~/IAW/practica1$ curl -k -i -H "Host: empre-nimbus.com" https://localhost/nimbus3000.json00.json
HTTP/1.1 403 Forbidden
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 16:09:13 GMT
Content-Type: text/html; charset=iso-8859-1
Content-Length: 239
Connection: keep-alive
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
</body></html>

• Un dominio que no coincida con ningún Virtual Host debe caer en un comportamiento
controlado por vosotros (no el primero al azar) — explicad en el README por qué pasa
eso.

asir@INF1-05:~/IAW/practica1$ curl -k -i -H "Host: dominio-inexistente.com" https://localhost/
HTTP/1.1 200 OK
Server: nginx/1.31.7
Date: Mon, 19 Oct 2026 16:09:34 GMT
Content-Type: text/html
Content-Length: 377
Connection: keep-alive
Last-Modified: Mon, 19 Oct 2026 15:38:16 GMT
ETag: "179-65e334d7f5d02"
Accept-Ranges: bytes
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN

<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Clientes de Nimbus</title>
</head>
<body>
    <h1>Gracias por contar con nosotros</h1>
    <p>Espero que disfruten de nuestros servicios, cualquier duda contacten al 12345678 y un profesional les atendera</p>
</body>

III. Fase 2 — Terminación TLS en Nginx (UD1 + UD3)

1. Generar vuestro propio certificado de laboratorio (curva ECDSA, igual que en UD1) con
el CN de una de vuestras dos marcas.

openssl req -x509 -nodes -days 365 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 \
  -keyout /etc/nginx/certs/nimbus.key \
  -out /etc/nginx/certs/nimbus.crt \
  -subj "/C=ES/ST=Andalucia/O=NimbusDocs/CN=clientes-nimbus.com"

2. Configurar el bloque server de Nginx con ssl_protocols restringido a TLS 1.2 y 1.3 (nada
de versiones obsoletas) y un cipher suite razonado en el README.

 jinx-proxy:
>>>>>>> b10d2ff68d71702bdc97ca67deaeba3bb328c369
    image: nginx:latest
    container_name: practica_nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      - ./nginx/ec-cert.pem:/etc/nginx/certs/ec-cert.pem
      - ./nginx/ec-privkey.pem:/etc/nginx/certs/ec-privkey.pem
    depends_on:
      - apache-mapache
    restart: always
<<<<<<< HEAD
```

## 2. Configuración de Nginx (`nginx.conf`)

Nginx hace de "intermediario". Si alguien entra por HTTP normal e inseguro, lo redirige automáticamente a HTTPS. Además, cuando le pasa la petición a Apache, le adjunta la cabecera `Host` para que Apache no se pierda.

### Las partes más importantes:

* **Forzar HTTPS:** Si alguien entra por el puerto 80, le devolvemos un código `301` mandándolo a HTTPS:

  ```nginx
  server {
      listen 80;
      server_name clientes-nimbus.com empre-nimbus.com;
      return 301 https://$host$request_uri;
  }
  ```

* **Pasar la pelota a Apache sin perder información:**

  ```nginx
  location / {
      proxy_pass http://apache-mapache:80;
      proxy_set_header Host $host; # ¡Muy importante! Si no ponemos esto, Apache no sabe qué web servir.
  }
  ```

## 3. Poner límites al tráfico (Rate Limiting)

Para evitar que un bot o un ataque nos tumbe el servidor con miles de peticiones por segundo, usamos la técnica de la **cubeta con agujero** (*Leaky Bucket*):

```nginx
# Arriba del todo creamos la zona de memoria (10 MB da para recordar miles de IPs)
limit_req_zone $binary_remote_addr zone=perfil_nimbus:10m rate=3r/s;

# Dentro del location / la aplicamos:
limit_req zone=perfil_nimbus burst=3 nodelay;
```

### Explicado para humanos:

* **`rate=3r/s`:** Es el ritmo normal. Un usuario humano haciendo clics no suele hacer más de 3 peticiones por segundo.

* **`burst=3`:** Es una "mochila de tolerancia". Cuando entras a una web, tu navegador pide de golpe varias cosas (imágenes, CSS...). Este margen permite esos picos pequeños sin bloquear al usuario real.

* **`nodelay`:** Significa "no hagas esperar". Si la petición entra dentro de la ráfaga permitida, respóndela al instante. Pero en cuanto el usuario se ponga a mandar peticiones a lo loco y desborde el límite, le cortamos el grifo devolviéndole un error **503**.

## 4. Los Virtual Hosts de Apache

Apache recibe la petición limpia desde Nginx. Para saber qué carpeta enseñar, mira el dominio que viene en la cabecera `Host`:

* Si pides `clientes-nimbus.com` $\rightarrow$ Te enseña la carpeta `/var/www/clientes`.

* Si pides `empre-nimbus.com` $\rightarrow$ Te enseña la carpeta `/var/www/privado`.

### Ejemplo de configuración (`cliente.conf`):

```apache
<VirtualHost *:80>
    ServerName clientes-nimbus.com
    DocumentRoot "/var/www/clientes"
    <Directory "/var/www/clientes">
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

## 5. Típicos fallos y cómo los solucionamos (Fase 3)

### Fallo 1: "El límite de peticiones no hace nada" (Phantom Rate Limiting)

* **¿Qué pasaba?:** Hacías 50 peticiones seguidas con `curl` y todas daban `200 OK`. Nginx no bloqueaba nada.

* **¿Por qué ocurría?:** Habíamos definido la zona `limit_req_zone` arriba en la configuración, pero se nos olvidó escribir la línea `limit_req zone=...` dentro del bloque `location /`. Nginx sabía que existía la regla, ¡pero nadie le había dicho dónde aplicarla!

* **Solución:** Añadir la directiva `limit_req` dentro de `location /`.

### Fallo 2: "Siempre sale la página por defecto de Ubuntu" (Catch-All)

* **¿Qué pasaba?:** Al hacer `curl` pidiendo `clientes-nimbus.com`, Apache devolvía el HTML genérico de bienvenida de Apache/Ubuntu.

* **¿Por qué ocurría?:** El archivo `000-default.conf` venía activado por defecto y Apache lo cargaba primero. Además, nuestro archivo `cliente.conf` estaba guardado en `sites-available` pero no estaba activado en `sites-enabled`.

* **Solución:** Entrar al contenedor de Apache para desactivar el sitio por defecto y activar el nuestro:

  ```bash
  # 1. Entramos en la terminal interactiva del contenedor de Apache
  docker exec -it practica_nimbus bash

  # 2. Dentro de Apache, desactivamos el sitio por defecto y activamos el del cliente
  a2dissite 000-default.conf
  a2ensite cliente.conf

  # 3. Recargamos la configuración de Apache
  apache2ctl graceful

  # 4. Salimos de la consola del contenedor
  exit
  ```

## 6. Machete de comandos para la defensa en directo

### 1. Levantar todo desde cero de forma limpia (en tu máquina anfitriona)

```bash
docker compose down
docker compose up -d
docker compose ps
```

### 2. Comprobar que los dominios responden bien (en tu máquina anfitriona)

```bash
curl -k -H "Host: clientes-nimbus.com" https://localhost/
curl -k -H "Host: empre-nimbus.com" https://localhost/
```

### 3. Demostrar el Rate Limiting en directo (en tu máquina anfitriona)

Ejecuta este bucle para enviar 10 peticiones del tirón. Verás cómo las primeras dan `200` y en cuanto se llena el margen, Nginx las frena con `503`:

```bash
for i in {1..10}; do curl -k -o /dev/null -s -w "%{http_code}\n" -H "Host: clientes-nimbus.com" https://localhost/; done
```

### 4. Recargar configuraciones entrando en cada contenedor

* **Para recargar Apache (desde dentro de `practica_nimbus`):**
  ```bash
  docker exec -it practica_nimbus bash
  apache2ctl graceful
  exit
  ```

* **Para recargar Nginx (desde dentro de `practica_nginx`):**
  ```bash
  docker exec -it practica_nginx bash
  nginx -s reload
  exit
  ```
=======

3. Redirigir automáticamente todo el tráfico HTTP (puerto 80) a HTTPS con un código de
estado correcto — justificad en el README por qué ese código y no otro.

server {
    listen 80;
    server_name clientes-nimbus.com empre-nimbus.com;
    return 301 https://$host$request_uri;
}
Este utiliza el código de estado **`301` (Moved Permanently)** porque permite a los clientes y navegadores almacenar en caché la redirección y acceder directamente por HTTPS en futuras peticiones sin pasar por el puerto HTTP

4. Nginx debe reenviar la petición a Apache preservando la cabecera Host original — sin
eso, Apache no puede resolver a qué marca corresponde la petición. Probadlo primero
SIN esa línea para ver el fallo con vuestros propios ojos antes de añadirla.

proxy_pass http://apache-mapache:8011;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;

5. Verificar con openssl s_client el protocolo y cifrado negociados, y adjuntar la salida real
(no una captura de pantalla del guion) en el README.

Ejecutamos: openssl s_client -connect localhost:443 -servername clientes-nimbus.com para comprobarlo
openssl s_client -connect localhost:443 -servername clientes-nimbus.com

Protocol  : TLSv1.3

IV. Fase 2 — Seguridad y Límite de Peticiones (UD2 +
UD3)
Demostrad con una ráfaga de peticiones repetidas (un bucle for con curl, o ab desde
un contenedor cliente) que el límite calculado realmente bloquea, adjuntando la salida.
for i in {1..10}; do curl -k -o /dev/null -s -w "Petición $i -> HTTP Status: %{http_code}\n" --resolve clientes-nimbus.com:443:127.0.0.1 https://clientes-nimbus.com/; sleep 0.1; done

V. Fase 3 — Diagnóstico Provocado

En apache cambiamos el nombre de por ejemplo empre-nimbus, o lo comentamos y cuando hacemos el curl nos da error.

En nginx comentamos la primera linea del proxy y nos da error al hacer el curl ya que no le devuelve el trafico a apache.
>>>>>>> b10d2ff68d71702bdc97ca67deaeba3bb328c369
