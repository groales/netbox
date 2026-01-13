# NetBox

IPAM (IP Address Management) y DCIM (Data Center Infrastructure Management) de código abierto. Solución completa para documentar y gestionar infraestructura de red.

## Características

- 📊 **IPAM completo**: Gestión de direcciones IP, VLANs, VRFs
- 🖥️ **DCIM**: Inventario de racks, dispositivos, cables
- 🔌 **Gestión de circuitos**: Proveedores, circuitos, conexiones
- 📝 **Documentación**: Custom fields, tags, journaling
- 🔗 **API REST**: Integración completa con sistemas externos
- 🔐 **Multi-tenancy**: Soporte para múltiples organizaciones
- 📈 **Reportes**: Visualización y exportación de datos
- 🔄 **Webhooks**: Automatización y notificaciones

## Requisitos Previos

- Docker Engine instalado
- Docker Compose instalado
- **Para Traefik o NPM**: Red Docker `proxy` creada
- **Dominio configurado**: Para acceso HTTPS
- **Contraseñas generadas**: DB_PASSWORD, REDIS_PASSWORD y SUPERUSER_PASSWORD

⚠️ **IMPORTANTE**: NetBox requiere PostgreSQL 18 y Redis. Este compose incluye ambos contenedores.

### Sobre Redis

Redis se utiliza como **caché de sesiones y tareas**:
- ✅ **tmpfs (RAM)**: Más rápido, no usa disco
- 🔒 **Contraseña requerida**: Seguridad defense-in-depth
- ⚡ **No persistente**: El caché se regenera automáticamente al reiniciar
- 💾 **No requiere backup**: Solo almacena datos temporales

## Archivos de este Repositorio

Este repositorio contiene archivos de ejemplo:
- `docker-compose.yml` - Configuración base de los contenedores
- `.env.example` - Plantilla de variables de entorno
- `docker-compose.override.traefik.yml.example` - Labels para Traefik
- `README.md` - Esta documentación

> 💡 **Tip**: Puedes copiar estos archivos manualmente o clonar el repositorio.

---

## Generar Contraseñas

**Antes de cualquier despliegue**, genera contraseñas seguras:

```bash
# DB_PASSWORD (PostgreSQL)
openssl rand -base64 32

# REDIS_PASSWORD
openssl rand -base64 32

# SUPERUSER_PASSWORD (admin de NetBox)
openssl rand -base64 32
```

Guarda los resultados, los necesitarás en el archivo `.env`.

> ⚠️ **Importante**: Usa comillas simples en el archivo `.env` si las contraseñas contienen caracteres especiales.
> Ejemplo: `DB_PASSWORD='tu_password_generado'`

---

## Despliegue con Docker Compose

### 1. Crear Directorio y Archivos

```bash
# Crear directorio
mkdir netbox
cd netbox
```

### 2. Crear docker-compose.yml

Crea el archivo `docker-compose.yml`:

```yaml
services:
  netbox:
    container_name: netbox
    image: lscr.io/linuxserver/netbox:latest
    restart: unless-stopped
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Madrid
      SUPERUSER_EMAIL: ${SUPERUSER_EMAIL}
      SUPERUSER_PASSWORD: ${SUPERUSER_PASSWORD}
      ALLOWED_HOST: ${ALLOWED_HOST:-*}
      DB_NAME: ${DB_NAME:-netbox}
      DB_USER: ${DB_USER:-netbox}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_HOST: netbox-db
      DB_PORT: 5432
      REDIS_HOST: netbox-redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    volumes:
      - netbox_config:/config
    networks:
      - proxy
      - default
    depends_on:
      - netbox-db
      - netbox-redis

  netbox-db:
    container_name: netbox-db
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-netbox}
      POSTGRES_USER: ${DB_USER:-netbox}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - netbox_db:/var/lib/postgresql

  netbox-redis:
    container_name: netbox-redis
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    tmpfs:
      - /data:rw,noexec,nosuid,size=256m

volumes:
  netbox_config:
    name: netbox_config
  netbox_db:
    name: netbox_db

networks:
  default:
  proxy:
    external: true
```

