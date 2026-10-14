# Evidencias del Proyecto 2

Equipo **Glitch Tec**, ID `glitch-tec`, namespace `proyecto-2-glitch-tec`, repositorio final **[TaTan09/proyecto_2_glitch-tec](https://github.com/TaTan09/proyecto_2_glitch-tec)**, rama final de entrega `main`. Las siete imágenes `javierini/elquetzal-*:2.0` están publicadas según confirmación del equipo; las capturas y pruebas de VM/k3s siguen pendientes. Cada captura debe indicar qué se hace, el comando, resultado real y qué criterio acredita. Mantener reloj del sistema y carné identificable en todas las capturas pertinentes. No mostrar passwords, tokens, kubeconfig ni YAML/JSON del Secret. Preparar al final un PDF con estas seis secciones exactamente A-F.

## A

Manifiestos y namespace (15 puntos).

- [ ] Namespace `proyecto-2-glitch-tec` y explicación de DNS frente a `proyecto_2_glitch-tec`.
- [ ] Ocho Deployments propios y namespace en los YAML.
- [ ] Siete imágenes propias en Docker Hub `javierini`, tag 2.0; identificar carné del responsable y correspondencia con el equipo.
- [ ] Pods Running **y Ready**, con imágenes y nombres del equipo.
- [ ] `kubectl get all -n "$NS" -o wide`, reloj visible.
- [ ] Hostname del Pod con ID y hostname real de la VM visible.

Capturas sugeridas: `A01-namespace`, `A02-deployments-imagenes`, `A03-pods-get-all`, `A04-hostname`. Registrar fecha, autor/carné y enlace/imagen del Docker Hub real.

## B

Persistencia (20 puntos).

- [ ] PVC 1Gi RWO `local-path` Bound y montaje `/var/lib/postgresql/data`.
- [ ] Producto con SKU/carné identificable antes de eliminar el Pod.
- [ ] Nombre/UID del PostgreSQL original y comando que elimina **solo el Pod**.
- [ ] ReplicaSet recrea PostgreSQL con nombre/UID distinto.
- [ ] Mismo registro después, desde el navegador y consulta API; mismo PVC conservado.

Capturas sugeridas: `B01-pvc-montaje`, `B02-registro-antes`, `B03-delete-pod`, `B04-pod-recreado`, `B05-registro-despues`. No eliminar PVC/PV/namespace como parte de esta prueba.

## C

Configuración desacoplada (20 puntos).

- [ ] ConfigMap con hosts, puertos, bases y umbral, sin passwords.
- [ ] Deployments usando `configMapKeyRef` y `secretKeyRef`.
- [ ] `kubectl get secret` mostrando solamente nombre/tipo/cantidad de claves.
- [ ] Código del backend que toma DB_HOST/PORT/NAME/USER/PASSWORD del entorno.
- [ ] Consulta/CRUD exitoso usando esas variables, sin credenciales expuestas.
- [ ] Archivo real de credenciales ignorado por Git y ausente del repositorio; no fotografiar su contenido.

Capturas sugeridas: `C01-configmap`, `C02-referencias-secret`, `C03-backend-variables`. No usar `kubectl get secret -o yaml/json` ni imprimir env.

## D

Ciclo de vida (15 puntos).

- [ ] YAML de inventario con `replicas: 3` editado.
- [ ] `kubectl apply -f "k8s/app/inventario-deployment.yaml"`.
- [ ] Tres Pods Ready y estado del Deployment.
- [ ] Comando eliminando un Pod de inventario identificado.
- [ ] Pod nuevo creado automáticamente por el ReplicaSet y tres Ready recuperados.

Capturas sugeridas: `D01-yaml-tres`, `D02-apply-pods`, `D03-delete`, `D04-reemplazo`. No usar `kubectl scale` como evidencia principal ni otro Deployment.

## E

Exposición y gateway (20 puntos).

- [ ] Siete Services ClusterIP y endpoints no vacíos.
- [ ] NodePort 30080 y navegador del host entrando por IP real de la VM.
- [ ] LoadBalancer 8080, EXTERNAL-IP real obtenida con kubectl y prueba de acceso; si falla, registrar el fallo y el diagnóstico.
- [ ] ConfigMap nginx: `/` a frontend y cinco `/api/...` a DNS de Services.
- [ ] Rutas `/`, `/inventario`, `/pedidos`, productos, dashboard, clientes, catálogo y pedidos.
- [ ] Recarga de rutas SPA sin 404.
- [ ] `port-forward` de respaldo con URL del navegador y túnel si se ejecuta en la VM; describirlo como respaldo.

Capturas sugeridas: `E01-clusterip-endpoints`, `E02-nodeport`, `E03-loadbalancer`, `E04-gateway-dns`, `E05-rutas`, `E06-portforward`. No inventar IP ni presentar port-forward como si fuera LoadBalancer.

## F

Aplicación funcional (10 puntos), incluyendo trazabilidad de integrantes exigida por el enunciado.

- [ ] CRUD Inventario: creación desde navegador, consulta tras recarga, actualización, eliminación confirmada.
- [ ] URL corresponde al gateway servido por Kubernetes.
- [ ] Registro CRUD tiene nombre/SKU con carné del responsable.
- [ ] Dashboard muestra Grupo Glitch Tec y los seis integrantes reales.
- [ ] Respuesta de `GET /api/clientes` corrobora origen desde PostgreSQL.
- [ ] Mario Arévalo 1072123; Jonatán Chávez 1185023; Javier Ruano 1207022; Javier Aldana 1184523; Hector Pérez 1304223; José Rosales 1143323.

Capturas sugeridas: `F01-crear`, `F02-consultar`, `F03-editar`, `F04-eliminar`, `F05-seis-carnes`, `F06-clientes-api`. Junto a cada imagen incluir fecha/hora, carné, URL, acción, resultado y explicación breve. Las pruebas de pedidos complementan la funcionalidad; no reemplazan el CRUD de Inventario.
