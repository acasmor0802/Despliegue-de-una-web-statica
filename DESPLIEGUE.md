# Documentacion de Despliegue - Servidor Web Nginx

## Indice
- [Parte 2 - Evaluacion RA2](#parte-2---evaluacion-ra2)
  - [a) Parametros de administracion](#a-parametros-de-administracion-mas-importantes-del-servidor-web)
  - [b) Ampliacion de funcionalidad](#b-ampliacion-de-funcionalidad-mediante-activacion-y-configuracion)
  - [c) Sitios virtuales / multi-sitio](#c-creacion-y-configuracion-de-sitios-virtuales--multi-sitio)
  - [d) Autenticacion y control de acceso](#d-autenticacion-y-control-de-acceso)
  - [e) Certificados digitales](#e-obtencion-e-instalacion-de-certificados-digitales)
  - [f) Asegurar comunicaciones](#f-asegurar-comunicaciones-cliente-servidor)
  - [g) Documentacion](#g-documentacion)
  - [h) Ajustes para implantacion](#h-ajustes-necesarios-para-implantacion-de-aplicaciones)
  - [i) Virtualizacion en despliegue](#i-virtualizacion-en-despliegue)
  - [j) Logs y monitorizacion](#j-logs-monitorizacion-consolidacion-y-analisis)
- [Checklist Final](#checklist-final)

---

## Parte 2 - Evaluacion RA2

### a) Parametros de administracion mas importantes del servidor web

#### Directivas localizadas en /etc/nginx/nginx.conf

| Directiva | Que controla | Configuracion incorrecta y efecto | Como comprobarlo |
|-----------|--------------|-----------------------------------|------------------|
| `worker_processes` | Numero de procesos worker. `auto` detecta CPUs disponibles | `worker_processes 1;` en servidor con 8 CPUs limita rendimiento | `grep worker_processes /etc/nginx/nginx.conf` |
| `worker_connections` | Conexiones simultaneas por worker (defecto: 1024) | `worker_connections 10;` rechaza conexiones en produccion | `grep worker_connections /etc/nginx/nginx.conf` |
| `access_log` | Ubicacion del registro de accesos HTTP | `access_log off;` impide analisis de trafico | `ls -lh /var/log/nginx/access.log` |
| `error_log` | Ubicacion y nivel de logging de errores | `error_log debug;` en produccion genera logs enormes | `tail /var/log/nginx/error.log` |
| `keepalive_timeout` | Tiempo de conexion keep-alive (defecto: 65s) | `keepalive_timeout 300;` agota recursos del servidor | `nginx -T \| grep keepalive` |
| `include` | Incluye archivos de configuracion adicionales | Olvidar `include /etc/nginx/conf.d/*.conf;` ignora virtual hosts | `ls -la /etc/nginx/conf.d/` |
| `gzip` | Compresion gzip de respuestas HTTP | `gzip off;` desperdicia ancho de banda | `curl -I -H "Accept-Encoding: gzip"` |

#### Cambio aplicado

Se modifico `keepalive_timeout` de 65 a 30 segundos en `conf/default.conf`:

```nginx
server {
    listen 443 ssl;
    keepalive_timeout  30;
}
```

#### Evidencias
- [nginxconfDirectiva.png](evidencias/nginxconfDirectiva.png) - Directivas en nginx.conf
- [02_nginx-t.png](evidencias/02_nginx-t.png) - Validacion correcta
- [configuracionCargada.png](evidencias/configuracionCargada.png) - Recarga correcta

---

### b) Ampliacion de funcionalidad mediante activacion y configuracion

#### B1) Opcion Gzip

Archivo `conf/gzip.conf`:
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_comp_level 5;
gzip_vary on;
```

Montaje en docker-compose.yml:
```yaml
volumes:
  - ./conf/gzip.conf:/etc/nginx/conf.d/gzip.conf
```

#### Evidencias
- [b1-01_gzipconf.png](evidencias/b1-01_gzipconf.png) - Contenido de gzip.conf
- [b1-02_compose_volume_gzip.png](evidencias/b1-02_compose_volume_gzip.png) - Montaje en docker-compose.yml
- [b1-03_nginx-t.png](evidencias/b1-03_nginx-t.png) - Config valida
- [b1-04_curl_gzip.png](evidencias/b1-04_curl_gzip.png) - Content-Encoding: gzip en respuesta

#### Modulo investigado: ngx_http_headers_more_module

**Nombre**: headers-more-nginx-module

**Para que sirve**: Permite manipular cabeceras HTTP de peticiones y respuestas de forma avanzada. Puede establecer, eliminar y modificar cabeceras, a diferencia del modulo estandar que solo permite `add_header`.

**Como se instala/carga**:
- Compilacion: `./configure --add-module=/path/to/headers-more-nginx-module`
- Modulo dinamico: `load_module modules/ngx_http_headers_more_filter_module.so;`
- Paquete Debian/Ubuntu: `apt-get install libnginx-mod-http-headers-more-filter`

**Fuentes consultadas**:
- https://github.com/openresty/headers-more-nginx-module
- https://www.nginx.com/resources/wiki/modules/headers_more/

---

### c) Creacion y configuracion de sitios virtuales / multi-sitio

#### Multi-sitio implementado
- Web principal en `/`
- Web secundaria en `/reloj`

#### Diferencia entre multi-sitio por path y por nombre

El multi-sitio por PATH utiliza diferentes rutas URL dentro del mismo dominio, configurado mediante bloques `location` en el mismo `server`. El cliente accede a `https://localhost/` y `https://localhost/reloj`.

El multi-sitio por NOMBRE (virtual hosting) utiliza diferentes dominios configurados mediante bloques `server` con diferentes `server_name`. Nginx selecciona el server block segun la cabecera Host de la peticion.

#### Otros tipos de multi-sitio
1. **Por puerto**: Diferentes aplicaciones en distintos puertos (80, 8080, 3000)
2. **Por IP**: Cada server block escucha en una IP especifica del servidor

#### Configuracion aplicada

```nginx
root /usr/share/nginx/html;

location / {
    try_files $uri $uri/ =404;
}

location /reloj {
    alias /usr/share/nginx/html/reloj;
    index index.html;
    try_files $uri $uri/ /reloj/index.html;
}
```

#### Evidencias
- [c1-01-root.png](evidencias/c1-01-root.png) - Web principal en /
- [clock.png](evidencias/clock.png) - Web secundaria en /reloj
- [c1-03-defaultconf.png](evidencias/c1-03-defaultconf.png) - default.conf dentro del contenedor

---

### d) Autenticacion y control de acceso

#### Configuracion de /admin

```nginx
location /admin/ {
    alias /usr/share/nginx/html/admin/;
    index index.html;
    auth_basic "Area Restringida";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

#### Evidencias
- [d-01-admin-html.png](evidencias/d-01-admin-html.png) - Contenido webdata/admin/index.html
- [d-02-defaultconf_auth.png](evidencias/d-02-defaultconf_auth.png) - location /admin/ con auth_basic
- [d-03curl-401.png](evidencias/d-03curl-401.png) - Acceso sin credenciales: 401
- [d-04curl-200.png](evidencias/d-04curl-200.png) - Acceso con credenciales: 200

---

### e) Obtencion e instalacion de certificados digitales

#### Que es .crt y .key

- **.crt (Certificado)**: Contiene la clave publica y la informacion del certificado (dominio, organizacion, validez, emisor). Se puede compartir publicamente.
- **.key (Clave privada)**: Contiene la clave privada asociada. Nunca debe compartirse. Se usa para descifrar datos encriptados con la clave publica.

#### Por que -nodes en laboratorio

La opcion `-nodes` indica que no se cifre la clave privada con contrasena. En laboratorio permite que Nginx inicie automaticamente sin introducir contrasena. En produccion se recomienda cifrar la clave privada.

#### Evidencias
- [e-01-ls-certs.png](evidencias/e-01-ls-certs.png) - Certificados en el host
- [e-02-compose-certs.png](evidencias/e-02-compose-certs.png) - Montaje en docker-compose.yml
- [e-03-defaultconf-ssl.png](evidencias/e-03-defaultconf-ssl.png) - ssl_certificate y ssl_certificate_key en default.conf

---

### f) Asegurar comunicaciones cliente-servidor

#### Configuracion de dos server blocks

**Server Block puerto 80 (redireccion)**:
```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}
```

**Server Block puerto 443 (servicio HTTPS)**:
```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
}
```

#### Por que dos server blocks

Se usan dos server blocks para separar responsabilidades: el bloque HTTP (80) solo redirige a HTTPS, garantizando que todas las peticiones se cifren. El bloque HTTPS (443) sirve el contenido real. Esto evita servir contenido sin cifrar accidentalmente.

#### Evidencias
- [f-01-https.png](evidencias/f-01-https.png) - Navegacion por https://localhost:8443
- [f-02-301-network.png](evidencias/f-02-301-network.png) - 301 en DevTools al entrar por HTTP

---

### g) Documentacion

#### Arquitectura

**Servicios**:
- `web-server`: Nginx (nginx:latest)
- `sftp-server`: SFTP (atmoz/sftp)

**Puertos**:
- 80:80 y 8080:80 - HTTP (redirige a HTTPS)
- 8443:443 - HTTPS
- 2222:22 - SFTP

**Volumenes**:
- `web_data:/usr/share/nginx/html`
- `./conf/default.conf:/etc/nginx/conf.d/default.conf`
- `./conf/gzip.conf:/etc/nginx/conf.d/gzip.conf`
- `./nginx-selfsigned.crt:/etc/ssl/certs/nginx-selfsigned.crt`
- `./nginx-selfsigned.key:/etc/ssl/private/nginx-selfsigned.key`

#### Configuracion Nginx

- **Ubicacion**: practica-nginx/conf/default.conf
- **Server blocks**: HTTP (80) redirige, HTTPS (443) sirve
- **Root**: /usr/share/nginx/html
- **Locations**: /, /reloj, /admin/

#### Seguridad

- Certificados SSL autofirmados
- HTTPS en puerto 8443
- Redirect HTTP a HTTPS con 301
- Compresion Gzip activada
- Autenticacion HTTP Basic en /admin/

---

### h) Ajustes necesarios para implantacion de aplicaciones

#### Despliegue de /reloj

Al desplegar una aplicacion en un subdirectorio como `/reloj`, las rutas relativas (`./style.css`) funcionan correctamente. Las rutas absolutas desde raiz (`/style.css`) fallarian porque buscarian en la raiz del servidor.

La configuracion usa `alias` para mapear `/reloj` a la carpeta fisica correspondiente.

#### Problema tipico de permisos SFTP

Al subir archivos via SFTP, pueden crearse con permisos restrictivos causando errores 403. El usuario SFTP puede crear archivos con UID/GID que no coincide con Nginx.

**Solucion**: Configurar usuario SFTP con UID correcto en docker-compose.yml y ajustar permisos con chmod/chown tras la subida.

#### Evidencias
- [c1-01-root.png](evidencias/c1-01-root.png) - / funciona
- [clock.png](evidencias/clock.png) - /reloj funciona

---

### i) Virtualizacion en despliegue

#### Diferencia operativa

**Instalacion nativa en SO**:
- Nginx instalado directamente (`apt install nginx`)
- Configuracion en `/etc/nginx/` modificada directamente
- Cambios persistentes en el filesystem
- Dificulta portabilidad y puede generar conflictos

**Contenedor efimero + volumenes**:
- Nginx en contenedor Docker aislado
- Configuracion montada desde el host mediante volumenes
- Contenedor efimero: puede destruirse y recrearse sin perder configuracion
- Portabilidad, aislamiento, versionado facil, reproducibilidad

#### Evidencias
- [i-01-compose-ps.png](evidencias/i-01-compose-ps.png) - Servicios activos con puertos y estado Up

---

### j) Logs: monitorizacion, consolidacion y analisis

#### Generacion de trafico

```powershell
1..20 | ForEach-Object { curl.exe -s http://localhost:8080/ > $null }
1..10 | ForEach-Object { curl.exe -s http://localhost:8080/no-existe-$_ > $null }
```

#### Monitorizacion en tiempo real

```bash
docker compose logs -f web-server
```

#### Extraccion de metricas

```bash
# Top URLs
docker exec mi-servidor-nginx sh -c 'awk "{print $7}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'

# Codigos de estado
docker exec mi-servidor-nginx sh -c 'awk "{print $9}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'

# URLs con 404
docker exec mi-servidor-nginx sh -c 'awk "$9==404 {print $7}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'
```

#### Evidencias
- [j-01-logs-follow.png](evidencias/j-01-logs-follow.png) - Monitorizacion en tiempo real

---

## Checklist Final

### Parte 1: Configuracion basica
- [x] 1. Docker y Docker Compose instalados
- [x] 2. Contenedor Nginx funcionando
- [x] 3. Contenedor SFTP funcionando
- [x] 4. Puertos mapeados (80, 8080, 8443, 2222)
- [x] 5. Volumen web_data compartido
- [x] 6. Certificado SSL autofirmado
- [x] 7. Web principal en /
- [x] 8. Aplicacion reloj en /reloj
- [x] 9. HTTPS en puerto 8443
- [x] 10. Redirect HTTP a HTTPS
- [x] 11. SFTP funcional

### Parte 2: RA2 a-j

| Criterio | Completado | Evidencias |
|----------|------------|------------|
| a) Parametros administracion | [x] | [nginxconfDirectiva.png](evidencias/nginxconfDirectiva.png), [02_nginx-t.png](evidencias/02_nginx-t.png), [configuracionCargada.png](evidencias/configuracionCargada.png) |
| b) Gzip + Modulo investigado | [x] | [b1-01_gzipconf.png](evidencias/b1-01_gzipconf.png), [b1-02_compose_volume_gzip.png](evidencias/b1-02_compose_volume_gzip.png), [b1-03_nginx-t.png](evidencias/b1-03_nginx-t.png), [b1-04_curl_gzip.png](evidencias/b1-04_curl_gzip.png) |
| c) Multi-sitio por path | [x] | [c1-01-root.png](evidencias/c1-01-root.png), [clock.png](evidencias/clock.png), [c1-03-defaultconf.png](evidencias/c1-03-defaultconf.png) |
| d) Autenticacion /admin | [x] | [d-01-admin-html.png](evidencias/d-01-admin-html.png), [d-02-defaultconf_auth.png](evidencias/d-02-defaultconf_auth.png), [d-03curl-401.png](evidencias/d-03curl-401.png), [d-04curl-200.png](evidencias/d-04curl-200.png) |
| e) Certificados digitales | [x] | [e-01-ls-certs.png](evidencias/e-01-ls-certs.png), [e-02-compose-certs.png](evidencias/e-02-compose-certs.png), [e-03-defaultconf-ssl.png](evidencias/e-03-defaultconf-ssl.png) |
| f) HTTPS y redirect | [x] | [f-01-https.png](evidencias/f-01-https.png), [f-02-301-network.png](evidencias/f-02-301-network.png) |
| g) Documentacion | [x] | Este documento |
| h) Implantacion apps | [x] | [c1-01-root.png](evidencias/c1-01-root.png), [clock.png](evidencias/clock.png) |
| i) Virtualizacion | [x] | [i-01-compose-ps.png](evidencias/i-01-compose-ps.png) |
| j) Logs y metricas | [x] | [j-01-logs-follow.png](evidencias/j-01-logs-follow.png) |

---

## Estructura del Repositorio

```
Despliegue-de-una-web-statica/
├── DESPLIEGUE.md
├── evidencias/
└── practica-nginx/
    ├── docker-compose.yml
    ├── openssl.cnf
    ├── nginx-selfsigned.crt
    ├── nginx-selfsigned.key
    ├── .htpasswd
    ├── conf/
    │   ├── default.conf
    │   ├── gzip.conf
    │   └── timeouts.conf
    └── webdata/
        └── admin/
            └── index.html
```
