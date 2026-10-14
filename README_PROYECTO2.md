# Proyecto 2: El Quetzal en Kubernetes/k3s

Migración de la aplicación del Proyecto 1 a la VM Ubuntu Server con k3s. Se conservan los cinco microservicios Flask, Vue, PostgreSQL 16, las tres bases y los contratos HTTP. La demostración del Proyecto 2 se ejecuta exclusivamente con Kubernetes.

Repositorio final: **[TaTan09/proyecto_2_glitch-tec](https://github.com/TaTan09/proyecto_2_glitch-tec)**. **`main` es la rama final de entrega y despliegue**, con la implementación Kubernetes incorporada en el commit `ec3f77e`. `docker-compose.yml` y `docker-compose.dev.yml` permanecen como referencia del Proyecto 1. El gateway está adaptado a Kubernetes; el Compose histórico necesita su configuración original para volver a ejecutar el Proyecto 1.

## Equipo y nomenclatura

Grupo **Glitch Tec**:

| Integrante | Carné |
|---|---|
| Mario Arévalo | 1072123 |
| Jonatán Chávez | 1185023 |
| Javier Ruano | 1207022 |
| Javier Aldana | 1184523 |
| Hector Pérez | 1304223 |
| José Rosales | 1143323 |

Los registros se siembran en `clientes_db`; el Dashboard obtiene los nombres y carnés con `GET /api/clientes`. No simula los nombres cuando el backend falla. Advierte si falta alguno de los seis registros.

El equipo **Glitch Tec** confirmó el **idEquipo `glitch-tec`**. El namespace Kubernetes final es **`proyecto-2-glitch-tec`** y el repositorio final es **`TaTan09/proyecto_2_glitch-tec`**, en la rama **`main`**.

El enunciado usa guiones bajos para carpeta y repositorio; Kubernetes requiere nombres DNS sin `_`, por lo que el namespace usa guiones. Los recursos, labels `team` y hostnames de los Pods ya tienen `glitch-tec`. Los 22 manifiestos finales versionados en **`k8s/` son directamente desplegables**; no requieren sustituciones ni una carpeta generada.

`configure-project.sh` se conserva como comprobación opcional del ID final; no genera ni modifica archivos. El flujo normal de despliegue no necesita ejecutarlo. La sincronización de ConfigMaps sigue disponible en `python3 scripts/project.py sync-configmaps` y utiliza siempre el ID final.

Los Pods tienen `hostname: <servicio>-glitch-tec`; se puede mostrar con `kubectl exec ... -- hostname`. El hostname de la VM se comprueba aparte, sin que los scripts cambien el sistema. El enunciado también menciona imágenes `usuario-<carné>`; por instrucción del equipo se conservan `javierini/elquetzal-<servicio>:2.0`.

## Arquitectura

```mermaid
flowchart TD
    Browser[Navegador] --> Entry[NodePort 30080 / LoadBalancer 8080]
    Entry --> Gateway[Gateway nginx :80]
    Gateway -->|/| FrontService[Service frontend :80]
    FrontService --> Front[Vue servido por nginx con fallback SPA]
    Gateway -->|/api/catalogo| Catalogo[Service catalogo :5000]
    Gateway -->|/api/clientes| Clientes[Service clientes :5000]
    Gateway -->|/api/inventario| Inventario[Service inventario :5000]
    Gateway -->|/api/pedidos| Pedidos[Service pedidos :5000]
    Gateway -->|/api/reportes| Reportes[Service reportes :5000]
    Catalogo --> Inventario
    Reportes --> Inventario
    Reportes --> Pedidos
    Pedidos --> Inventario
    Clientes --> DBService[Service postgres :5432]
    Inventario --> DBService
    Pedidos --> DBService
    DBService --> PG[PostgreSQL 16 - una réplica]
    PG --> PVC[PVC 1Gi ReadWriteOnce / local-path]
```

Todos los recursos propios pertenecen al namespace del equipo. La comunicación usa DNS de Services; no usa IPs de Pods ni IPs internas fijas. Los Services seleccionan `app`, `project: el-quetzal` y `team` idénticos a los labels de cada Pod.

## Archivos

```text
k8s/
  namespace.yaml
  config/      app-configmap.yaml, postgres-init-configmap.yaml
  db/          postgres-pvc.yaml, postgres-deployment.yaml, postgres-service.yaml
  app/         frontend y cinco backends: *-deployment.yaml
  services/    frontend y cinco backends: *-service.yaml
  gateway/     gateway-configmap.yaml, gateway-deployment.yaml,
               gateway-nodeport.yaml, gateway-loadbalancer.yaml
scripts/
  configure-project.sh       comprueba el ID final (opcional)
  create-secret.sh           crea el Secret sin versionar credenciales
  build-push-images.sh       build local o push explícito
  validate-manifests.sh      validación estática y dry-run opcional
  project.py                comprobación de ID y sincronización de ConfigMaps
  prepare-secret.py         lectura y validación de credenciales
  validate-manifests.py      contratos y referencias de los YAML
  requirements.txt          PyYAML para validación
config/credentials.env.example  plantilla vacía
docs/VALIDACION_VM.md        pasos de despliegue y diagnóstico
docs/EVIDENCIAS.md           capturas A-F pendientes
docs/AUDITORIA_PROYECTO1.md  contratos y problemas heredados
docs/INFORME_IMPLEMENTACION.md  informe y resultados reales
```

## Imágenes

| Componente | Imagen | Contexto de build | Puerto |
|---|---|---|---|
| catálogo | `javierini/elquetzal-catalogo:2.0` | `services/catalogo` | 5000 |
| clientes | `javierini/elquetzal-clientes:2.0` | `services/clientes` | 5000 |
| inventario | `javierini/elquetzal-inventario:2.0` | `services/inventario` | 5000 |
| pedidos | `javierini/elquetzal-pedidos:2.0` | `services/pedidos` | 5000 |
| reportes | `javierini/elquetzal-reportes:2.0` | `services/reportes` | 5000 |
| frontend | `javierini/elquetzal-frontend:2.0` | `frontend` | 80 |
| gateway | `javierini/elquetzal-gateway:2.0` | raíz, `-f gateway/Dockerfile` | 80 |
| PostgreSQL | `postgres:16` | imagen oficial | 5432 |

Las siete imágenes propias `:2.0` ya fueron publicadas correctamente en Docker Hub, según confirmación del equipo. **No se deben reconstruir, modificar ni volver a publicar para esta entrega.** Las referencias de imagen en los Deployments permanecen exactamente como están en la tabla. `build-push-images.sh` se conserva como utilidad de mantenimiento; no forma parte del despliegue normal.

Kubernetes monta la configuración final de nginx desde `gateway-config-glitch-tec`. `gateway/nginx.conf` contiene los mismos DNS finales de Services. La comprobación `nginx -t` dentro del Pod se realiza en la VM, donde existen esos Services.

## Configuración y credenciales

`app-config-glitch-tec` contiene `DB_HOST=postgres-glitch-tec`, `DB_PORT=5432`, las bases `inventario_db`, `clientes_db`, `pedidos_db`, `UMBRAL_STOCK_BAJO=10`, `INVENTARIO_URL` y `PEDIDOS_URL`. Cada backend recibe su base como `DB_NAME` mediante `configMapKeyRef`.

El Secret `db-credentials-glitch-tec` se crea localmente con ocho claves: `POSTGRES_ADMIN_USER/PASSWORD`, `INVENTARIO_DB_USER/PASSWORD`, `CLIENTES_DB_USER/PASSWORD` y `PEDIDOS_DB_USER/PASSWORD`. Los cuatro usuarios deben ser distintos. Los Deployments usan `secretKeyRef`, mapeando `POSTGRES_ADMIN_USER` a `POSTGRES_USER` y los usuarios de servicio a `DB_USER`/`DB_PASSWORD`.

```bash
mkdir -p secrets
cp config/credentials.env.example secrets/database.secret
chmod 600 secrets/database.secret
nano secrets/database.secret
./scripts/create-secret.sh glitch-tec secrets/database.secret
```

También acepta las ocho variables exportadas, sin archivo; tienen prioridad sobre el archivo. No usar `source`: el lector solo acepta pares `CLAVE=valor` sin ejecutar código. La plantilla no incluye contraseñas ni usuarios reales. Se recomiendan usuarios `admin`, `inventario_user`, `clientes_user`, `pedidos_user` y contraseñas distintas generadas localmente. El bootstrap ya aceptaba solo un conjunto restringido de caracteres; el backend construye una URI sin escaping. Para conservar ambos se admiten contraseñas con letras, números, `.`, `_` y `-`; una cadena aleatoria hexadecimal es compatible. No se modifica el contrato Python para esta migración.

El Secret real, `*.secret`, credenciales locales y `.generated` están ignorados. `create-secret.sh` crea el namespace antes del Secret, usa un temporal con permisos privados y lo elimina al salir. No ejecutar con `bash -x` ni capturar `kubectl get secret -o yaml`, variables o archivos de credenciales. Cambiar el Secret no cambia los roles de una BD ya inicializada: la rotación requiere actualizar PostgreSQL y reiniciar backends con los valores correspondientes.

## PostgreSQL e inicialización

Deployment de una réplica y estrategia `Recreate`, PVC 1Gi `ReadWriteOnce` con `local-path`, montado en `/var/lib/postgresql/data`. `PGDATA` queda en su subcarpeta `pgdata`, evitando archivos ajenos en la raíz del volumen. No utiliza `hostPath`.

El ConfigMap de inicialización reproduce `init-databases.sh`, `schema/` y `seeds/` en `/docker-entrypoint-initdb.d`, con permiso ejecutable y solo lectura. Se conserva la lógica de roles, bases, schemas y seeds; únicamente se eliminan las tres contraseñas por defecto heredadas para exigir valores del Secret. `PGHOST=/var/run/postgresql` conserva el socket interno del Proyecto 1. La imagen oficial ejecuta el bootstrap solo al iniciar un volumen vacío.

Las probes de PostgreSQL usan TCP a `127.0.0.1`, porque el servidor temporal del bootstrap solo atiende el socket y no debe habilitar los backends prematuramente. Clientes, inventario y pedidos esperan el Service en 5432 con un initContainer BusyBox. Las probes HTTP existentes no verifican la BD continuamente; el CRUD es la comprobación funcional real.

Cambiar ConfigMaps no vuelve a sembrar un PVC inicializado. La guía incluye un comando para aplicar únicamente el seed de clientes al volumen existente sin borrar datos. Nunca eliminar el PVC como prueba de persistencia.

## Gateway, Services y orden de despliegue

El gateway nginx usa exclusivamente DNS de Services con el sufijo `-glitch-tec`. Conserva `/health`, prefijos completos `/api/...` y headers Host/Real-IP/Forwarded. `proxy_pass` no añade una URI que recorte el prefijo. `/` se reenvía al Service frontend; su nginx soporta `/`, `/inventario` y `/pedidos` con fallback SPA.

Siete Services ClusterIP: frontend 80, cinco backends 5000, PostgreSQL 5432. Un NodePort del gateway usa `80 → 80`, puerto de nodo 30080. Otro Service del gateway es LoadBalancer `8080 → 80`. Se elige 8080 porque el Traefik incluido en k3s puede ocupar 80/443. La IP externa y el acceso desde host quedan sujetos a la VM y su red; no se fija una EXTERNAL-IP. Referencia: [Networking Services de k3s](https://docs.k3s.io/networking/networking-services).

Orden: confirmar nodo → namespace/Secret → ConfigMaps → Service PostgreSQL/PVC/Deployment → esperar PostgreSQL → frontend/backends y ClusterIP → esperar Pods/endpoints → gateway → NodePort y LoadBalancer. El script del Secret aplica primero el namespace; la guía vuelve a aplicarlo de forma idempotente. PostgreSQL necesita su Service antes de que los initContainers intenten resolverlo.

La guía copiable completa está en [docs/VALIDACION_VM.md](docs/VALIDACION_VM.md). Las capturas requeridas están en [docs/EVIDENCIAS.md](docs/EVIDENCIAS.md). Después de cambiar nginx aplicar ConfigMap y reiniciar gateway, pues nginx carga su configuración al iniciar. Después de cambiar variables de entorno aplicar ConfigMap/Secret y reiniciar los Deployments consumidores. Referencia: [ConfigMaps de Kubernetes](https://kubernetes.io/docs/concepts/configuration/configmap/).

## Validación

```bash
python3 -m pip install -r scripts/requirements.txt
./scripts/validate-manifests.sh
# Con kubectl y su API accesible:
./scripts/validate-manifests.sh --kubectl
```

La validación estática lee directamente los manifiestos finales de `k8s/` y exige el ID `glitch-tec`; no sustituye valores ni genera copias. Verifica YAML, labels/selectors, nombres DNS, imágenes, puertos, probes reales, variables contra código, referencias ConfigMap/Secret/PVC, mounts, SQL sincronizado, fallback SPA y carnés. No prueba el clúster.

Para regenerar ConfigMaps después de modificar SQL o nginx:

```bash
python3 scripts/project.py sync-configmaps
```

La compilación local de Vue y los controles estáticos se ejecutaron; consulta el informe para sus resultados y limitaciones. No se afirma que haya Pods Running, PVC Bound, endpoints, CRUD, persistencia, escalado, autoreparación o acceso de red hasta probarlos en la VM. El ID y la publicación de las siete imágenes ya están confirmados por el equipo; quedan pendientes las pruebas reales de la VM y las credenciales locales de la instalación.
