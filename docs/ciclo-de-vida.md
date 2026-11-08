# Levantar, retirar y reconstruir

[Volver a la vista general](../README.md)

**El sandbox remoto se recrea para cada sesión cuando haga falta. Retirarlo
significa eliminar su infraestructura temporal, no suspender los Pods.**

Estado: procedimiento objetivo. `environment-up` y `environment-down` son workflows
por implementar y comprobar; esta página no contiene un script de borrado listo
para ejecutar contra el laboratorio actual.

## Tres decisiones independientes

| Decisión | Opciones | Significado |
| --- | --- | --- |
| Disponibilidad del ambiente | `up` / `down` | Crear o retirar infraestructura temporal |
| Promoción por aplicación | Automática / manual | Cambiar o conservar la versión elegida en Git |
| Política de datos | Descartable / persistente | Poder recrear datos de prueba o conservar información |

Congelar despliegues conserva los recursos y sus cargos. Poner réplicas o nodos
en cero tampoco elimina EKS. Las aplicaciones pueden estar detenidas y seguir
existiendo discos, balanceadores o bases externas facturables.

## Política prevista del ambiente

Ejemplo conceptual de `clusters/sandbox/environment.yaml`:

```yaml
name: sandbox
runtime: down
promotionDefault: manual
dataPolicy: disposable
```

Estos nombres son un contrato propio que deberán leer nuestros workflows; no son
opciones incorporadas de AWS, Terraform o Argo CD. Cambiar el archivo sin esos
workflows no ejecuta ninguna operación. Argo CD no aplicará este archivo como
un manifiesto. La promoción puede sobrescribirse por aplicación.

El sandbox comenzará retirado y la promoción se probará en manual antes de
habilitar el modo automático para los servicios elegidos. Una sesión solicita
explícitamente `up`; un merge de aplicación nunca solicita esa transición.

## Qué se conserva y qué se retira

| Grupo | Ejemplos | Ciclo de vida |
| --- | --- | --- |
| Base conservada | Estado S3, ECR, identidad OIDC y roles de CI | Estado Terraform independiente; no entra en el down cotidiano |
| Ambiente temporal | VPC dedicada, EKS, grupo de nodos y discos temporales | Se crea en up y se elimina en down |
| Recursos de Kubernetes | Aplicaciones, Argo CD y componentes opcionales | Se reconstruyen desde Git |
| Recursos externos creados por controladores | ALB, target groups y recursos asociados | Se eliminan desde su recurso Kubernetes antes de retirar controladores |
| Datos declarados persistentes | Respaldos o base externa elegida | Conservación y costo explícitos, independientes del clúster |

Al retirar el primer sandbox se conservará la base mínima para reconstruirlo.
Eso no garantiza costo cero. Una purga de la base sería otra operación,
deliberada y excepcional, con exportación previa de lo que deba conservarse.

## Levantar: environment-up

1. Identificar cuenta, región, ambiente y revisión de configuración. Tomar el
   bloqueo de operación del ambiente y comprobar que no haya un down en curso.
2. Verificar base conservada, acceso del plan AWS, política de datos y presupuesto.
   El backend y OIDC requieren un bootstrap inicial separado.
3. Marcar la intención `up` y ejecutar un plan Terraform de la composición temporal.
4. Aplicar el plan elegido, esperar el clúster y los nodos y configurar su acceso.
5. Obtener valores privados por el canal definido e instalar Argo CD mediante Helm.
   Las credenciales de repositorios privados también se reconstruyen desde ese canal.
6. Registrar fuentes Git y aplicaciones con las versiones fijadas. Crear/restaurar
   dependencias y ejecutar migraciones cuando existan; desplegar las cargas en el
   orden probado. Los templates iniciales en memoria no requieren esos pasos de datos.
7. Comprobar salud y una operación de negocio. Mostrar acceso, versiones, recursos
   creados y lo que se cobrará mientras permanezcan.

Si algo falla después de crear AWS, el workflow debe informar qué quedó creado.
Un workflow fallido no equivale a un ambiente retirado. Se podrá reintentar o
ejecutar down sobre ese estado parcial; no se abandonarán recursos silenciosamente.

## Retirar: environment-down

