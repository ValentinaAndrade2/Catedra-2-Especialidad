# Validación final

Pruebas ejecutadas en GitHub Codespaces el 24 de noviembre de 2026 sobre la versión corregida de la aplicación.

## Resultados

| Prueba | Resultado | Evidencias |
|---|---|---|
| Arranque limpio | Los tres servicios se reconstruyeron e iniciaron. PostgreSQL y backend quedaron `healthy`; frontend publicó `8080:80`. | `01_arranque_limpio_down.txt` a `03_estado_arranque_limpio.txt` |
| Comunicación completa | Nginx respondió HTTP 200, FastAPI creó un registro con HTTP 201 y PostgreSQL almacenó los datos. | `04_health_via_nginx.txt` a `07_consulta_directa_postgresql.txt` |
| Persistencia | Los registros anteriores continuaron disponibles gracias al volumen nombrado `pgdata`. | Evidencias previas en `../mejoras_obligatorias/` y `06_listar_items_final.txt` |
| Fallo controlado | Backend fue detenido con código de salida 0. Frontend y PostgreSQL permanecieron activos. La petición a la API agotó el tiempo de espera de 10 segundos. | `08_detener_backend.txt` a `12_inspect_backend_detenido.txt` |
| Recuperación | Backend volvió a estado `healthy`, `/api/health` respondió HTTP 200 y los registros permanecieron disponibles. | `13_iniciar_backend.txt` a `16_datos_recuperados.txt` |

## Conclusión

La aplicación puede iniciarse de forma reproducible, comunica correctamente frontend, backend y base de datos, conserva los datos al recrear contenedores, permite diagnosticar una interrupción controlada y recupera el servicio sin pérdida de información.