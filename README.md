# 🔐 Keycloak Identity Server (Dockerized)

Infraestructura de **Gestión de Identidad y Acceso (IAM)** basada en Keycloak y PostgreSQL, lista para producción. Diseñada para integrarse como un servicio "drop-in" en nuestro [Reverse Proxy Docker](https://github.com/StudentUCO/reverse-proxy-docker).

## 🚀 Características

- **Producción Ready:** Configuración optimizada (modo `edge`, métricas activas).
- **Persistencia Robusta:** Volumen nombrado para datos de PostgreSQL (`keycloak_postgres_data_prod`).
- **Seguridad:** 
  - Red interna aislada para la base de datos.
  - Sin exposición de puertos al host (solo accesible vía Proxy).
  - Variables de entorno para secretos.
- **Integración Transparente:** Se conecta automáticamente a la red `proxy-shared-network`.

---

## 🛠️ Despliegue Paso a Paso

### 1. Prerrequisitos
- Docker y Docker Compose instalados.
- El [Reverse Proxy](https://github.com/StudentUCO/reverse-proxy-docker) debe estar corriendo (necesita la red `proxy-shared-network`).
- Un subdominio apuntando a tu servidor (ej: `auth.tudominio.com`).

### 2. Instalación

Clona este repositorio:

```bash
git clone https://github.com/StudentUCO/keycloak-docker-deploy.git
cd keycloak-docker-deploy
cp .env.example .env
```

### 3. Configuración

Edita `.env` con tus credenciales seguras:

```ini
KEYCLOAK_HOSTNAME=auth.tudominio.com
KEYCLOAK_ADMIN_PASSWORD=<generar-password-seguro>
KEYCLOAK_DB_PASSWORD=<generar-password-bd>
```

### 4. Iniciar Servicio

```bash
docker-compose up -d
```

Keycloak iniciará pero **no será accesible aún** hasta que configures el Proxy.

---

## 🔗 Integración con Reverse Proxy

Para exponer Keycloak, debes agregar un archivo de configuración en tu repositorio **reverse-proxy-docker**.

1. Ve a tu repositorio `reverse-proxy-docker`.
2. Crea el archivo `nginx/sites-enabled/keycloak.conf`.
3. Pega el siguiente contenido (ajustando `server_name`):

```nginx
server {
    listen 80;
    server_name auth.tudominio.com; # Tu subdominio
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl http2;
    server_name auth.tudominio.com; # Tu subdominio

    ssl_certificate /etc/letsencrypt/live/tudominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/tudominio.com/privkey.pem;
    
    include /etc/nginx/includes/security.conf;
    include /etc/nginx/includes/ssl_params.conf;

    location / {
        # Nombre del servicio definido en docker-compose de Keycloak
        proxy_pass http://keycloak-server:8080;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
}
```

4. Recarga el proxy:
   ```bash
   docker exec reverse-proxy nginx -s reload
   ```

### 5. Generar Certificado SSL
Si es un subdominio nuevo, recuerda generar el certificado en el repo del proxy:

```bash
# Desde la carpeta reverse-proxy-docker
docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  -d auth.tudominio.com
```

---

## 🛡️ Mantenimiento

### Backups
La base de datos se almacena en el volumen `keycloak_postgres_data_prod`. Para hacer un backup:

```bash
docker exec -t keycloak-db-server pg_dumpall -c -U keycloak > dump_$(date +%Y-%m-%d).sql
```

### Actualización
Para actualizar la versión de Keycloak:
1. Cambia el tag en `docker-compose.yml`.
2. Ejecuta `docker-compose pull && docker-compose up -d`.