1. Tomar el mismo bloqueo del ambiente, rechazar nuevas operaciones de creación
   y registrar intención `down` antes de iniciar limpieza.
2. Bloquear la reconciliación que podría recrear aplicaciones o entradas durante
   la retirada. Incluir aplicaciones raíz y ApplicationSets, si existen; no basta
   modificar una aplicación hija si un padre vuelve a habilitarla.
3. Comprobar la política de datos. Para datos persistentes, verificar respaldo o
   separación de la base y el procedimiento de restauración antes de eliminarlos.
4. Eliminar Ingress y Services de tipo LoadBalancer administrados por el sandbox.
   Mantener sus controladores y permisos funcionando hasta confirmar que AWS liberó
   balanceadores y dependencias. Esperar finalizers, sin forzarlos para ocultar fallos.
5. Retirar aplicaciones y datos descartables. Gestionar PVC, volúmenes y snapshots
   según su política; borrar un namespace no demuestra que desapareció un disco.
6. Eliminar la instalación de plataforma que corresponda y aplicar el plan de
   destrucción del estado Terraform temporal. Eliminar el grupo de nodos y EKS;
   no detener instancias EC2 sueltas que un grupo intentaría reemplazar.
7. Verificar recursos restantes en AWS por identificadores y etiquetas de propiedad.
   Informar cuáles se conservaron y qué costos siguen existiendo.

AWS documenta la limpieza de recursos expuestos antes de retirar el clúster.
[Eliminación de EKS](https://docs.aws.amazon.com/eks/latest/userguide/delete-cluster.html).

Una caída de Argo CD o un fallo de finalizers puede requerir intervención antes
de continuar. La salida correcta es mostrar el recurso pendiente, no declarar
un down exitoso ni forzar un borrado general de la cuenta.

## Dónde se serializan las operaciones

Los workflows que modifican infraestructura vivirán en `paideia-infra-environments`
y compartirán un grupo de concurrencia por ambiente. El bloqueo remoto de Terraform
protege además su estado. Son protecciones diferentes.

La concurrencia de GitHub se limita a su ámbito de repositorio: escribir el mismo
nombre de grupo en varios repositorios no crea un bloqueo global. Los pipelines
de aplicación publican/promueven; no crean ni destruyen EKS por su cuenta.

El backend S3 tendrá cifrado, versionado, acceso limitado y bloqueo con
`use_lockfile`. Conservaremos el estado fuera del clúster; puede contener valores
sensibles. [Backend S3 de Terraform](https://developer.hashicorp.com/terraform/language/backend/s3).

## Qué ocurre con Git mientras está retirado

Las construcciones pueden seguir publicando imágenes. En promoción automática,
Git registra el nuevo digest aunque no haya clúster. En manual conserva la versión
elegida. Al hacer up se despliega esa selección de Git, no una búsqueda ambigua
de la imagen `latest` ni necesariamente el último commit de todos los servicios.

Una migración vuelve a consultar su historial. Si la base se conserva, aplica lo
pendiente; si era descartable y se recrea, aplica el esquema desde cero y los datos
de prueba necesarios. Conservar SQL no conserva los registros de negocio.

## Opciones posteriores

- Recordatorio o retirada programada después de una sesión, una vez probado down.
- Tiempo máximo previsto de sesión y alarmas de gasto.
- Automatización de respaldos para proyectos con datos persistentes.

Un cron de GitHub puede retrasarse o fallar. Las alarmas de presupuesto también
pueden reflejar consumo con demora. Ninguno se considerará un límite duro de gasto
ni sustituirá la comprobación del cierre.

## Validación antes de llamar a esto reproducible

| Caso | Resultado necesario |
| --- | --- |
| Up desde cero | Aplicación disponible y versión identificable |
| Up repetido | Sin duplicar infraestructura ni perder datos |
| Down normal | EKS y recursos temporales ausentes; inventario conservado visible |
| Down tras fallo de creación | Limpieza de recursos parciales sin perder el estado |
| Down repetido | Detecta ambiente ya retirado y no afecta la base conservada |
| Main durante down | Publicación/promoción permitida; ninguna recreación del clúster |
| Reconstrucción | Misma configuración lógica, con nuevos IDs/endpoints posibles |
| Datos persistentes | Respaldo restaurado y verificado, no solo archivo generado |
