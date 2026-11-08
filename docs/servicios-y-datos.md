# Servicios compartidos, secretos y datos

[Volver a la vista general](../README.md)

**Compartir la manera de instalar un componente y compartir una instalación son
decisiones independientes.** Podemos reutilizar un chart PostgreSQL y levantar
instancias separadas. También podemos compartir un servidor con bases y permisos
distintos cuando tenga sentido.

Estado: política objetivo. La incorporación de operadores y servicios se hará
por etapas; no es un requisito instalar todo antes de ejecutar el primer template.

La [primera vuelta](templates-iniciales.md) usa memoria: no hay base ni volumen
de aplicación que conservar, ni Job de migración. Las políticas de persistencia
de esta guía se aplicarán al incorporar esos recursos en la segunda vuelta.

## Alcance propuesto de cada componente

| Componente | Alcance inicial | Qué sigue perteneciendo al proyecto |
| --- | --- | --- |
| Argo CD | Uno por clúster | Aplicaciones, versiones y parámetros elegidos |
| External Secrets Operator | Componente compartido por clúster, cuando se incorpore | Referencias a secretos e identidades autorizadas |
| Grafana y recolección de métricas | Opcionales, compartidos por clúster/ambiente | Instrumentación, dashboards y necesidades de retención |
| Config Server Spring | Del proyecto que lo requiere | Configuración y compatibilidad de sus clientes |
| PostgreSQL | Por proyecto al empezar; servidor compartido opcional después | Base, usuario, migraciones, datos y recuperación de cada aplicación |
| Colas y tópicos | Recursos identificados por proyecto/ambiente | Contratos de mensajes, productores, consumidores e idempotencia |
| Registro ECR | Base común con repositorios de imágenes por aplicación/trabajo | Imagen, versiones y política de adopción |

Una sola instalación compartida puede ser un Deployment dentro de Kubernetes,
no necesariamente una máquina EC2 dedicada. No habrá un servidor global que
mezcle por defecto todos los proyectos y ambientes.

```text
Clúster sandbox
├── argocd                       Administración de despliegues
├── platform-secrets             Operador opcional de secretos
├── platform-observability       Grafana y fuentes de datos opcionales
├── poc-aws-microservices         Courses y demás aplicaciones del POC
└── poc-microservices-spring      Servicios y Config Server propios del POC

Fuera del clúster
├── S3                           Estado y respaldos elegidos
├── ECR                          Imágenes conservadas
└── Parameter Store              Fuente privada de secretos/configuración elegida
```

Los nombres del dibujo son ilustrativos. La separación requiere permisos y red,
no solo carpetas, namespaces o prefijos. Producción tendrá una separación propia.

## Configuración normal y secretos

| Tipo | Dónde vive en el diseño |
| --- | --- |
| Puerto, réplicas, flags y endpoints públicos | Valores versionados; ConfigMaps o variables |
| Contraseña, token o clave privada | Almacén privado; Git solo guarda la referencia |
| Reglas y estructura de acceso | Terraform y manifiestos declarativos revisables |
| Credenciales temporales de automatización | Sesión OIDC de GitHub hacia AWS |

La primera opción para secretos estáticos del sandbox es Parameter Store con
`SecureString`. Secrets Manager se evaluará para rotación y necesidades más
avanzadas. Es una decisión de alcance/costo del laboratorio: AWS recomienda
Secrets Manager para credenciales con sus controles específicos.
[Comparación oficial](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html).

Ejemplo de organización de referencias, no valores reales:

```text
/paideia/sandbox/poc-aws-microservices/courses/database-password
/paideia/sandbox/poc-aws-microservices/enrollments/database-password
```

