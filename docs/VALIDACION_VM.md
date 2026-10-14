# Validación del Proyecto 2 en la VM Ubuntu Server

Ejecutar en Bash dentro de la VM con k3s ya instalado. Esta guía describe pruebas **pendientes**; los resultados se registran al ejecutarlas. Desde el host se usa el navegador de Windows. No usar Compose para demostrar este proyecto.

## 1. Nodo y repositorio

```bash
kubectl get nodes
kubectl cluster-info
kubectl config current-context
kubectl get storageclass
```

El nodo debe estar Ready, el contexto debe ser el k3s de la VM y debe existir `local-path`. No continuar con otro clúster por accidente. Si el usuario no tiene acceso al kubeconfig, el responsable debe darle acceso local al kubeconfig de k3s; no copiarlo al repositorio ni a capturas.

Si el repositorio final ya está clonado, desde su carpeta:

```bash
git fetch origin
git switch main
git pull --ff-only origin main
git branch --show-current
python3 --version
```

Si todavía no está clonado en la VM:

```bash
git clone https://github.com/TaTan09/proyecto_2_glitch-tec.git
cd proyecto_2_glitch-tec
```

Equipo: **Glitch Tec**. ID confirmado: **`glitch-tec`**. Namespace: **`proyecto-2-glitch-tec`**. Repositorio final: **[TaTan09/proyecto_2_glitch-tec](https://github.com/TaTan09/proyecto_2_glitch-tec)**. La rama final de entrega y despliegue es **`main`**. Se conservan las imágenes `javierini/elquetzal-*:2.0` publicadas por el equipo.

## 2. Configuración final y herramientas

Desde la raíz del repositorio:

```bash
export TEAM_ID='glitch-tec'
export NS='proyecto-2-glitch-tec'
```

Los manifiestos finales ya están versionados en **`k8s/`**. Los comandos de esta guía los aplican directamente. No ejecutar ningún paso de sustitución de ID ni generar copias para desplegar. `configure-project.sh` es una comprobación opcional y no forma parte de esta secuencia.

La validación necesita PyYAML. Si el sistema no permite instalar paquetes globalmente, utilizar un entorno virtual local ignorado por Git:

```bash
python3 -m venv tmp/validation-venv
source tmp/validation-venv/bin/activate
python3 -m pip install -r scripts/requirements.txt
./scripts/validate-manifests.sh
./scripts/validate-manifests.sh --kubectl
```

`python3-venv` puede requerir instalación por el administrador de Ubuntu si no está presente. No ignorar fallos. Dry-run lee `k8s/` y no crea recursos.

Las siete imágenes `javierini/elquetzal-{catalogo,clientes,inventario,pedidos,reportes,frontend,gateway}:2.0` ya fueron publicadas correctamente en Docker Hub según confirmación del equipo. **Usar esas imágenes sin reconstruirlas, modificarlas ni volver a publicarlas.** No se ejecuta `build-push-images.sh` durante esta entrega.

Para desplegar no hace falta Docker Engine en la VM; k3s descarga las imágenes mediante su runtime. Comprobar acceso desde el nodo y su arquitectura (`kubectl get nodes -o wide` y `uname -m`) si aparece un problema de descarga o arranque.

## 3. Preparar y crear Secret

```bash
mkdir -p secrets
cp config/credentials.env.example secrets/database.secret
chmod 600 secrets/database.secret
nano secrets/database.secret
```

Completar las ocho claves del archivo. Usuarios sugeridos: `admin`, `inventario_user`, `clientes_user`, `pedidos_user`, todos distintos. Contraseñas: valores aleatorios distintos con letras/números/`.`/`_`/`-`; 32 bytes hexadecimales generados localmente son compatibles. No poner comillas ni espacios, no usar passwords de ejemplo. No mostrar el archivo en capturas. Alternativa: exportar las ocho variables en la terminal sin registrarlas en historial/capturas y ejecutar el script sin archivo.

```bash
./scripts/create-secret.sh "$TEAM_ID" secrets/database.secret
```

El script valida credenciales, aplica **primero el namespace** (un Secret no puede existir sin él) y crea el Secret por stdin usando un temporal privado que elimina al salir. Nunca imprime los valores.

## 4. Namespace y ConfigMaps

```bash
kubectl apply -f "k8s/namespace.yaml"
kubectl get namespace "$NS"
kubectl -n "$NS" get secret "db-credentials-$TEAM_ID"
kubectl apply -f "k8s/config"
kubectl -n "$NS" get configmaps
kubectl -n "$NS" get configmap "app-config-$TEAM_ID" -o yaml
```

La aplicación repetida del namespace es idempotente. `get secret` sin `-o yaml/json` muestra metadatos y cantidad de claves, no valores. El Secret contiene 8 claves. No capturar el Secret codificado: base64 no protege las contraseñas.

## 5. PostgreSQL: Service, PVC y Deployment

```bash
kubectl apply -f "k8s/db/postgres-service.yaml"
kubectl apply -f "k8s/db/postgres-pvc.yaml"
kubectl apply -f "k8s/db/postgres-deployment.yaml"
kubectl -n "$NS" get pvc
kubectl -n "$NS" get pods -o wide
kubectl -n "$NS" rollout status "deployment/postgres-$TEAM_ID" --timeout=300s
kubectl -n "$NS" get pvc "postgres-data-$TEAM_ID"
kubectl -n "$NS" logs "deployment/postgres-$TEAM_ID" --tail=100
```

Esperar PVC Bound y Pod Ready. `local-path` puede usar `WaitForFirstConsumer`: PVC Pending antes de crear el Pod no necesariamente es fallo. Los logs del primer arranque deben reflejar roles, bases, schemas y seeds; revisarlos antes de capturarlos para no mostrar información sensible de errores.

No importar archivos desde una carpeta `hostPath`. El ConfigMap proyecta bootstrap y SQL, el PVC guarda solo los datos y `PGHOST` usa `/var/run/postgresql`. Si el PVC tenía datos, la imagen oficial no ejecuta nuevamente los seeds.

## 6. Frontend, backends y ClusterIP

```bash
kubectl apply -f "k8s/app"
kubectl apply -f "k8s/services"
kubectl -n "$NS" rollout status "deployment/frontend-$TEAM_ID" --timeout=300s
kubectl -n "$NS" rollout status "deployment/inventario-$TEAM_ID" --timeout=300s
kubectl -n "$NS" rollout status "deployment/clientes-$TEAM_ID" --timeout=300s
kubectl -n "$NS" rollout status "deployment/pedidos-$TEAM_ID" --timeout=300s
kubectl -n "$NS" rollout status "deployment/catalogo-$TEAM_ID" --timeout=300s
kubectl -n "$NS" rollout status "deployment/reportes-$TEAM_ID" --timeout=300s
kubectl -n "$NS" get pods -o wide
kubectl -n "$NS" get svc
kubectl -n "$NS" get endpoints
```

Los initContainers de clientes, inventario y pedidos deben finalizar antes del Flask. Deben existir siete ClusterIP y direcciones en sus endpoints con Pods Ready. En versiones recientes también se pueden comprobar EndpointSlices:

```bash
kubectl -n "$NS" get endpointslices
kubectl -n "$NS" get endpoints "inventario-$TEAM_ID"
```

## 7. Gateway y entradas externas

```bash
kubectl apply -f "k8s/gateway/gateway-configmap.yaml"
kubectl apply -f "k8s/gateway/gateway-deployment.yaml"
kubectl -n "$NS" rollout status "deployment/gateway-$TEAM_ID" --timeout=180s
kubectl apply -f "k8s/gateway/gateway-nodeport.yaml"
kubectl apply -f "k8s/gateway/gateway-loadbalancer.yaml"
kubectl -n "$NS" get all -o wide
kubectl -n "$NS" get svc
kubectl -n "$NS" get endpoints
kubectl -n "$NS" exec "deployment/gateway-$TEAM_ID" -- nginx -t
kubectl -n "$NS" exec "deployment/gateway-$TEAM_ID" -- hostname
hostname
date
```

El hostname del Pod contiene el ID. El hostname de la VM se muestra aparte; si el docente exige ese hostname con ID, el responsable de la VM debe configurarlo según lo acordado con el equipo. Las capturas deben mostrar reloj/carné identificable.

## 8. Acceso desde el host y rutas

Identificar la IP de red real de la VM:

```bash
ip -br addr
kubectl get nodes -o wide
```

En el navegador de Windows abrir `http://IP_DE_LA_VM:30080`. Registrar la IP real; no usar IP de Pod ni asumir que funciona antes de abrirla. Bridge o las reglas NAT deben permitir llegar al puerto 30080.

Dentro de la VM, sustituir el valor antes de copiar:

```bash
export VM_IP='IP_REAL_DE_LA_VM'
export BASE_URL="http://$VM_IP:30080"
curl -fsS "$BASE_URL/health"
curl -fsS "$BASE_URL/" -o /dev/null
curl -fsS "$BASE_URL/inventario" -o /dev/null
curl -fsS "$BASE_URL/pedidos" -o /dev/null
curl -fsS "$BASE_URL/api/inventario/productos"
curl -fsS "$BASE_URL/api/reportes/dashboard"
curl -fsS "$BASE_URL/api/clientes"
curl -fsS "$BASE_URL/api/catalogo/productos"
curl -fsS "$BASE_URL/api/pedidos"
```

Abrir `/`, `/inventario` y `/pedidos` en el navegador, recargar cada ruta con F5 y verificar que funcionan. El status 200 de HTML con curl no sustituye comprobar Vue en el navegador. Verificar también que las cinco rutas health devuelven JSON del servicio correspondiente.

## 9. CRUD completo desde navegador

Usar Inventario, que sí tiene crear/consultar/editar/eliminar. Pedidos conserva su flujo de creación/consulta, no tiene PUT ni DELETE.

1. Mantener visibles URL del gateway, reloj y carné identificable (por ejemplo una terminal junto al navegador con `date` y el carné del responsable).
2. En `/inventario`, abrir el formulario de producto nuevo. Usar SKU único `P2-1072123-CRUD`, nombre `Producto prueba Mario 1072123`, categoría `Evidencia`, precio Q25 y stock 5.
3. Guardar; capturar el registro y, opcionalmente, Network con POST 201.
4. Recargar; consultar el producto en la lista. Capturar su persistencia inicial.
5. Editar nombre a `Producto editado 1072123`, precio Q30 y stock 8. Guardar y recargar. Capturar actualización (PUT 200).
6. Eliminar y confirmar en el modal. Recargar y mostrar que desapareció (DELETE 204).
7. Desde `/pedidos`, ingresar el carné del responsable y hacer un pedido con un SKU que tenga stock. Mostrar pedido confirmado y stock reducido. Un SKU sin stock permite comprobar el rechazo sin ampliar el CRUD.

Cambiar el carné de ejemplo por el propio para evidencia individual. No eliminar los productos originales necesarios para pedidos.

## 10. Seis carnés visibles y origen en PostgreSQL

En `/`, la sección **Grupo Glitch Tec** debe mostrar los seis nombres y carnés. Capturar la sección junto al reloj y URL Kubernetes. En Network mostrar `GET /api/clientes` y los mismos registros; esto evita que la evidencia dependa solo de texto estático.

```bash
curl -fsS "$BASE_URL/api/clientes" | python3 -m json.tool
```

Para corroborar directamente desde PostgreSQL sin mostrar passwords:

```bash
kubectl -n "$NS" exec "deployment/postgres-$TEAM_ID" -- sh -c 'psql -U "$POSTGRES_USER" -d clientes_db -c "SELECT carne, nombre FROM clientes ORDER BY carne;"'
```

Si la BD ya estaba inicializada y le faltan los nuevos seeds, aplicar **solo el seed de clientes**, sin borrar PVC:

```bash
kubectl -n "$NS" exec -i "deployment/postgres-$TEAM_ID" -- sh -c 'psql -U "$POSTGRES_USER" -d clientes_db -v ON_ERROR_STOP=1' < postgres-init/seeds/clientes.sql
```

El seed conserva `ON CONFLICT` y actualiza nombres de los seis carnés. Volver a consultar y pulsar Actualizar en Dashboard. No afirmar que `apply` del ConfigMap resembró un volumen existente.

## 11. Persistencia: eliminar solamente el Pod PostgreSQL

En el navegador crear un **segundo** producto, que no se eliminará antes de esta prueba: SKU `P2-1072123-PERSIST`, nombre `Persistencia Mario 1072123`, categoría `Evidencia`, precio Q40, stock 7. Consultarlo y capturar antes.

```bash
curl -fsS "$BASE_URL/api/inventario/productos" | python3 -m json.tool
kubectl -n "$NS" get pvc "postgres-data-$TEAM_ID" -o wide
kubectl -n "$NS" get pods -l "app=postgres,team=$TEAM_ID" -o wide
PG_POD="$(kubectl -n "$NS" get pods -l "app=postgres,team=$TEAM_ID" -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" get pod "$PG_POD" -o jsonpath='{.metadata.uid}{"\n"}{.spec.volumes}{"\n"}'
date
kubectl -n "$NS" delete pod "$PG_POD"
kubectl -n "$NS" get pods -l "app=postgres,team=$TEAM_ID" -w
```

Observar nombre/UID del reemplazo y Ready, luego Ctrl+C para salir del watch. Se eliminó solo el Pod, no el Deployment, el namespace, el PVC ni el PV. Puede haber errores transitorios mientras vuelve PostgreSQL.

```bash
kubectl -n "$NS" rollout status "deployment/postgres-$TEAM_ID" --timeout=300s
kubectl -n "$NS" get pvc "postgres-data-$TEAM_ID" -o wide
kubectl -n "$NS" get pods -l "app=postgres,team=$TEAM_ID" -o wide
curl -fsS "$BASE_URL/api/inventario/productos" | python3 -m json.tool
date
```

Recargar Inventario desde el host y mostrar el mismo SKU, nombre, precio y stock. Conservar el producto hasta capturar el después. Si la API falla temporalmente, esperar PostgreSQL Ready y reintentar; no borrar el volumen como diagnóstico. La evidencia requiere el mismo dato antes/después y Pod distinto.

## 12. Escalado declarativo de inventario

Los manifiestos finales parten de una réplica. Editar **el YAML versionado** aplicado al clúster:

```bash
nano "k8s/app/inventario-deployment.yaml"
```

Cambiar la línea de nivel Deployment de `replicas: 1` a `replicas: 3`. Guardar y mostrar el archivo editado en la captura:

```bash
grep -n 'replicas:' "k8s/app/inventario-deployment.yaml"
date
kubectl apply -f "k8s/app/inventario-deployment.yaml"
kubectl -n "$NS" rollout status "deployment/inventario-$TEAM_ID" --timeout=300s
kubectl -n "$NS" get deployment "inventario-$TEAM_ID"
kubectl -n "$NS" get pods -l "app=inventario,team=$TEAM_ID" -o wide
kubectl -n "$NS" get endpoints "inventario-$TEAM_ID"
```

Capturar YAML, `apply` y tres Pods Ready. No usar `kubectl scale` como prueba principal ni crear otro Deployment. Se edita directamente `k8s/app/inventario-deployment.yaml`; no hay una copia generada intermedia. Conservar el cambio durante la prueba y volver a una réplica editando y aplicando el mismo archivo cuando hayan terminado las capturas.

## 13. Autoreparación de inventario

```bash
kubectl -n "$NS" get pods -l "app=inventario,team=$TEAM_ID" -o wide
INV_POD="$(kubectl -n "$NS" get pods -l "app=inventario,team=$TEAM_ID" -o jsonpath='{.items[0].metadata.name}')"
date
kubectl -n "$NS" delete pod "$INV_POD"
kubectl -n "$NS" get pods -l "app=inventario,team=$TEAM_ID" -w
```

Observar reemplazo automático sin crear manualmente Pods; Ctrl+C después de recuperar tres Ready:

```bash
kubectl -n "$NS" rollout status "deployment/inventario-$TEAM_ID" --timeout=300s
kubectl -n "$NS" get pods -l "app=inventario,team=$TEAM_ID" -o wide
kubectl -n "$NS" get replicasets -l "app=inventario,team=$TEAM_ID"
curl -fsS "$BASE_URL/api/inventario/productos"
date
```

Capturar nombre del Pod eliminado, nuevo nombre y ReplicaSet manteniendo tres réplicas. Para regresar a una réplica, editar ese mismo YAML a 1 y aplicar. No hacerlo antes de capturar la evidencia de tres Pods.

## 14. NodePort y LoadBalancer desde el host

```bash
kubectl -n "$NS" get svc "gateway-nodeport-$TEAM_ID" -o wide
kubectl -n "$NS" get svc "gateway-loadbalancer-$TEAM_ID" -o wide
kubectl -n "$NS" describe svc "gateway-loadbalancer-$TEAM_ID"
kubectl -n kube-system get pods -o wide
```

Probar en Windows `http://IP_DE_LA_VM:30080` y `http://EXTERNAL_IP_REAL:8080`. Obtener EXTERNAL-IP de la salida del Service, no inventarla. Cuando ServiceLB use la IP de la VM, verificar también `http://IP_DE_LA_VM:8080`. Capturar navegador y Service para cada acceso. El puerto externo 8080 debe estar libre y alcanzable; Traefik suele ocupar 80/443. ServiceLB crea recursos auxiliares administrados por k3s en su namespace de sistema; los manifiestos propios siguen en `$NS`.

Si EXTERNAL-IP está Pending, diagnosticar ServiceLB, puertos y eventos. No presentarlo como acceso correcto ni como prueba de LoadBalancer terminada. Mantener NodePort como opción y port-forward como respaldo permitido.

## 15. Port-forward de respaldo

Con SSH a la VM, un túnel permite que el navegador de Windows llegue al port-forward que escucha solo en localhost de la VM. Primera terminal dentro de la VM:

```bash
kubectl -n "$NS" port-forward "service/gateway-nodeport-$TEAM_ID" 18080:80 --address 127.0.0.1
```

Segunda terminal de PowerShell en Windows, sustituyendo usuario e IP:

```powershell
ssh -L 18080:127.0.0.1:18080 USUARIO_VM@IP_DE_LA_VM
```

Abrir `http://localhost:18080` en Windows. Si kubectl está configurado directamente en el host contra la VM, puede ejecutar el port-forward allí y abrir el mismo URL sin túnel SSH. Capturar comandos y navegador, explicando que es el respaldo; no lo presentar como evidencia de NodePort o LoadBalancer.

## 16. Capturas y cierre

Usar [EVIDENCIAS.md](EVIDENCIAS.md), exactamente A-F. Para cada captura registrar comando, resultado, qué demuestra, carné, fecha/hora y reloj visible. No incluir passwords, archivo `.secret`, kubeconfig, tokens ni JSON/YAML del Secret. `kubectl get all` no incluye PVC/ConfigMaps/Secrets; capturarlos separadamente con los comandos seguros.

```bash
date
kubectl -n "$NS" get all -o wide
kubectl -n "$NS" get pvc
kubectl -n "$NS" get configmaps
kubectl -n "$NS" get secret "db-credentials-$TEAM_ID"
kubectl -n "$NS" get svc
kubectl -n "$NS" get endpoints
```

El entregable final del docente es un PDF de evidencias con capturas reales explicadas y organizadas A-F; este repositorio entrega la guía y plantilla, no capturas inventadas. Generar ese PDF una vez realizadas las pruebas. Preparar a cada integrante para explicar namespace, Deployment/ReplicaSet, PVC, ConfigMap/Secret, DNS y exposición.

## Diagnóstico

**Service sin endpoints:** comparar selector y labels, después Ready:

```bash
kubectl -n "$NS" get svc "inventario-$TEAM_ID" -o yaml
kubectl -n "$NS" get pods --show-labels
kubectl -n "$NS" describe deployment "inventario-$TEAM_ID"
kubectl -n "$NS" get endpoints "inventario-$TEAM_ID"
```

**Backend no conecta:** confirmar ConfigMap, presencia del Secret, Service PostgreSQL y logs. No imprimir credenciales:

```bash
kubectl -n "$NS" get configmap "app-config-$TEAM_ID" -o yaml
kubectl -n "$NS" get secret "db-credentials-$TEAM_ID"
kubectl -n "$NS" get svc "postgres-$TEAM_ID"
kubectl -n "$NS" get endpoints "postgres-$TEAM_ID"
kubectl -n "$NS" logs "deployment/inventario-$TEAM_ID" -c wait-postgres
kubectl -n "$NS" logs "deployment/inventario-$TEAM_ID" -c inventario --tail=80
kubectl -n "$NS" describe pods -l "app=inventario,team=$TEAM_ID"
```

Si el bootstrap falló en un volumen vacío y dejó datos parciales, analizar logs y completar/reparar inicialización; la imagen no repetirá todos los scripts sobre PGDATA existente. No borrar PVC automáticamente. Cambiar un password del Secret con PVC existente tampoco cambia el password del rol SQL.

**Gateway 502 o CrashLoop:** comprobar Services/endpoints, nginx y DNS. Nginx necesita que existan los Services cuando carga la configuración:

```bash
kubectl -n "$NS" get svc
kubectl -n "$NS" get endpoints
kubectl -n "$NS" get configmap "gateway-config-$TEAM_ID" -o yaml
kubectl -n "$NS" logs "deployment/gateway-$TEAM_ID" --tail=80
kubectl -n "$NS" exec "deployment/gateway-$TEAM_ID" -- nginx -t
kubectl -n "$NS" run dns-check --image=busybox:1.37 --restart=Never --rm -i -- nslookup "inventario-$TEAM_ID"
```

`dns-check` es un Pod temporal de diagnóstico, no un Deployment duplicado para demostrar escalado. Si queda terminado por una interrupción, eliminar solo ese Pod.

**Frontend 404 en recarga:** confirmar que el frontend contiene `try_files $uri $uri/ /index.html;`, gateway reenvía `/` al frontend y se usa la imagen 2.0 correcta. Revisar Network/console para distinguir 404 de assets de rutas API.

**PVC Pending:** comprobar StorageClass y eventos:

```bash
kubectl get storageclass
kubectl -n "$NS" describe pvc "postgres-data-$TEAM_ID"
kubectl -n "$NS" describe pods -l "app=postgres,team=$TEAM_ID"
kubectl -n "$NS" get events --sort-by=.metadata.creationTimestamp
```

**ImagePullBackOff:** comprobar nombres/tags en Docker Hub, publicación efectiva, acceso de red y arquitectura del nodo. Si las imágenes son privadas, se necesita un imagePullSecret local autorizado; no se versiona ni se supone existente.

**LoadBalancer Pending/host inaccesible:** revisar ServiceLB, eventos del Service, uso del puerto 8080, red Bridge/NAT y firewall del host/VM. No editar IPs internas ni atribuir una IP externa ficticia. No abrir firewall automáticamente sin entender la red del equipo.

**ConfigMap actualizado pero comportamiento antiguo:** nginx carga al iniciar y las variables de entorno se leen al crear el contenedor. Aplicar el ConfigMap y reiniciar consumidores:

```bash
kubectl apply -f "k8s/gateway/gateway-configmap.yaml"
kubectl -n "$NS" rollout restart "deployment/gateway-$TEAM_ID"
kubectl -n "$NS" rollout status "deployment/gateway-$TEAM_ID" --timeout=180s
```

Las modificaciones de la prueba de réplicas se hacen directamente en el YAML versionado de `k8s/`. La comprobación opcional del ID no modifica archivos ni sobrescribe esas réplicas.