### 3. Configurar Variables de Entorno

Crea el archivo `.env`:

```env
# Base de datos
DB_NAME=netbox
DB_USER=netbox
DB_PASSWORD=tu_password_generado_1

# Redis
REDIS_PASSWORD=tu_password_generado_2

# Superusuario NetBox
SUPERUSER_EMAIL=admin@example.com
SUPERUSER_PASSWORD=tu_password_generado_3

# Hosts permitidos (opcional)
ALLOWED_HOST=*
```

### 4. (Opcional) Configurar Traefik

Si usas Traefik, crea `docker-compose.override.yml`:

```yaml
services:
  netbox:
    labels:
      - traefik.enable=true
      - traefik.http.routers.netbox-http.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.netbox-http.entrypoints=web
      - traefik.http.routers.netbox-http.middlewares=redirect-to-https
      - traefik.http.routers.netbox.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.netbox.entrypoints=websecure
      - traefik.http.routers.netbox.tls.certresolver=letsencrypt
      - traefik.http.services.netbox.loadbalancer.server.port=8000
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
      - traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true
```

Y añade en `.env`:
```env
DOMAIN_HOST=netbox.dominio.com
```

### 5. Desplegar

```bash
# Crear red proxy si no existe
docker network create proxy

# Iniciar servicios
docker compose up -d

# Ver logs
docker compose logs -f netbox
```

La inicialización puede tardar **60-90 segundos** (PostgreSQL + Redis + NetBox migrations).

---

## Método Alternativo: Clonar desde Git

Si prefieres usar Git para mantener la configuración actualizada:

```bash
# Clonar repositorio
git clone https://git.ictiberia.com/groales/netbox.git
cd netbox

# Copiar y editar variables
cp .env.example .env
nano .env

# Para Traefik
cp docker-compose.override.traefik.yml.example docker-compose.override.yml

# Desplegar
docker network create proxy
docker compose up -d
```

---

## Verificar el Despliegue

```bash
# Ver logs en tiempo real
docker compose logs -f netbox

# Verificar contenedores activos
docker compose ps

# Comprobar base de datos
docker compose exec netbox-db psql -U netbox -d netbox -c '\dt'
```

**Crear directorio de medios** (necesario para subir imágenes):

```bash
docker compose exec netbox mkdir -p /config/media
docker compose exec netbox chown 1000:1000 /config/media
```

**Acceso Inicial**:

Una vez desplegado, accede a NetBox:
- Traefik: `https://netbox.dominio.com`
- NPM: Configura el proxy host en NPM apuntando a `netbox` puerto `8000`
- Standalone: `http://IP_SERVIDOR:8000`

**Credenciales iniciales**:
- Usuario: `admin` (⚠️ **NO** es el email, es el nombre de usuario)
- Contraseña: La configurada en `SUPERUSER_PASSWORD`

---

## Comandos Útiles

### Ver Logs
```bash
docker compose logs -f netbox
```

### Reiniciar Servicio
```bash
docker compose restart netbox
```

### Actualizar Contenedor
```bash
docker compose pull
docker compose up -d
```

### Backup de Base de Datos
```bash
docker compose exec netbox-db pg_dump -U netbox netbox > netbox_backup.sql
```

---

## Ejemplos de Compose Completos

### Con Traefik

