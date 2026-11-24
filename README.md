# ACI606 - Docker Rescue Challenge

Recuperación técnica de una aplicación web multi-servicio entregada con fallas deliberadas de construcción, dependencias, red, puertos, credenciales y persistencia.

La solución final se inicia con un único comando Docker Compose, conserva los datos de PostgreSQL mediante un volumen nombrado y expone comprobaciones de salud para controlar el orden de inicio.

## Arquitectura final

```mermaid
flowchart LR
    U["Usuario :8080"] --> F["Frontend Nginx :80"]
    F -->|/api| B["Backend FastAPI :8000"]
    B --> D[("PostgreSQL :5432")]
    D --- V[("Volumen pgdata")]
```

Los contenedores comparten la red predeterminada de Compose. Nginx utiliza el nombre `backend` y FastAPI utiliza el nombre `db`; los puertos internos no dependen de `localhost` ni del puerto publicado en el host.

Consulta los diagramas y la explicación completa en [docs/ARQUITECTURA.md](docs/ARQUITECTURA.md).

## Requisitos

- Docker Engine.
- Docker Compose v2.
- GitHub Codespaces o un equipo con Docker disponible.
- Puerto 8080 libre, salvo que se configure otro mediante `APP_PORT`.

## Inicio rápido

Desde la raíz del repositorio:

```bash
docker compose up --build -d
docker compose ps
```

Compose utiliza valores predeterminados, por lo que este comando también funciona desde un clon o Codespace limpio sin crear previamente un archivo `.env`.

Resultado esperado:

- `db`: `Up ... (healthy)`.
- `backend`: `Up ... (healthy)`.
- `frontend`: `Up` y publicado en `8080 -> 80`.

En GitHub Codespaces, abre la pestaña **Ports** y utiliza la URL correspondiente al puerto 8080.

## Configuración

Las variables disponibles están documentadas en `.env.example`:

| Variable | Valor predeterminado | Uso |
|---|---:|---|
| `APP_PORT` | `8080` | Puerto publicado para el frontend. |
| `POSTGRES_USER` | `appuser` | Usuario de PostgreSQL. |
| `POSTGRES_PASSWORD` | `rescue123` | Contraseña de laboratorio. |
| `POSTGRES_DB` | `appdb` | Base de datos utilizada por la API. |

Para personalizarlas:

```bash
cp .env.example .env
```

Edita `.env` y vuelve a aplicar la configuración:

```bash
docker compose up -d --force-recreate
```

`.env` está excluido mediante `.gitignore`. No debe subirse al repositorio.

## Verificación funcional

### Estado de los servicios

```bash
docker compose ps
```

### Salud de la aplicación completa

```bash
curl -i http://localhost:8080/api/health
```

Respuesta esperada:

```json
{"status":"ok","database":"reachable"}
```

### Crear un registro

```bash
curl -i -X POST http://localhost:8080/api/items \
  -H "Content-Type: application/json" \
  -d '{"name":"registro-de-prueba"}'
```

### Consultar los registros

```bash
curl http://localhost:8080/api/items
```

### Consultar PostgreSQL directamente

```bash
docker compose exec db sh -c 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c "SELECT id, name, created_at FROM items ORDER BY id;"'
```

## Persistencia

PostgreSQL monta el volumen nombrado `pgdata` en `/var/lib/postgresql/data`.

```bash
docker volume ls
docker volume inspect catedra-2-especialidad_pgdata
```

Para recrear PostgreSQL sin eliminar los datos:

```bash
docker compose up -d --force-recreate db
docker compose exec db pg_isready -U appuser -d appdb
curl http://localhost:8080/api/items
```

El proyecto ya contiene evidencia de que el mismo registro permaneció después de cambiar el contenedor de PostgreSQL.

## Detener y recrear

Detener los servicios sin eliminarlos:

```bash
docker compose stop
```

Iniciarlos nuevamente:

```bash
docker compose start
```

Eliminar contenedores y red, conservando el volumen nombrado:

```bash
docker compose down
```

Recrear toda la solución:

```bash
docker compose up --build -d
docker compose ps
```

> `docker compose down -v` elimina también `pgdata` y sus registros. Úsalo únicamente cuando se requiera reiniciar deliberadamente la base de datos.

## Fallo controlado y recuperación

Detener el backend:

```bash
docker compose stop backend
docker compose ps -a
curl -i http://localhost:8080/api/health
docker compose logs backend
```

La página estática de Nginx puede seguir disponible, pero `/api/health` deja de responder correctamente porque la API está detenida.

Recuperar el servicio:

```bash
docker compose start backend
docker compose ps
curl -i http://localhost:8080/api/health
```

Espera hasta que el backend figure como `healthy` antes de validar la recuperación.

## Diagnóstico

```bash
docker ps -a
docker compose ps
docker compose logs <servicio>
docker compose config
docker inspect <contenedor>
docker exec -it <contenedor> sh
docker network ls
docker network inspect catedra-2-especialidad_default
docker volume ls
docker volume inspect catedra-2-especialidad_pgdata
```

La secuencia aplicada para cada reparación fue:

```text
síntoma → evidencia → causa raíz → corrección → verificación
```

## Documentación y evidencias

- [Bitácora de reparaciones](docs/BITACORA_REPARACIONES.md)
- [Arquitectura inicial y final](docs/ARQUITECTURA.md)
- [Plan de pruebas de validación](docs/PRUEBAS_VALIDACION.md)
- [Evidencias del diagnóstico inicial](evidencias/diagnostico_inicial)
- [Evidencias de mejoras obligatorias](evidencias/mejoras_obligatorias)

## Estructura del repositorio

```text
.
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── frontend/
│   ├── Dockerfile
│   ├── index.html
│   └── nginx.conf
├── docs/
├── evidencias/
├── .env.example
├── .gitignore
├── compose.yaml
└── README.md
```

## Estado final

La solución recuperada cumple con comunicación entre servicios, configuración externalizada, healthchecks, persistencia explícita, observabilidad mediante logs y ejecución reproducible con Docker Compose.