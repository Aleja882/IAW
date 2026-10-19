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