```yaml
services:
  netbox:
    container_name: netbox
    image: lscr.io/linuxserver/netbox:latest
    restart: unless-stopped
    environment:
      PUID: 1000
      PGID: 1000
      TZ: Europe/Madrid
      SUPERUSER_EMAIL: ${SUPERUSER_EMAIL}
      SUPERUSER_PASSWORD: ${SUPERUSER_PASSWORD}
      ALLOWED_HOST: ${ALLOWED_HOST:-*}
      DB_NAME: ${DB_NAME:-netbox}
      DB_USER: ${DB_USER:-netbox}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_HOST: netbox-db
      DB_PORT: 5432
      REDIS_HOST: netbox-redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    volumes:
      - netbox_config:/config
    networks:
      - proxy
      - netbox-internal
    depends_on:
      - netbox-db
      - netbox-redis
    labels:
      - traefik.enable=true
      - traefik.http.routers.netbox-http.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.netbox-http.entrypoints=web
      - traefik.http.routers.netbox-http.middlewares=redirect-to-https
      - traefik.http.routers.netbox.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.netbox.entrypoints=websecure
      - traefik.http.routers.netbox.tls.certresolver=letsencrypt
      - traefik.http.services.netbox.loadbalancer.server.port=8000
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
      - traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true

  netbox-db:
    container_name: netbox-db
    image: postgres:18-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-netbox}
      POSTGRES_USER: ${DB_USER:-netbox}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - netbox_db:/var/lib/postgresql
    networks:
      - netbox-internal

  netbox-redis:
    container_name: netbox-redis
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    tmpfs:
      - /data:rw,noexec,nosuid,size=256m
    networks:
      - netbox-internal

volumes:
  netbox_config:
    name: netbox_config
  netbox_db:
    name: netbox_db

networks:
  proxy:
    external: true
  netbox-internal:
    name: netbox-internal
```

### Nginx Proxy Manager (NPM)

**Requisitos**:
- NPM desplegado y accesible
- Red `proxy` creada
- DNS apuntando al servidor

**Pasos**:

1. Despliega el stack con el `docker-compose.yml` base (sin override)

2. En NPM, crea un nuevo **Proxy Host**:
   - **Domain Names**: `netbox.tudominio.com`
   - **Scheme**: `http`
   - **Forward Hostname / IP**: `netbox`
   - **Forward Port**: `8000`
   - **Cache Assets**: ✅ Activado
   - **Block Common Exploits**: ✅ Activado
   - **Websockets Support**: ✅ Activado

