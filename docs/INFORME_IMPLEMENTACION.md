# Informe de implementación del Proyecto 2

Estado final: equipo **Glitch Tec**, ID confirmado **`glitch-tec`**, namespace **`proyecto-2-glitch-tec`**, repositorio final **[TaTan09/proyecto_2_glitch-tec](https://github.com/TaTan09/proyecto_2_glitch-tec)** y rama de entrega **`main`**. Los 22 manifiestos versionados en `k8s/` se despliegan directamente. Las siete imágenes `javierini/elquetzal-*:2.0` fueron publicadas según confirmación del equipo y se conservan. Los apartados 2 y 3 describen la migración inicial; la sección «Finalización con idEquipo confirmado» documenta la etapa incorporada en `ec3f77e`.

## 1. Rama utilizada

La rama final de entrega y despliegue es **`main`** de **`TaTan09/proyecto_2_glitch-tec`**. El repositorio final incorporó exactamente el commit `ec3f77e` de la implementación Kubernetes antes de esta limpieza documental. La migración inicial partió del código del Proyecto 1 en el commit `2e3d9baf557ea3c0f9fda9fbccb56c949dfbd884`.

## 2. Archivos nuevos

40 archivos nuevos:

```text
.dockerignore
README_PROYECTO2.md
config/credentials.env.example
frontend/.dockerignore
frontend/Dockerfile
frontend/nginx.conf
docs/AUDITORIA_PROYECTO1.md
docs/VALIDACION_VM.md
docs/EVIDENCIAS.md
docs/INFORME_IMPLEMENTACION.md
k8s/namespace.yaml
k8s/config/app-configmap.yaml
k8s/config/postgres-init-configmap.yaml
k8s/db/postgres-pvc.yaml
k8s/db/postgres-deployment.yaml
k8s/db/postgres-service.yaml
k8s/app/frontend-deployment.yaml
k8s/app/catalogo-deployment.yaml
k8s/app/clientes-deployment.yaml
k8s/app/inventario-deployment.yaml
k8s/app/pedidos-deployment.yaml
k8s/app/reportes-deployment.yaml
k8s/services/frontend-service.yaml
k8s/services/catalogo-service.yaml
k8s/services/clientes-service.yaml
k8s/services/inventario-service.yaml
k8s/services/pedidos-service.yaml
k8s/services/reportes-service.yaml
k8s/gateway/gateway-configmap.yaml
k8s/gateway/gateway-deployment.yaml
k8s/gateway/gateway-nodeport.yaml
k8s/gateway/gateway-loadbalancer.yaml
scripts/project.py
scripts/prepare-secret.py
scripts/validate-manifests.py
scripts/requirements.txt
scripts/configure-project.sh
scripts/create-secret.sh
scripts/build-push-images.sh
scripts/validate-manifests.sh
```

Los archivos de `.generated/`, dependencias, build `dist`, cachés y comprobaciones locales temporales no forman parte del código versionado.

## 3. Archivos modificados

9 archivos:

- `.gitattributes`: conserva LF para Bash y añade LF para Python, YAML y nginx.
- `.gitignore`: excluye Secrets, credenciales locales, copias generadas, dependencias auxiliares y temporales.
- `README.md`: enlace destacado a la documentación del Proyecto 2, preservando la historia del Proyecto 1.
- `frontend/src/views/DashboardView.vue`: sección Grupo Glitch Tec obtenida desde `/api/clientes`, con errores de carga/registros faltantes y actualización.
- `frontend/vite.config.js`: proxy de clientes para desarrollo local de esa sección.
- `gateway/Dockerfile`: únicamente nginx, sin compilación/copia de Vue.
- `gateway/nginx.conf`: proxy de frontend y cinco Services Kubernetes, health y headers.
- `postgres-init/init-databases.sh`: exige las tres contraseñas de servicio; elimina únicamente los fallbacks inseguros, conserva toda la lógica SQL.
- `postgres-init/seeds/clientes.sql`: seis integrantes reales y `ON CONFLICT` para sus nombres.

Los dos archivos Docker Compose permanecen sin cambios y se usan como referencia histórica.

## 4. Arquitectura final

Navegador → NodePort 30080 o LoadBalancer 8080 → gateway nginx 80 → Service frontend 80 para `/` y cinco ClusterIP 5000 para `/api/...`. Inventario/clientes/pedidos → Service PostgreSQL 5432 → un Deployment PostgreSQL 16 → PVC 1Gi RWO `local-path`. Catálogo consulta inventario; reportes consulta inventario/pedidos; pedidos reserva stock por HTTP en inventario. Todos los recursos propios están en un namespace del equipo. Ocho Deployments y nueve Services, DNS interno, sin IPs fijas ni hostPath.

## 5. Correspondencia A-F

| Criterio | Implementación entregada | Evidencia real pendiente |
|---|---|---|
| A | namespace, ocho Deployments, siete imágenes propias publicadas (confirmado por el equipo) | capturas de imágenes, Pods Running/Ready, get all, hostname |
| B | PVC y montaje de datos, PostgreSQL una réplica Recreate | dato antes/después de eliminar solo el Pod |
| C | ConfigMaps, creación local de Secret, keyRefs y variables reales | conexión/CRUD con configuración desplegada sin mostrar passwords |
| D | réplicas iniciales 1, guía de edición YAML a 3 y apply | tres Pods, eliminación y reemplazo del ReplicaSet |
| E | siete ClusterIP, NodePort, LoadBalancer, gateway por DNS, fallback SPA | endpoints, acceso real host, EXTERNAL-IP, rutas y respaldo |
| F | CRUD Inventario conservado, seed real y Dashboard por API | CRUD navegador servido por k3s y seis carnés visibles |

## 6. Imágenes publicadas y conservadas

```text
javierini/elquetzal-catalogo:2.0
javierini/elquetzal-clientes:2.0
javierini/elquetzal-inventario:2.0
javierini/elquetzal-pedidos:2.0
javierini/elquetzal-reportes:2.0
javierini/elquetzal-frontend:2.0
javierini/elquetzal-gateway:2.0
```

PostgreSQL usa `postgres:16` y el initContainer `busybox:1.37`. El equipo confirmó la publicación correcta de las siete imágenes anteriores. Esta finalización conserva exactamente sus referencias y no ejecuta builds, login ni push de imágenes. El script de build/push queda como utilidad de mantenimiento.

## 7. Valores reales utilizados

Grupo Glitch Tec; Docker Hub `javierini`; tag `2.0`; motor PostgreSQL 16; bases `inventario_db`, `clientes_db`, `pedidos_db`; umbral de stock 10; puertos 5000/5432/80; NodePort 30080 y LoadBalancer 8080; una réplica inicial por Deployment; PVC 1Gi RWO local-path.

Integrantes: Mario Arévalo 1072123, Jonatán Chávez 1185023, Javier Ruano 1207022, Javier Aldana 1184523, Hector Pérez 1304223 y José Rosales 1143323. No se inventaron contactos. Los nombres mostrados en Dashboard se leen de PostgreSQL por clientes. No se generaron ni almacenaron credenciales reales.

## 8. Identidad final y preparativos restantes

- Equipo: Glitch Tec; ID confirmado: `glitch-tec`; namespace: `proyecto-2-glitch-tec`.
- `k8s/`, nginx y scripts de despliegue/validación usan nombres finales sin placeholders ni sustituciones.
- Las siete imágenes `:2.0` están publicadas según el equipo; no requieren reconstrucción ni modificación.
- Crear usuarios/passwords privados para la instalación; la plantilla está vacía y el Secret permanece fuera del repositorio.
- Obtener IP real y acceso a la VM por los responsables; no hay valores inventados.
- El repositorio final ya es `TaTan09/proyecto_2_glitch-tec`; la entrega y el despliegue utilizan su rama `main`.
- Tras reunir capturas reales, exportar el PDF final de evidencias A-F exigido por el docente. No se fabricó un PDF con supuestas evidencias.

## 9. Pruebas ejecutadas

- Extracción y lectura del enunciado PDF y auditoría de contratos del código actual antes de los YAML.
- `npm ci` en frontend: **correcto**, 248 paquetes instalados; avisos de engines por Node local 22.15.0 inferior al rango declarado. No se cambió package-lock.
- `npm run build` en frontend: **correcto**, Vite compiló 33 módulos y generó dist.
- ESLint directo sobre `src/views/DashboardView.vue` desde frontend: **correcto**, sin errores. No se ejecutó el lint global con `--fix` para alterar código ajeno a la migración.
- Validador Python sobre los **22 manifiestos**: **correcto**. YAML, DNS, selector/labels, puertos, ocho Deployments, nueve Services, imágenes, rutas health existentes, variables contra AST Python, ConfigMap/Secret/PVC, mounts, SQL, socket, probes, fallback SPA y carnés.
- En la migración inicial se comprobó el render con un ID temporal. La finalización reemplaza ese mecanismo por validación directa de `k8s/` con `glitch-tec`.
- Casos de IDs válidos e inválidos: **correctos**, incluyendo underscore, mayúsculas, extremos con guion, exceso de longitud y traversal rechazados.
- Lector de credenciales con valores de prueba, sin datos reales: **correcto**. Ocho claves, prioridad de entorno, faltantes, duplicados, claves desconocidas y caracteres/inyección rechazados antes de escribir.
- `create-secret.sh` con **kubectl simulado**: **correcto**. Namespace antes del Secret, ocho claves transmitidas por archivo temporal y stdin, sin passwords en argumentos. Esto no acredita la creación real en k3s.
- Funciones reales del Dashboard ejecutadas con fetch simulado: **correcto**. Usa nombres de API, filtra los carnés, mantiene orden y muestra estados para registros incompletos, respuesta inválida o API caída. Esto no acredita BD/HTTP desplegados.
- `bash -n` de los cuatro scripts shell y bootstrap: **correcto**.
- Script de imágenes con **Docker simulado**: **correcto**, siete comandos de build; `push` con configuración vacía se rechaza antes de intentar publicar. No equivale a builds Docker reales.
- `git diff --check`: **correcto**; `git check-ignore` confirmó exclusión de credenciales y `.generated`, conservando la plantilla documentada.

## 10. Pruebas que fallaron o no pudieron ejecutarse

- `kubectl apply --dry-run=client -R -f .generated/validacion-local/k8s` **falló**: el contexto local apunta a `https://kubernetes.docker.internal:6443`, cuyo DNS/API no está disponible. No se ocultó con `--validate=false` ni se confundió con un despliegue exitoso.
- `docker version` encontró el cliente pero **no el motor**: falta la tubería `dockerDesktopLinuxEngine`. Durante la migración inicial no fue posible ejecutar builds ni `nginx -t` dentro de imágenes. Para la finalización el equipo confirmó las imágenes publicadas y pidió no reconstruirlas; `nginx -t` en el Pod queda para la VM.
- El sandbox inicialmente bloqueó Git/Bash y DNS de descarga; clonación e instalación con permisos ampliados y comprobación Bash posterior sí se completaron. La prueba auxiliar de credenciales usó después un temporal dentro del workspace y pasó.
- Un intento auxiliar de `npx` desde la raíz sin dependencias locales se interrumpió; se sustituyó por ESLint instalado directamente en frontend, que pasó. No es un fallo del frontend compilado.

No se ejecutó ningún despliegue, CRUD real en Kubernetes, prueba de persistencia real ni captura de k3s desde este entorno.

## 11. Problemas detectados en Proyecto 1

Gateway y frontend unidos; carnés ficticios; falta de UI para mostrar clientes; proxy Vite sin clientes; fallbacks de passwords en bootstrap; Compose dev incompleto respecto al final; README histórico desactualizado; URI DB sin escaping; health HTTP sin verificación de BD; pedidos sin PUT/DELETE ni compensación automática de commit fallido. Detalle y alcance en `AUDITORIA_PROYECTO1.md`. Las limitaciones de negocio ajenas a la migración se conservan.

## 12. Decisiones técnicas

- Mantener motor, schemas, servicios y contratos; separar únicamente despliegue de Vue/gateway y agregar evidencia de integrantes.
- Usar DNS válido con `glitch-tec` en todos los recursos y manifiestos finales versionados en `k8s/`; el flujo normal no genera ni sustituye nombres.
- Secret externo con ocho claves; password obligatorio y compatible con bootstrap/URI heredados, sin reescribir Python.
- Proyectar un ConfigMap con rutas `schema/`/`seeds/`, script ejecutable, `PGHOST` socket y SQL sincronizado desde fuentes originales.
- PostgreSQL único con Recreate, PGDATA dentro del PVC y probes TCP que no aceptan el servidor temporal del bootstrap.
- Espera TCP simple con BusyBox para tres backends que conectan al arrancar; startup/readiness/liveness sin depender de Compose.
- LoadBalancer 8080 para evitar conflicto habitual con Traefik. No asignar una EXTERNAL-IP fija.
- Imágenes frontend/gateway separadas, gateway con ConfigMap al ejecutar, `.dockerignore` para contextos de build limpios.
- Pruebas funcionales reales y evidencias reservadas para el entorno k3s; guía explícita de port-forward/SSH como respaldo.

## Finalización con idEquipo confirmado

Esta etapa modifica únicamente nombres del equipo, scripts que usaban copias generadas y documentación. Se conservan arquitectura, réplicas iniciales, puertos, rutas, Dockerfiles, código de negocio, schemas/seeds y las siete referencias de imagen `javierini/elquetzal-*:2.0`. No se crean archivos versionados nuevos ni se versiona el Secret.

### Archivos modificados en esta finalización

34 archivos existentes:

```text
README.md
README_PROYECTO2.md
docs/AUDITORIA_PROYECTO1.md
docs/EVIDENCIAS.md
docs/INFORME_IMPLEMENTACION.md
docs/VALIDACION_VM.md
gateway/nginx.conf
k8s/app/catalogo-deployment.yaml
k8s/app/clientes-deployment.yaml
k8s/app/frontend-deployment.yaml
k8s/app/inventario-deployment.yaml
k8s/app/pedidos-deployment.yaml
k8s/app/reportes-deployment.yaml
k8s/config/app-configmap.yaml
k8s/config/postgres-init-configmap.yaml
k8s/db/postgres-deployment.yaml
k8s/db/postgres-pvc.yaml
k8s/db/postgres-service.yaml
k8s/gateway/gateway-configmap.yaml
k8s/gateway/gateway-deployment.yaml
k8s/gateway/gateway-loadbalancer.yaml
k8s/gateway/gateway-nodeport.yaml
k8s/namespace.yaml
k8s/services/catalogo-service.yaml
k8s/services/clientes-service.yaml
k8s/services/frontend-service.yaml
k8s/services/inventario-service.yaml
k8s/services/pedidos-service.yaml
k8s/services/reportes-service.yaml
scripts/configure-project.sh
scripts/create-secret.sh
scripts/project.py
scripts/validate-manifests.py
scripts/validate-manifests.sh
```

### Validaciones de finalización

- **Correcto:** validación directa de los 22 YAML en `k8s/`, tanto por defecto como indicando el directorio explícitamente: DNS, labels/selectors, hostnames, namespace, imágenes, puertos, probes, variables, ConfigMaps/Secret/PVC, mounts, SQL, SPA y carnés.
- **Correcto:** comparación semántica de cada YAML con el commit anterior: la única diferencia de contenido es la sustitución del ID. nginx conserva exactamente rutas, puertos y headers al sustituir el ID.
- **Correcto:** ningún cambio en frontend, servicios, bootstrap SQL/seeds, Dockerfiles, Compose, configuración de imágenes ni lector de credenciales. No se reconstruyen ni modifican imágenes.
- **Correcto:** búsqueda en todos los archivos versionados sin encontrar el antiguo placeholder; no hay credenciales reales ni Secrets versionados. Los scripts de despliegue/validación y la guía de VM no dependen de carpetas generadas.
- **Correcto:** sintaxis Python y Bash, `validate-manifests.sh`, comprobación opcional del ID final y rechazo de un ID distinto sin modificar archivos.
- **Correcto:** `create-secret.sh` con kubectl simulado utiliza directamente `k8s/namespace.yaml`, aplica el namespace antes del Secret final y no expone valores. Es una prueba local, no creación real en k3s.
- **Correcto:** formato del diff comprobado con `git diff --check`.
- **Falló por entorno:** `kubectl --request-timeout=5s apply --dry-run=client -R -f k8s` intentado contra el contexto local; la API `kubernetes.docker.internal:6443` no está disponible. Debe repetirse en la VM. No se oculta ese resultado ni se afirma un despliegue real.

La implementación de esta etapa quedó incorporada en el commit `ec3f77e` del repositorio final `TaTan09/proyecto_2_glitch-tec`. La rama final de entrega y despliegue es `main`. La publicación de las siete imágenes está confirmada por el equipo y se conserva sin reconstruirlas ni volver a publicarlas.

## PENDIENTE ÚNICAMENTE EN LA VM

1. Verificar nodo real k3s Ready, contexto, StorageClass/local-path y capacidad del nodo.
2. Crear el namespace y Secret real, aplicar ConfigMaps, PVC, PostgreSQL, frontend/backends, ClusterIP y gateway en k3s; repetir dry-run contra su API.
3. Verificar PVC Bound, montaje real, inicialización SQL, Pods Running/Ready, endpoints y DNS de Services.
4. Probar NodePort desde Windows/host por la IP real de la VM y configuración Bridge/NAT.
5. Verificar ServiceLB, puerto 8080, EXTERNAL-IP real y acceso LoadBalancer desde host; probar port-forward/túnel de respaldo.
6. Validar rutas, recargas SPA, cinco health, Dashboard y pedidos desde el gateway desplegado.
7. Ejecutar CRUD completo en navegador servido por k3s y comprobar los seis integrantes desde PostgreSQL/API/interfaz.
8. Crear registro, eliminar únicamente el Pod PostgreSQL, observar recreación y comprobar el mismo registro con el mismo PVC.
9. Editar YAML de inventario a tres réplicas, aplicar y comprobar tres Pods; eliminar uno y observar autoreparación real del ReplicaSet.
10. Obtener capturas reales A-F con reloj, carnés y hostname identificables; guardar comandos/resultados reales para el documento final de evidencias.