Los prefijos ayudan a organizar; IAM decide el acceso. La integración del
operador tendrá roles limitados y stores adecuados por namespace. RBAC impedirá
que un proyecto elija arbitrariamente identidades o secretos de otro.
[External Secrets y Parameter Store](https://external-secrets.io/latest/provider/aws-parameter-store/).

```text
Fuente privada AWS → External Secrets Operator → Secret Kubernetes → Aplicación
```

La fuente AWS permanece al retirar EKS. El operador y las copias Kubernetes se
recrean en up. Para el primer ejercicio local se pueden crear Secrets manualmente
desde archivos privados ignorados; el origen privado no necesita estar automatizado
antes de aprender Deployment y Service.

Un Secret Kubernetes codificado en base64 no protege un valor publicado en Git.
Tampoco `sensitive = true` elimina un secreto del estado Terraform: limita su
presentación. El backend y los archivos privados requieren acceso restringido.
[Secrets de Kubernetes](https://kubernetes.io/docs/concepts/configuration/secret/).

No instalaremos Vault en la primera plataforma. Operarlo requiere su propia
persistencia, recuperación y administración. Si su estudio se vuelve un objetivo,
tendrá un laboratorio propio y no será la única fuente de las claves necesarias
para reconstruir el mismo clúster que lo contiene.

## Config Server y discovery del POC Spring

Ese POC estudia una arquitectura Spring y puede conservar Config Server, gateway
y discovery. Sus componentes se desplegarán como aplicaciones del proyecto con
las mismas plantillas comunes, sin hacerlos requisitos para los demás proyectos.

ConfigMaps/Secrets cubren entrega habitual de configuración, pero no equivalen
a todas las funciones de Spring Cloud Config. La elección depende del caso, no
de una regla de eliminar o imponer Config Server al usar Kubernetes.
[Spring Cloud Config](https://docs.spring.io/spring-cloud-config/reference/).

## Datos de laboratorio y datos valiosos

| Política | Qué permite down | Qué hace up |
| --- | --- | --- |
| Descartable | Eliminar la base/volumen temporal declarado | Crear base, ejecutar migraciones y datos iniciales de prueba |
| Persistente | Conservar la base externa o verificar respaldo/restauración antes de retirar almacenamiento | Reconectar o restaurar y comprobar datos |

La política descartable será la del **nuevo sandbox de práctica**. No cambia
automáticamente la política ni borra los volúmenes existentes de Floci.

Las migraciones crean/evolucionan estructura; los datos iniciales son otro paso.
Un backup de esquema o un archivo SQL de migración no recupera cursos, usuarios
o inscripciones creados durante una sesión. Un volumen persistente tampoco
garantiza por sí solo un respaldo ni su conservación al eliminar infraestructura.

Para datos persistentes registraremos origen, destino, retención y una prueba de
restauración mediante la aplicación. Si no se puede asegurar la conservación,
down falla con un motivo concreto antes de eliminar esa información.

## Usuarios y administración de bases

La aplicación y su migración tendrán credenciales con responsabilidades definidas.
En AWS real separaremos permisos de administración, migración y uso habitual
cuando el caso lo requiera. Las credenciales maestras compartidas que usamos en
Floci fueron una simplificación de aprendizaje.

Terraform crea infraestructura y configuración admitida por sus proveedores;
Flyway administra el esquema. Crear roles/bases PostgreSQL adicionales puede
requerir un paso declarativo específico, no un efecto automático de Compose.
Evitaremos que Terraform y scripts independientes modifiquen el mismo usuario
sin una propiedad definida. Los valores privados no se incorporarán a Git.

## Observabilidad cuando la retomemos

Grafana visualiza datos; necesita fuentes como un sistema de métricas o logs.
La instalación común puede servir a varios proyectos mediante etiquetas, permisos
y dashboards. Guardaremos dashboards y configuración en Git para reconstruirlos.

El historial requiere almacenamiento y retención explícitos. Si se conserva
fuera del clúster tendrá costos; si es descartable, retirarlo perderá ese historial.
No confundiremos recrear un dashboard con recuperar sus datos históricos.

## Cuándo compartir o separar más

Compartir reduce instalaciones y consumo. Separar puede convenir por permisos,
versiones incompatibles, independencia de datos, carga o impacto de fallos. La
decisión se toma por componente y ambiente; no por una regla de "todo compartido"
o "todo por proyecto". Primero reutilizaremos definiciones y añadiremos servicios
compartidos funcionando solo cuando haya un beneficio comprobable.
