# Bitácora de reparaciones y mejoras

Cada registro sigue la secuencia exigida: síntoma, evidencia, causa raíz, elemento modificado, reparación, justificación, verificación e impacto.

## R-01 — Módulo ASGI incorrecto

| Campo | Registro |
|---|---|
| Síntoma | El backend terminaba inmediatamente con `Exited (1)`. |
| Evidencia | `docker compose logs backend` informó `Could not import module "app"`. `docker inspect` mostró el comando `uvicorn app:app`. Véanse `07_backend_logs_inicial.txt` y `08_backend_inspect_inicial.txt`. |
| Causa raíz | Uvicorn buscaba un módulo `app.py`, pero el archivo entregado era `main.py` y contenía el objeto `app`. |
| Elemento modificado | `backend/Dockerfile`. |
| Reparación | Se sustituyó `app:app` por `main:app`. |
| Justificación | La sintaxis de Uvicorn es `módulo:objeto`; `main:app` identifica correctamente ambos elementos. |
| Verificación | Desapareció el error de importación y la ejecución avanzó hasta detectar la dependencia PostgreSQL faltante. |
| Impacto | Sin la reparación no existe API y el frontend no puede consultar ni guardar datos. |

## R-02 — Controlador PostgreSQL ausente

| Campo | Registro |
|---|---|
| Síntoma | Después de corregir Uvicorn, el backend seguía en `Exited (1)`. |
| Evidencia | El log informó `ModuleNotFoundError: No module named 'psycopg2'`; véase `12_r01_logs_posteriores.txt`. |
| Causa raíz | SQLAlchemy estaba instalado, pero no el DBAPI requerido para la URL PostgreSQL utilizada. |
| Elemento modificado | `backend/requirements.txt`. |
| Reparación | Se agregó `psycopg2-binary==2.9.10`. |
| Justificación | El controlador permite a SQLAlchemy abrir conexiones PostgreSQL. |
| Verificación | La imagen instaló el paquete y el siguiente log avanzó hasta intentar una conexión real a `localhost:5432`. |
| Impacto | Sin el controlador el backend no puede crear el motor de base de datos. |

## R-03 — Uso incorrecto de localhost hacia PostgreSQL

| Campo | Registro |
|---|---|
| Síntoma | El backend no iniciaba y recibía `Connection refused`. |
| Evidencia | `16_r02_logs_posteriores.txt` muestra intentos contra `localhost` en `::1` y `127.0.0.1`. |
| Causa raíz | Dentro del contenedor backend, `localhost` representa al propio backend, no al contenedor PostgreSQL. |
| Elemento modificado | `DATABASE_URL` en `compose.yaml`. |
| Reparación | Se cambió el host de `localhost` a `db`. |
| Justificación | Compose registra los servicios en DNS interno con su nombre de servicio. |
| Verificación | El siguiente error provino de `db (172.18.0.2):5432`, lo que comprobó la comunicación de red. |
| Impacto | Sin el cambio no existe comunicación backend → PostgreSQL. |

## R-04 — Credenciales inconsistentes

| Campo | Registro |
|---|---|
| Síntoma | PostgreSQL era alcanzable, pero rechazaba el inicio de sesión del backend. |
| Evidencia | `21_r03_logs_posteriores.txt` informó `password authentication failed for user "appuser"`. |
| Causa raíz | `DATABASE_URL` utilizaba `wrongpass` y el servicio `db` estaba inicializado con `rescue123`. |
| Elemento modificado | `DATABASE_URL` en `compose.yaml`. |
| Reparación | Se alineó la contraseña del cliente con la configurada en PostgreSQL. |
| Justificación | Usuario, contraseña y base deben coincidir en ambos extremos. |
| Verificación | `25_r04_logs_posteriores.txt` registra `Application startup complete`; `/health` devolvió `database: reachable`. |
| Impacto | Sin credenciales coherentes PostgreSQL rechaza todas las operaciones de la API. |

## R-05 — Puerto interno incorrecto del frontend

| Campo | Registro |
|---|---|
| Síntoma | El frontend figuraba `Up`, pero `curl` devolvía código `000` y `Connection reset by peer`. |
| Evidencia | `27_r05_puerto_nginx.txt` muestra `listen 80`; Compose publicaba `8080:8080`. `28_r05_prueba_inicial_frontend.txt` registra el fallo. |
| Causa raíz | El puerto 8080 del host apuntaba al puerto 8080 del contenedor, donde Nginx no escuchaba. |
| Elemento modificado | `ports` del servicio `frontend` en `compose.yaml`. |
| Reparación | Se cambió el mapeo a `8080:80` y luego se externalizó como `${APP_PORT:-8080}:80`. |
| Justificación | El segundo puerto del mapeo debe coincidir con `listen 80` de Nginx. |
| Verificación | La página principal respondió HTTP 200. |
| Impacto | Sin la reparación ningún usuario puede acceder a la aplicación web. |