3. En la pestaña **SSL**:
   - **SSL Certificate**: Request a new SSL Certificate (Let's Encrypt)
   - **Force SSL**: ✅ Activado
   - **HTTP/2 Support**: ✅ Activado
   - **HSTS Enabled**: ✅ Activado (opcional)

4. Guarda y accede a `https://netbox.tudominio.com`

---

## Configuración Inicial

### Primer Acceso

1. Accede a NetBox usando tu dominio configurado
2. Haz login con:
   - **Username**: `admin`
   - **Password**: El configurado en `SUPERUSER_PASSWORD`
3. NetBox creará automáticamente la base de datos en el primer inicio (puede tardar 1-2 minutos)

> ℹ️ **Nota**: El username es siempre `admin`, el email (`SUPERUSER_EMAIL`) se usa solo para notificaciones.

### Panel de Administración

1. Ve a **Admin** (esquina superior derecha) → **Admin Panel**
2. Configura los parámetros básicos:

#### Users & Groups
- Crea usuarios adicionales en **Users** → **Add**
- Define grupos y permisos en **Groups**
- Asigna tokens de API en **Tokens**

#### Sites & Racks
- Crea sitios (locations) en **DCIM** → **Sites**
- Define racks y ubicaciones físicas
- Configura regiones y tenant groups

#### IP Management
- Define prefijos de red en **IPAM** → **Prefixes**
- Crea VLANs en **IPAM** → **VLANs**
- Configura VRFs si usas routing avanzado

#### Device Types
- Importa tipos de dispositivos desde [NetBox Device Type Library](https://github.com/netbox-community/devicetype-library)
- O crea tipos personalizados en **Device Types**

---

## Personalización

### Configuración Avanzada

NetBox almacena su configuración en `/config/` dentro del contenedor. Puedes personalizarla:

```bash
# Acceder al contenedor
docker exec -it netbox bash

# Editar configuración
nano /config/configuration.py
```

**Configuraciones comunes**:

```python
# Banner personalizado
BANNER_TOP = 'NetBox Producción - ICT Iberia'
BANNER_BOTTOM = ''

# Preferencias de sesión
SESSION_COOKIE_AGE = 1209600  # 2 semanas

# Paginación
PAGINATE_COUNT = 50

# Tiempo de expiración de tokens
AUTH_TOKEN_TIMEOUT = 60 * 60 * 24 * 7  # 7 días
```

Reinicia el contenedor después de cambios: `docker restart netbox`

### Plugins

NetBox soporta plugins para funcionalidad extendida. Instálalos en `/config/plugins/`:

```bash
# Ejemplo: NetBox Topology Views
docker exec -it netbox pip install netbox-topology-views

# Añadir a configuration.py
PLUGINS = ['netbox_topology_views']

# Reiniciar
docker restart netbox
```

### LDAP / SSO

Configura autenticación LDAP editando `/config/ldap_config.py`:

```python
import ldap
from django_auth_ldap.config import LDAPSearch

AUTH_LDAP_SERVER_URI = "ldap://ldap.example.com"
AUTH_LDAP_BIND_DN = "CN=netbox,OU=Services,DC=example,DC=com"
AUTH_LDAP_BIND_PASSWORD = "password"
AUTH_LDAP_USER_SEARCH = LDAPSearch(
    "OU=Users,DC=example,DC=com",
    ldap.SCOPE_SUBTREE,
    "(sAMAccountName=%(user)s)"
)
```

---

## Backup y Restauración

### Backup Manual

```bash
# Backup de PostgreSQL
docker exec netbox-db pg_dump -U netbox netbox > netbox-backup-$(date +%Y%m%d).sql

# Backup de configuración
docker run --rm -v netbox_config:/backup -v $(pwd):/target alpine tar czf /target/netbox-config-$(date +%Y%m%d).tar.gz -C /backup .

# Redis usa tmpfs (no requiere backup - solo caché)
```

### Backup Automático

Crea un script en `/root/backup-netbox.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backups/netbox"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

# PostgreSQL
docker exec netbox-db pg_dump -U netbox netbox | gzip > $BACKUP_DIR/netbox-db-$DATE.sql.gz

# Configuración
docker run --rm -v netbox_config:/backup -v $BACKUP_DIR:/target alpine tar czf /target/netbox-config-$DATE.tar.gz -C /backup .

# Limpiar backups antiguos (mantener 7 días)
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completado: $DATE"
```

Programa con cron:

```bash
chmod +x /root/backup-netbox.sh
crontab -e

# Backup diario a las 2 AM
0 2 * * * /root/backup-netbox.sh
```

### Restauración

```bash
# Detener NetBox
docker stop netbox

# Restaurar PostgreSQL
gunzip < netbox-db-20250101.sql.gz | docker exec -i netbox-db psql -U netbox netbox

# Restaurar configuración
docker run --rm -v netbox_config:/restore -v $(pwd):/source alpine tar xzf /source/netbox-config-20250101.tar.gz -C /restore

# Iniciar NetBox
docker start netbox
```

---

## Actualización

### Actualizar NetBox

```bash
# 1. Backup ANTES de actualizar
docker exec netbox-db pg_dump -U netbox netbox > netbox-pre-update-$(date +%Y%m%d).sql

# 2. Detener stack
docker stop netbox netbox-redis netbox-db

# 3. Actualizar imágenes
docker pull lscr.io/linuxserver/netbox:latest
docker pull postgres:18-alpine
docker pull redis:7-alpine

# 4. Iniciar stack
docker start netbox-db netbox-redis netbox

# 5. Verificar logs
docker logs -f netbox

# 6. Verificar versión en Admin Panel
```

### Actualizar PostgreSQL

Si necesitas actualizar de PostgreSQL 16 a 18 (ya está en 18):

```bash
# 1. Backup
docker exec netbox-db pg_dump -U netbox netbox > netbox-pg-migration.sql

# 2. Detener y eliminar contenedor antiguo
docker stop netbox-db
docker rm netbox-db

# 3. Eliminar volumen antiguo
docker volume rm netbox_db

# 4. Recrear con PostgreSQL 18
docker compose up -d netbox-db

# 5. Restaurar datos
cat netbox-pg-migration.sql | docker exec -i netbox-db psql -U netbox netbox

# 6. Iniciar NetBox
docker start netbox
```

---

## Solución de Problemas

### NetBox no inicia

**Síntomas**: Contenedor se reinicia constantemente

**Diagnóstico**:
```bash
docker logs netbox
```

**Soluciones**:
- Verificar que PostgreSQL esté funcionando: `docker logs netbox-db`
- Verificar que Redis esté funcionando: `docker logs netbox-redis`
- Comprobar contraseñas en `.env`
- Verificar permisos del volumen: `docker exec netbox ls -la /config`

### Error de conexión a base de datos

**Síntomas**: `could not connect to server: Connection refused`

**Solución**:
```bash
# Verificar que la BD esté lista
docker exec netbox-db pg_isready -U netbox

# Reiniciar servicios en orden
docker restart netbox-db
sleep 5
docker restart netbox-redis
sleep 5
docker restart netbox
```

### Error de permisos

**Síntomas**: `PermissionError: [Errno 13] Permission denied`

**Solución**:
```bash
docker run --rm -v netbox_config:/data alpine chown -R 1000:1000 /data
docker restart netbox
```

### NetBox lento o no responde

**Diagnóstico**:
```bash
# Ver uso de recursos
docker stats netbox netbox-db netbox-redis

# Ver queries lentas en PostgreSQL
docker exec netbox-db psql -U netbox -c "SELECT query, calls, total_time FROM pg_stat_statements ORDER BY total_time DESC LIMIT 10;"
```

**Soluciones**:
- Incrementar memoria de Redis
- Optimizar índices en PostgreSQL
- Limpiar sesiones antiguas en Django admin

### Comandos de emergencia

```bash
# Reiniciar todo el stack
docker restart netbox netbox-redis netbox-db

# Ver logs en tiempo real
docker logs -f --tail 100 netbox

# Acceder a shell de NetBox
docker exec -it netbox bash

# Ejecutar comandos de Django
docker exec netbox python /app/netbox/manage.py shell

# Limpiar caché
docker exec netbox-redis redis-cli --pass "TU_REDIS_PASSWORD" FLUSHALL

# Reconstruir búsqueda
docker exec netbox python /app/netbox/manage.py reindex --lazy
```

---

## Variables de Entorno

### Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------||
| `DB_PASSWORD` | Contraseña de PostgreSQL | `generada_con_openssl` |
| `REDIS_PASSWORD` | Contraseña de Redis | `generada_con_openssl` |
| `SUPERUSER_EMAIL` | Email del admin (para notificaciones) | `admin@example.com` |
| `SUPERUSER_PASSWORD` | Contraseña del usuario `admin` | `generada_con_openssl` |

### Opcionales

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `DB_NAME` | Nombre de la base de datos | `netbox` |
| `DB_USER` | Usuario de PostgreSQL | `netbox` |
| `ALLOWED_HOST` | Hosts permitidos | `*` |
| `DOMAIN_HOST` | Dominio (solo Traefik) | - |
| `PUID` / `PGID` | UID/GID del usuario | `1000` |
| `TZ` | Zona horaria | `Europe/Madrid` |

---

## Recursos

- [Documentación oficial de NetBox](https://docs.netbox.dev/)
- [LinuxServer NetBox Image](https://docs.linuxserver.io/images/docker-netbox)
- [NetBox Device Type Library](https://github.com/netbox-community/devicetype-library)
- [NetBox Plugins](https://github.com/netbox-community/netbox/wiki/Plugins)
- [API Documentation](https://demo.netbox.dev/api/docs/)

---

## Licencia

NetBox es software de código abierto bajo licencia Apache 2.0.
