# Pruebas de validación

Este documento reúne el procedimiento mínimo exigido para la entrega. Las pruebas deben ejecutarse desde la raíz del repositorio y sus salidas relevantes deben conservarse en `evidencias/validacion_final/`.

## 1. Arranque limpio

```bash
docker compose down
docker compose up --build -d
docker compose ps
```

Criterio de aceptación:

- Los tres servicios aparecen en operación.
- PostgreSQL y backend figuran como `healthy`.
- El frontend publica `${APP_PORT:-8080}:80`.

## 2. Comunicación frontend → backend → PostgreSQL

```bash
curl -i http://localhost:8080/api/health
curl -i -X POST http://localhost:8080/api/items \
  -H "Content-Type: application/json" \
  -d '{"name":"validacion-final"}'
curl http://localhost:8080/api/items
```

Criterio de aceptación:

- `/api/health` devuelve HTTP 200 y `database: reachable`.
- El POST devuelve HTTP 201.
- El registro creado aparece en la consulta posterior.

## 3. Persistencia

```bash
curl http://localhost:8080/api/items
docker compose ps -q db
docker compose up -d --force-recreate db
docker compose exec db pg_isready -U appuser -d appdb
docker compose ps -q db
curl http://localhost:8080/api/items
```

Criterio de aceptación:

- El ID del contenedor `db` cambia.
- El registro continúa presente.
- El volumen utilizado sigue siendo `catedra-2-especialidad_pgdata`.

Esta prueba ya fue ejecutada. Los archivos `09_datos_antes_recreacion.txt` y `13_datos_despues_recreacion.txt` contienen el mismo registro `evidencia-persistencia`.

## 4. Fallo controlado

```bash
docker compose stop backend
docker compose ps -a
curl -i http://localhost:8080/api/health
docker compose logs --tail=30 backend
```

Criterio de aceptación:

- Backend aparece detenido.
- La ruta de API deja de responder correctamente, normalmente con HTTP 502 desde Nginx.
- Los registros y el estado permiten explicar el alcance del fallo.

## 5. Recuperación

```bash
docker compose start backend
docker compose ps
curl -i http://localhost:8080/api/health
curl http://localhost:8080/api/items
```

Criterio de aceptación:

- Backend vuelve a `healthy`.
- `/api/health` devuelve HTTP 200.
- Los datos anteriores siguen disponibles.

## 6. Inspección técnica complementaria

```bash
docker compose config
docker inspect "$(docker compose ps -q backend)"
docker network inspect catedra-2-especialidad_default
docker volume inspect catedra-2-especialidad_pgdata
```

## Matriz de cobertura

| Requisito | Evidencia principal |
|---|---|
| Arranque con un comando | `docker compose up --build -d` y `docker compose ps`. |
| Comunicación | HTTP 200 de `/api/health`, POST y GET de `/api/items`. |
| Persistencia | Mismo registro antes y después de recrear `db`. |
| Fallo controlado | Backend detenido, estado y respuesta fallida de Nginx. |
| Diagnóstico | Logs, `ps`, `inspect`, red y volumen. |
| Recuperación | Backend saludable, HTTP 200 y datos conservados. |

## Evidencias disponibles

- `evidencias/diagnostico_inicial/`: fallas R-01 a R-06.
- `evidencias/mejoras_obligatorias/`: volumen, externalización y healthchecks.
- `evidencias/validacion_final/`: reservado para arranque limpio, fallo controlado y recuperación final.