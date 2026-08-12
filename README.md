# Infraestructura local compartida

Stack en Docker Compose para levantar de forma rápida y persistente **PostgreSQL**, **Keycloak** y **Redis**. Pensado para que varios proyectos en tu máquina se conecten por `localhost` (o por red Docker si montás otro compose en la misma red).

## Requisitos

- [Docker Engine](https://docs.docker.com/engine/install/) reciente (con plugin Compose v2).
- Puertos libres en el host según la configuración (por defecto **5432**, **8080**, **6379**).
- Contenedores configurados para ejecutarse con usuarios dedicados (no como root), gracias a la opción `user:` en `docker-compose.yml`.

Comprobación rápida:

```bash
docker compose version
```

## Puesta en marcha

1. **Cloná o ubicá** este repositorio donde prefieras (por ejemplo `~/dev/local-infrastructure`).

2. **Variables de entorno** (recomendado en entornos reales):

   El archivo `.env` ya está presente en el repositorio. Simplemente editá sus valores con los datos deseados (usuario, contraseña, puertos, bootstrap admin), sin cambiar la estructura del archivo.

   Por ejemplo, verificá que tenga al menos:

   ```text
   POSTGRES_USER=admin
   POSTGRES_PASSWORD=admin
   KC_BOOTSTRAP_ADMIN_USERNAME=admin
   KC_BOOTSTRAP_ADMIN_PASSWORD=admin
   ```

   Si no querés usar `.env`, Compose igualmente usará los valores por defecto definidos en `docker-compose.yml`, pero para entornos locales conviene personalizar `.env`.

3. **Levantar los servicios**:

   ```bash
   docker compose up -d
   ```

4. **Comprobar que están en marcha**:

   ```bash
   docker compose ps
   ```

   Los tres servicios deberían aparecer como `running` (o `healthy` donde aplique).

5. **Ver logs** (por ejemplo Keycloak al primer arranque tarda un poco en migrar la BD):

   ```bash
   docker compose logs -f keycloak
   ```

   Para todos los servicios: `docker compose logs -f`.

## Servicios y persistencia

| Servicio    | Imagen / rol | Puerto host (por defecto) | Datos persistentes |
|------------|----------------|---------------------------|--------------------|
| `postgres` | PostgreSQL 16  | `5432` (`POSTGRES_PORT`)  | Volumen `postgres_data` |
| `keycloak` | Keycloak 26.x (`start-dev`) | `8080` (`KEYCLOAK_PORT`) | Volumen `keycloak_data` + base `keycloak` en Postgres |
| `redis`    | Redis 8.x      | `6379` (`REDIS_PORT`)     | Volumen `redis_data` (AOF) |

- **PostgreSQL**: los datos viven en el volumen nombrado `postgres_data`. Al **primer** arranque con volumen vacío se ejecutan los scripts en `postgres/init/` (solo entonces); ahí se crea la base **`keycloak`** para que Keycloak no use la misma BD que tus aplicaciones en `POSTGRES_DB`.
- **Keycloak**: guarda estado propio en `keycloak_data` y metadatos de realms/sesiones en Postgres (BD `keycloak`).
- **Redis**: persistencia con **AOF** (`appendonly yes`, `appendfsync everysec`), datos en `redis_data`.

## Variables de entorno

Definilas en `.env` (ya presente en el repo). Las que reconoce este `docker-compose.yml`:

| Variable | Descripción | Valor por defecto en compose |
|----------|-------------|----------------------------|
| `POSTGRES_USER` | Usuario superusuario de Postgres | `postgres` |
| `POSTGRES_PASSWORD` | Contraseña de ese usuario | `postgres` |
| `POSTGRES_DB` | Base de datos creada al iniciar el volumen | `postgres` |
| `POSTGRES_PORT` | Puerto publicado en el host | `5432` |
| `KEYCLOAK_PORT` | Puerto HTTP de Keycloak en el host | `8080` |
| `REDIS_PORT` | Puerto de Redis en el host | `6379` |
| `KC_BOOTSTRAP_ADMIN_USERNAME` | Usuario administrador inicial de Keycloak | `admin` |
| `KC_BOOTSTRAP_ADMIN_PASSWORD` | Contraseña de ese administrador | `admin` |

Keycloak se conecta a Postgres con **el mismo** `POSTGRES_USER` / `POSTGRES_PASSWORD` y a la base **`keycloak`**. Tus apps pueden usar la base `POSTGRES_DB` (por defecto `postgres`) o crear otras bases conectando con el mismo usuario (o creando usuarios dedicados desde `psql`).

## Cómo conectan tus proyectos

### Desde aplicaciones en tu máquina (fuera de Docker)

Usá `localhost` y los puertos que publicaste:

- **PostgreSQL**: `host=localhost`, `port=<POSTGRES_PORT>`, `user=<POSTGRES_USER>`, `password=<POSTGRES_PASSWORD>`, `database=<nombre de BD>` (por ejemplo `postgres`). No uses la BD `keycloak` para datos de aplicación; está reservada para Keycloak.
- **Redis**: URL típica `redis://localhost:<REDIS_PORT>/0` (o el índice de base que uses).
- **Keycloak**: consola de administración en `http://localhost:<KEYCLOAK_PORT>/` (ruta relativa según versión; en Keycloak reciente suele ser `http://localhost:<KEYCLOAK_PORT>/admin/`). Para **OpenID Connect** desde una app en el host, el issuer suele ser `http://localhost:<KEYCLOAK_PORT>/realms/<tu-realm>` (ajustá realm y si usás proxy HTTPS más adelante).

Ejemplo de cadena JDBC:

```text
jdbc:postgresql://localhost:5432/postgres
```

(Sustituí usuario, contraseña y puerto según tu `.env`.)

### Desde otros contenedores Docker

Si otro `docker-compose.yml` está en **otro proyecto**, por defecto **no** comparten red con este stack: desde esos contenedores `localhost` es el propio contenedor, no Postgres de acá.

Opciones habituales:

1. **Publicar puertos** (como ya está): desde el otro compose no podés usar `localhost` al servicio; necesitás `host.docker.internal` (Linux: puede requerir `extra_hosts`) o la IP del host.
2. **Red externa compartida**: creá una red y enlazá ambos proyectos, por ejemplo:

   ```bash
   docker network create local-shared
   ```

   En este repositorio podrías añadir al final de `docker-compose.yml`:

   ```yaml
   networks:
     default:
       name: local-shared
       external: true
   ```

   (Eso implica editar el compose; hacelo solo si querés esa topología fija.)

3. **Incluir este compose como dependencia** con `include` o varios archivos `-f` según tu flujo de trabajo.

La opción más simple para “cualquier proyecto en el host” es conectar por **`localhost` + puertos**.

## Keycloak: primer acceso y entornos

- El modo **`start-dev`** está pensado para **desarrollo local** (HTTP, configuración relajada). No lo uses tal cual como plantilla de producción.
- `KC_BOOTSTRAP_ADMIN_USERNAME` / `KC_BOOTSTRAP_ADMIN_PASSWORD` crean un usuario administrador en el arranque inicial según la [guía de Keycloak sobre bootstrap admin](https://www.keycloak.org/server/bootstrap-admin-recovery). Para políticas de seguridad y cuentas permanentes, consultá la documentación oficial de tu versión.

## Comandos útiles

| Acción | Comando |
|--------|---------|
| Arrancar en segundo plano | `docker compose up -d` |
| Parar sin borrar volúmenes | `docker compose stop` |
| Parar y eliminar contenedores | `docker compose down` |
| Parar y **borrar todos los datos** (Postgres, Keycloak, Redis) | `docker compose down -v` |
| Reiniciar un solo servicio | `docker compose restart redis` |
| Entrar a Postgres | `docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB"` (con variables cargadas desde tu `.env` o sustituyendo a mano) |
| Ping a Redis | `docker compose exec redis redis-cli ping` |

## Reinicializar solo la base de Keycloak o solo Postgres

- **Todo desde cero**: `docker compose down -v` y luego `docker compose up -d`. Se volverán a ejecutar los scripts de `postgres/init/` en un volumen nuevo.
- **Solo Keycloak** (conservar datos de Postgres salvo la BD `keycloak`): es más delicado; lo habitual es borrar el volumen `keycloak_data` y/o recrear la BD `keycloak` con cuidado. Para un entorno local suele ser más simple `docker compose down -v` si no te importa perder el resto de datos de Postgres.

## Solución de problemas

- **Puerto en uso**: cambiá `POSTGRES_PORT`, `KEYCLOAK_PORT` o `REDIS_PORT` en `.env` y volvé a levantar.
- **Keycloak no arranca / reinicios**: revisá `docker compose logs keycloak`; a menudo es esperar a que Postgres esté listo o un error de credenciales (`KC_DB_*` vs `POSTGRES_*`).
- **El script `postgres/init` no corre**: solo corre la **primera vez** que el volumen `postgres_data` está vacío. Si necesitás recrear la BD `keycloak` sin borrar todo el volumen, tendrás que hacerlo manualmente con `psql`.

## Estructura del repositorio

```text
.
├── docker-compose.yml      # Definición de servicios y volúmenes
├── .env                    # Variables de entorno (usuario, contraseñas, puertos)
├── postgres/
│   └── init/
│       └── 01-create-keycloak-db.sql
└── README.md
```

Editá `.env` directamente para personalizar; no subas `.env` con secretos reales a repositorios públicos.
