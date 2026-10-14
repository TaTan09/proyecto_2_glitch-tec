# Proyecto 2 Kubernetes - El Quetzal

Aplicación de Distribuidora El Quetzal desplegada en Kubernetes/k3s por el equipo **Glitch Tec**.

- **idEquipo:** `glitch-tec`.
- **Repositorio final:** [TaTan09/proyecto_2_glitch-tec](https://github.com/TaTan09/proyecto_2_glitch-tec).
- **Rama final de entrega y despliegue:** `main`.
- **Namespace Kubernetes:** `proyecto-2-glitch-tec`.
- **Imágenes publicadas:** las siete `javierini/elquetzal-*:2.0`, conservadas sin reconstrucción.

Los manifiestos finales se despliegan directamente desde `k8s/`. Consulta [README_PROYECTO2.md](README_PROYECTO2.md) para la arquitectura y configuración, [docs/VALIDACION_VM.md](docs/VALIDACION_VM.md) para desplegar desde este repositorio en `main`, y [docs/EVIDENCIAS.md](docs/EVIDENCIAS.md) para las capturas A-F. El Secret real permanece fuera del repositorio.

## Referencia histórica del Proyecto 1

Los apartados siguientes conservan el contexto de la integración original con Docker Compose, sus comandos y pendientes de aquella etapa. Para la entrega actual del Proyecto 2 utiliza las instrucciones Kubernetes enlazadas arriba. Los nombres de ramas y la arquitectura integrada de gateway/Vue de esta referencia corresponden al Proyecto 1.

Aplicación web para gestionar inventario y pedidos de Distribuidora El Quetzal sobre una infraestructura virtualizada.

En el Proyecto 1, la integración de datos se desarrolló en la rama histórica `datos/jeam256`, combinando el Backend de dominio de Héctor con PostgreSQL. Esta referencia describe aquella etapa; la entrega actual del Proyecto 2 utiliza `main` de `TaTan09/proyecto_2_glitch-tec`.

### Arquitectura del Proyecto 1

```text
Cliente / navegador
        |
        v
  Gateway Nginx + Vue       (integración de frontend y gateway)
        |
        v
  Microservicios Flask
  ├── inventario  ──> inventario_db
  ├── clientes    ──> clientes_db
  ├── pedidos     ──> pedidos_db
  ├── reportes    ──> consulta inventario y pedidos por HTTP
  └── catalogo    ──> consulta inventario por HTTP
        |
        v
  Un motor PostgreSQL con bases independientes por servicio
```

El Backend de aquella integración del Proyecto 1 se ejecutaba con `docker-compose.dev.yml`; gateway y frontend se integraron en etapas posteriores. Los comandos Compose de esta referencia pertenecen al Proyecto 1.

### Qué se integró

#### Backend existente

Se conserva el trabajo de Héctor:

- Microservicios Flask de inventario, clientes, pedidos, reportes y catálogo.
- CRUD de productos y clientes.
- Creación y consulta de pedidos.
- Validación de stock.
- Descuento atómico mediante `SELECT ... FOR UPDATE`.
- Dashboard agregado mediante llamadas HTTP.

El contrato completo de rutas y JSON está en [DocumentacionBackend.md](DocumentacionBackend.md).

#### Capa de Datos añadida

Se reemplazó el script inicial de PostgreSQL por una inicialización completa en `postgres-init/`:

```text
postgres-init/
├── init-databases.sh
├── schema/
│   ├── inventario.sql
│   ├── clientes.sql
│   └── pedidos.sql
└── seeds/
    ├── inventario.sql
    └── clientes.sql
```

El bootstrap crea:

| Servicio | Base | Usuario | Tablas |
|---|---|---|---|
| Inventario | `inventario_db` | `inventario_user` | `productos` |
| Clientes | `clientes_db` | `clientes_user` | `clientes` |
| Pedidos | `pedidos_db` | `pedidos_user` | `pedidos`, `pedido_items` |

`catalogo` y `reportes` no tienen base propia porque consumen los endpoints de otros servicios.

### Cómo ejecutar localmente

#### Requisitos

- Docker Engine.
- Docker Compose.
- Git.

#### 1. Crear las variables locales

En PowerShell:

```powershell
Copy-Item .env.example .env
```

Editar `.env` y cambiar las contraseñas de ejemplo. El archivo `.env` está ignorado por Git y no debe subirse.

Las variables más importantes son:

```text
POSTGRES_ADMIN_USER
POSTGRES_ADMIN_PASSWORD
INVENTARIO_DB_NAME
INVENTARIO_DB_USER
INVENTARIO_DB_PASSWORD
CLIENTES_DB_NAME
CLIENTES_DB_USER
CLIENTES_DB_PASSWORD
PEDIDOS_DB_NAME
PEDIDOS_DB_USER
PEDIDOS_DB_PASSWORD
UMBRAL_STOCK_BAJO
```

El valor actual de `UMBRAL_STOCK_BAJO` es `10`, porque es el valor que utiliza el Backend de inventario.

#### 2. Levantar el Backend

```powershell
docker compose --env-file .env -f docker-compose.dev.yml up --build -d
```

Servicios publicados localmente:

| Servicio | URL base |
|---|---|
| PostgreSQL | `localhost:5432` |
| Inventario | `http://localhost:5001` |
| Clientes | `http://localhost:5002` |
| Pedidos | `http://localhost:5003` |
| Reportes | `http://localhost:5004` |
| Catálogo | `http://localhost:5005` |

Para revisar el estado:

```powershell
docker compose --env-file .env -f docker-compose.dev.yml ps
docker compose --env-file .env -f docker-compose.dev.yml logs -f postgres
```

El script de PostgreSQL se ejecuta automáticamente únicamente cuando el volumen `postgres_data` está vacío. Si se modifican las tablas y se necesita recrear la base desde cero, se puede usar:

```powershell
docker compose --env-file .env -f docker-compose.dev.yml down -v
docker compose --env-file .env -f docker-compose.dev.yml up --build -d
```

`down -v` elimina los datos locales de PostgreSQL; usarlo solo cuando se quiera reiniciar la base.

### Pruebas rápidas

#### Salud de los servicios

```powershell
Invoke-RestMethod http://localhost:5001/api/inventario/health
Invoke-RestMethod http://localhost:5002/api/clientes/health
Invoke-RestMethod http://localhost:5003/api/pedidos/health
Invoke-RestMethod http://localhost:5004/api/reportes/health
Invoke-RestMethod http://localhost:5005/api/catalogo/health
```

#### Inventario

```powershell
Invoke-RestMethod http://localhost:5001/api/inventario/productos
Invoke-RestMethod http://localhost:5001/api/inventario/resumen
```

#### Pedido válido

```powershell
$body = @{
  carne = "1234567"
  items = @(
    @{ sku = "A1"; cantidad = 2 }
  )
} | ConvertTo-Json -Depth 4

Invoke-RestMethod `
  -Method Post `
  -Uri http://localhost:5003/api/pedidos `
  -ContentType "application/json" `
  -Body $body
```

El pedido debe quedar `confirmado` y el stock del producto `A1` debe disminuir.

#### Pedido rechazado

Probar con una cantidad mayor al stock disponible, por ejemplo:

```powershell
$body = @{
  carne = "1234567"
  items = @(
    @{ sku = "D4"; cantidad = 1 }
  )
} | ConvertTo-Json -Depth 4

Invoke-RestMethod `
  -Method Post `
  -Uri http://localhost:5003/api/pedidos `
  -ContentType "application/json" `
  -Body $body
```

Debe responder con estado HTTP `409` y no modificar el stock.

#### Persistencia

```powershell
docker compose --env-file .env -f docker-compose.dev.yml down
docker compose --env-file .env -f docker-compose.dev.yml up -d
Invoke-RestMethod http://localhost:5001/api/inventario/productos
```

Los productos y pedidos deben conservarse porque PostgreSQL utiliza el volumen nombrado `postgres_data`.

### Decisiones de datos

- Inventario es dueño de la tabla `productos` y de la transacción de reserva de stock.
- Pedidos no accede directamente a `inventario_db`; llama a `/api/inventario/reservar`.
- `pedido_items` guarda una copia del SKU, nombre y precio del producto en el momento del pedido.
- No existe una llave foránea entre `pedidos.carne` y `clientes.carne` porque pertenecen a bases distintas.
- Los carnés incluidos en `postgres-init/seeds/clientes.sql` son de demostración y deben reemplazarse por los reales del grupo.
- El endpoint `/api/inventario/liberar` existe para compensar una reserva si un pedido se cancela posteriormente.

### Pendientes registrados durante el Proyecto 1

1. Reemplazar los carnés de prueba por los carnés reales.
2. Confirmar con el equipo si `pedidos` debe validar el carné mediante el servicio `clientes`.
3. Definir con Orquestación el Compose final que incluya gateway, frontend, red interna y VM.
4. Verificar la compensación de `/api/inventario/liberar` si falla la persistencia posterior a una reserva.
5. Probar el flujo completo y capturar las evidencias exigidas: hostname de la VM, reloj, carnés y URL de Docker Hub.

### Rama histórica de integración

La integración histórica se desarrolló en:

```text
datos/jeam256
```

La integración del Proyecto 1 se revisaba mediante Pull Request en su repositorio original. La rama final de entrega del Proyecto 2 es `main` de `TaTan09/proyecto_2_glitch-tec`.
