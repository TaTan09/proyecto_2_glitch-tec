# Auditoría previa a la migración

Base del Proyecto 1 revisada: commit `2e3d9baf557ea3c0f9fda9fbccb56c949dfbd884`. Se revisaron ambos Compose, `.env.example`, `.gitignore`, Dockerfiles, nginx, fuentes/configuración de Vue, los cinco servicios Flask con sus modelos/requisitos, bootstrap SQL, schemas/seeds, README y documentación de backend/datos, antes de escribir los manifiestos.

## Contratos confirmados

| Componente | Puerto | Configuración requerida | Rutas usadas |
|---|---|---|---|
| inventario | 5000 | DB_HOST/PORT/NAME/USER/PASSWORD, UMBRAL_STOCK_BAJO | `/api/inventario/health`, CRUD `/api/inventario/productos[/<id>]`, reservar/liberar/resumen |
| clientes | 5000 | DB_HOST/PORT/NAME/USER/PASSWORD | `/api/clientes/health`, CRUD `/api/clientes[/<id>]` |
| pedidos | 5000 | DB_HOST/PORT/NAME/USER/PASSWORD, INVENTARIO_URL | `/api/pedidos/health`, `/api/pedidos[/<id>]`, `/api/pedidos/resumen-hoy` |
| reportes | 5000 | INVENTARIO_URL, PEDIDOS_URL | `/api/reportes/health`, `/api/reportes/dashboard` |
| catalogo | 5000 | INVENTARIO_URL | `/api/catalogo/health`, `/api/catalogo/productos` |
| PostgreSQL | 5432 | POSTGRES_USER/PASSWORD/DB, PGHOST, nombres/usuarios/passwords de tres bases | socket local durante bootstrap |
| Vue | nginx 80 en producción | fetch relativo, Vue Router history | `/`, `/inventario`, `/pedidos` |
| gateway | 80 | nginx.conf | cinco prefijos API, `/health`, frontend |

`inventario_db`, `clientes_db` y `pedidos_db` conservan sus schemas. Catálogo y reportes no tienen BD propia. Las clases construyen `postgresql+psycopg2://...` con variables y ejecutan `db.create_all()` al importar/inicializar la app; por ello requieren PostgreSQL listo antes de Gunicorn `--preload`.

## Problemas o limitaciones heredadas

1. El gateway compila Vue y copia `dist`: hay que separar imágenes y Pods para la migración. Se corrigió.
2. El seed de clientes contiene dos carnés ficticios y el Dashboard no consulta clientes. Se sembraron los seis carnés reales y se añadió una sección pequeña que consume la BD por API.
3. El proxy Vite no tiene `/api/clientes`. Se agregó para que la nueva sección también funcione en desarrollo.
4. `docker-compose.dev.yml` no pasa todos los usuarios/nombres de bases ni `PGHOST` y tiene un healthcheck con usuario `admin` fijo; difiere del Compose final. Se conserva como archivo histórico; los YAML usan el contrato completo.
5. El README describe una rama de datos y decisiones que ya no reflejan `main`. Se preserva como historia y se añade un enlace destacado al README del Proyecto 2.
6. El bootstrap incluye tres contraseñas de respaldo hardcodeadas. Se sustituyeron únicamente esas asignaciones por validaciones de variable obligatoria; no se cambió su lógica SQL ni el socket interno.
7. Los modelos Python interpolan usuario/password en una URI sin escaping. El bootstrap restringe caracteres e interpola passwords SQL. La migración conserva Python y valida un subconjunto compatible antes de crear Secret.
8. Las rutas health HTTP devuelven `ok` sin consultar la BD; son probes del proceso. Se comprueba disponibilidad funcional por CRUD y consultas, sin prometer que health garantice la BD.
9. Pedidos tiene creación y consulta; no tiene PUT/DELETE. La evidencia de CRUD completo se realiza en Inventario, que sí implementa los cuatro métodos desde la interfaz. No se inventaron rutas de pedidos.
10. Pedidos no valida el carné contra clientes y no compensa automáticamente una reserva si falla su propio commit. Son limitaciones de negocio existentes; no se amplió el alcance de la migración.
11. El Node instalado localmente, 22.15.0, es inferior al rango declarado por el frontend y algunas herramientas. `npm ci` emitió advertencias; el build efectivamente finalizó. El Dockerfile usa Node 22 Alpine; la publicación de las imágenes 2.0 fue confirmada por el equipo para la entrega final.
12. El PVC nuevo inicializa la BD, pero no importa datos del volumen Compose existente. Si necesitan conservar datos históricos, exportar/importar por un procedimiento separado; no se borró ni modificó el volumen del Proyecto 1.

## Decisiones Kubernetes

Ocho Deployments, una réplica inicial cada uno; PostgreSQL `Recreate`; 1Gi RWO `local-path`; ConfigMap proyectado que conserva `schema/` y `seeds/`; socket de bootstrap conservado; probes PostgreSQL por TCP para no aceptar su servidor temporal; espera TCP en los tres backends con BD; siete ClusterIP y dos entradas del gateway; LoadBalancer 8080 para evitar el 80 de Traefik; manifiestos finales versionados directamente desplegables para Glitch Tec, ID `glitch-tec` y namespace `proyecto-2-glitch-tec`; repositorio final `TaTan09/proyecto_2_glitch-tec`, rama de entrega `main`. Las siete imágenes `javierini/elquetzal-*:2.0` están publicadas según confirmación del equipo; ningún Secret real está versionado y las pruebas de k3s siguen pendientes.