## R-06 — Proxy de Nginx dirigido a localhost

| Campo | Registro |
|---|---|
| Síntoma | La página principal respondía HTTP 200, pero `/api/health` devolvía HTTP 502. |
| Evidencia | `32_r06_logs_frontend_502.txt` registra intentos contra `127.0.0.1:8000` y `::1:8000`. |
| Causa raíz | En el frontend, `localhost:8000` apunta al mismo contenedor Nginx, no a FastAPI. |
| Elemento modificado | `frontend/nginx.conf`. |
| Reparación | Se cambió `proxy_pass` a `http://backend:8000/`. |
| Justificación | El nombre `backend` es resoluble mediante el DNS de la red Compose. La barra final conserva la traducción `/api/health` → `/health`. |
| Verificación | `/api/health` respondió HTTP 200 y `{"status":"ok","database":"reachable"}`. |
| Impacto | Sin la reparación la interfaz abre, pero no puede consumir la API. |

## R-07 — Persistencia explícita

| Campo | Registro |
|---|---|
| Síntoma | PostgreSQL utilizaba un volumen anónimo y Compose no declaraba almacenamiento persistente identificable. |
| Evidencia | `01_volumenes_antes.txt` y `02_montajes_db_antes.txt` muestran un nombre aleatorio. |
| Causa raíz | Ausencia de un volumen nombrado en `compose.yaml`. |
| Elemento modificado | Servicio `db` y sección superior `volumes` en `compose.yaml`. |
| Reparación | Se declaró `pgdata:/var/lib/postgresql/data`. |
| Justificación | El ciclo de vida de `pgdata` es independiente del contenedor PostgreSQL. |
| Verificación | `09_datos_antes_recreacion.txt` y `13_datos_despues_recreacion.txt` contienen el mismo registro y fecha tras cambiar el ID del contenedor. |
| Impacto | Sin persistencia explícita la recreación o limpieza incorrecta puede causar pérdida de datos y dificulta administrar el almacenamiento. |

## R-08 — Configuración externalizada

| Campo | Registro |
|---|---|
| Síntoma | Puerto, usuario, contraseña y base estaban rígidamente escritos en Compose. |
| Evidencia | La revisión inicial de `compose.yaml` y `03_compose_config_inicial.txt` mostró valores fijos. |
| Causa raíz | No existía una interfaz de configuración para distintos entornos. |
| Elemento modificado | `compose.yaml`, `.env.example` y `.gitignore`. |
| Reparación | Se incorporaron `APP_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB`, con valores predeterminados reproducibles. |
| Justificación | La imagen no necesita reconstruirse para cambiar parámetros de ejecución y `.env` permanece fuera de Git. |
| Verificación | `16_r08_env_ignorado.txt` prueba que `.env` está ignorado; `19_r08_health_externalizado.txt` confirma operación correcta. |
| Impacto | Sin externalización, cada entorno exige editar archivos versionados y aumenta el riesgo de publicar configuración privada. |

## R-09 — Healthchecks y orden de inicio

| Campo | Registro |
|---|---|
| Síntoma | `depends_on` solo garantizaba que el proceso se iniciara, no que PostgreSQL o la API estuvieran listos. |
| Evidencia | La configuración inicial expandida mostraba `condition: service_started`. |
| Causa raíz | Ausencia de comprobaciones funcionales y condiciones `service_healthy`. |
| Elemento modificado | Servicios `db`, `backend` y dependencias en `compose.yaml`. |
| Reparación | Se agregó `pg_isready` para PostgreSQL, una solicitud a `/health` para FastAPI y espera por estado saludable. |
| Justificación | Compose inicia cada consumidor cuando su dependencia puede atender solicitudes reales. |
| Verificación | `22_r09_estado_salud.txt` muestra `db` y `backend` como `healthy`; `23_r09_health_final.txt` devuelve HTTP 200. |
| Impacto | Sin healthchecks, un contenedor puede estar `running` mientras la aplicación aún no está disponible. |

## Trazabilidad en Git

Los commits separan el proyecto original, el diagnóstico y las correcciones principales. Los mensajes permiten relacionar los cambios con R-01 a R-09 mediante `git log --oneline`.