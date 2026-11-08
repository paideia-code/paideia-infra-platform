# Estado y recorrido

[Volver a la vista general](../README.md)

**La prioridad es practicar Kubernetes y AWS real con templates pequeños de
Enrollments en Nest, Spring y Go.** Tendrán almacenamiento en memoria y no
necesitarán bases externas ni migraciones. No hace falta terminar ECS.
[Alcance de la primera vuelta](templates-iniciales.md).

Revisión: 8 de noviembre de 2026. Distinguimos archivos presentes, resultados
informados en el chat y trabajo pendiente. No es un inventario en vivo de AWS.

## Lo que ya tenemos

| Parte | Evidencia y alcance |
| --- | --- |
| Floci, red y almacenamiento | Compose existente; ejecución local confirmada |
| ECR, dos PostgreSQL emulados y SQS/SNS | Terraform y resultados de `apply` compartidos |
| Cuatro aplicaciones y dos imágenes Flyway | Versiones `0.1.0` construidas y publicadas en ECR de Floci |
| POC completo mediante Compose | Inscripción, pago simulado y notificación confirmados |
| EC2 emulado | Aprovisionamiento y SSH practicados; no aloja las aplicaciones |
| ECS y ALB emulados para Courses | Configuración presente y funcionamiento confirmado |
| Dos réplicas de Courses | `desired_count = 2` en código y dos tareas informadas |
| Migración de Courses en ECS | Ejecución y recreación de tablas confirmadas |
| Floci Dash | Cambio de consola confirmado; resuelto el conflicto del puerto 4500 |
| POC Spring | Monorepo Maven con servicios y componentes de plataforma; compatible con el diseño |

La prueba de detener una tarea y observar su reemplazo fue propuesta; falta
registrar su resultado para marcarla validada. La migración ECS de Enrollments
también fue propuesta; su definición aún no estaba presente al revisar los archivos.

Consulta el [laboratorio local-aws](../../paideia-infra-environments/pocs/paideia-poc-aws-microservices/local-aws/README.md).

## Lo pendiente

- Los seis templates iniciales implementados y su contrato HTTP acordado.
- Kubernetes local y manifiestos del primer template verificados.
- Argo CD y sincronización desde Git.
- Workflows reutilizables, promoción y controles automático/manual.
- Estado remoto, identidad OIDC de GitHub y permisos AWS reales.
- EKS y procedimientos `environment-up` / `environment-down` probados.
- Módulos extraídos a `paideia-infra-terraform`.
- Gestión externa de secretos y política de datos ejecutable.

No se ha verificado aquí la publicación de los directorios de infraestructura
como repositorios GitHub. Tampoco está confirmada la disponibilidad de EKS en
la cuenta con plan gratuito. Esta comprobación precede al despliegue remoto.

## Recorrido prioritario

La preferencia es empezar hoy y avanzar en un par de días. Es una prioridad de
alcance, no una promesa sobre tiempos de provisión o habilitación de cuenta.

| Etapa | Qué haremos | Criterio de terminación |
| --- | --- | --- |
| 1. Kubernetes local | Primer template de Enrollments, memoria y acceso HTTP | Crear y leer una inscripción; observar Pods y pérdida de datos al reemplazar el proceso |
| 2. GitOps local | Argo CD y configuración versionada | Un cambio elegido en Git aparece en Kubernetes y en Argo CD |
| 3. AWS mínimo | Cuenta, costos, identidad, ECR, red y EKS | El primer template funciona en AWS y sus recursos facturables están identificados |
| 4. Retirar y reconstruir | Operación manual documentada y luego workflows | Se elimina el ambiente temporal y se reconstruyen las versiones elegidas |
| 5. Automatizar `main` | CI, promoción y políticas del ambiente | Manual conserva versión; automático la actualiza; un merge no despierta EKS |
| 6. Extender | Templates Spring/Go y variantes hexagonales; módulos comprobados | Tres tecnologías usan las mismas piezas de despliegue |

En AWS se implementarán los permisos y la red necesarios. Omitir IAM en Floci
fue una simplificación del laboratorio, no un patrón trasladable a la nube.

## Primer despliegue remoto pequeño

- Un template de Enrollments con almacenamiento en memoria, una réplica y datos descartables.
- EKS estándar y un grupo de nodos administrado; tamaño pendiente de validar.
- Argo CD instalado mediante Helm.
- Acceso inicial mediante `kubectl port-forward`; ALB en una práctica posterior.
- Sin RDS, NAT Gateway, Grafana ni API Gateway obligatorios para el primer recorrido.

La red se diseña antes de crear: evitar NAT no proporciona salida automática a
nodos privados. Se comprobarán DNS, descarga de imágenes y acceso al API.

## Después del primer recorrido

| Práctica | Momento previsto |
| --- | --- |
| Persistencia y Flyway como Job | Segunda vuelta, después de los templates en memoria |
| POC completo en Kubernetes | Después del primer recorrido con los templates |
| Actualización y rollback | Con dos versiones y compatibilidad de esquema comprobada |
| ALB y rutas | Al incorporar una segunda aplicación HTTP |
| API Gateway | Al estudiar gestión de APIs y autorización |
| Grafana y registros centralizados | Cuando retomemos observabilidad, pospuesta expresamente |
| HPA y Spot | Con recursos y tolerancia a interrupciones claros |
| Ansible y AMIs | En ejercicios de servidores EC2 administrados por nosotros |
| Refactorizar Spring | Solo si el proyecto lo necesita; el monorepo es válido |

## Registrar las prácticas

Guardar versión/commit, acción aplicada, resultado esperado y observado. No
publicar secretos ni estados Terraform. Un `/health` exitoso no sustituye una
escritura y lectura de negocio ni demuestra que todas las réplicas reciben tráfico.
