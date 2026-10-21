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