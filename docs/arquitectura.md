# Arquitectura y organización

[Volver a la vista general](../README.md)

**Los proyectos definen su aplicación; la plataforma ofrece maneras comunes de
construirla, desplegarla y operarla.** Admitiremos distintas tecnologías y tipos
de aplicación. Los árboles siguientes son el diseño objetivo, todavía por implementar.

## Dos responsabilidades diferentes

| Plano | Qué controla | Herramienta |
| --- | --- | --- |
| Infraestructura AWS | Red, identidades, ECR, EKS y nodos | Terraform |
| Aplicaciones Kubernetes | Deployments, Services, Jobs y configuración | Argo CD usando Helm |

GitHub Actions coordina el bootstrap: crea la infraestructura e instala Argo CD
con una versión y configuración definidas. Después Argo CD administra las cargas.
El proceso de reconstrucción vive fuera de EKS y funciona aunque no haya clúster.

Un recurso tendrá un responsable. Terraform y Argo CD no administrarán el mismo
objeto Kubernetes. Tampoco declararemos un ALB con Terraform si lo genera un
controlador Kubernetes desde un Ingress.

## Repositorios

### `paideia-infra-terraform`: piezas AWS

```text
paideia-infra-terraform/
├── modules/
│   ├── network/             # VPC, subredes y conectividad elegida
│   ├── eks/                 # Clúster y grupo de nodos administrado
│   └── container-registry/  # ECR y retención de imágenes
└── examples/                # Ejemplos pequeños, sin credenciales
```

Los módulos reciben parámetros, sin nombres ni rutas de POCs concretos. No habrá
un módulo por cada recurso AWS: extraeremos unidades con una responsabilidad
comprensible y usos comprobados. Primero puede existir una composición pequeña;
no necesitamos generalizar todas las posibilidades antes de desplegar el primer template.

El nombre identifica código específico de Terraform, no toda la infraestructura.
Si elegimos otra herramienta en el futuro, su implementación tendrá un espacio
propio; cambiar el nombre del repositorio no convierte los módulos entre herramientas.

### `paideia-infra-platform`: plataforma Kubernetes común

```text
paideia-infra-platform/
├── bootstrap/argocd/         # Versión y configuración inicial de Argo CD
├── charts/
│   ├── application/         # API/web o worker; Service opcional
│   └── migration/           # Job con Flyway u otra herramienta
└── docs/                    # Diseño y guías de la plataforma
```

Una API utiliza Deployment y Service; un worker puede no tener entrada HTTP.
Una migración usa Job y una tarea programada puede usar CronJob. El chart no
obligará a todas las aplicaciones a utilizar todos los recursos.

Observabilidad y External Secrets Operator serán componentes opcionales. Para
herramientas externas usaremos charts versionados; no copiaremos su implementación.

### `paideia-infra-automations`: automatización reutilizable

```text
paideia-infra-automations/
├── .github/workflows/
│   ├── ci-node.yml          # Validación de aplicaciones Node
│   ├── ci-java.yml          # Validación de aplicaciones Java
│   ├── ci-go.yml            # Validación de aplicaciones Go
│   └── build-image.yml      # Construcción/publicación común de imágenes
└── scripts/                # Solo auxiliares que realmente necesitemos
```

Esta estructura es el objetivo, no workflows ya implementados. El repositorio
reúne procesos reutilizables, incluso para proyectos que no utilizan Kubernetes.
Los consumidores mantienen workflows pequeños con sus eventos y parámetros.
`environments` conserva las entradas de promoción y up/down con su política concreta;
`automations` aporta los procedimientos comunes. Las referencias compartidas se versionan.
Platform puede tener workflows propios para validar sus charts, llamando a las
piezas reutilizables de automations, sin duplicarlas.

### `paideia-infra-environments`: composición concreta

```text
paideia-infra-environments/
├── foundation/
│   └── terraform/            # Estado remoto, identidad CI y ECR conservados
├── clusters/
│   ├── local/
│   │   └── bootstrap/       # Argo CD y aplicación raíz local
│   └── sandbox/
│       ├── environment.yaml # Política propia del workflow, no manifiesto Kubernetes
│       ├── terraform/       # Red, EKS y nodos temporales
│       ├── bootstrap/       # Valores y versiones de plataforma del ambiente
│       └── applications/    # Applications de Argo CD de proyectos habilitados
├── pocs/
│   ├── paideia-poc-aws-microservices/
│   │   ├── local-aws/       # Laboratorio actual con estado independiente
│   │   └── kubernetes/
│   │       ├── local/       # Valores y versiones para Kubernetes local
│   │       └── sandbox/     # Valores y versiones para EKS
│   └── paideia-poc-microservices-spring/
│       └── kubernetes/      # Se incorpora conservando su monorepo
└── .github/workflows/
    ├── validate.yml
    ├── promote.yml
    ├── environment-up.yml
    └── environment-down.yml
```

Los demás pilares tendrán carpetas cuando lo necesiten. El registro de
aplicaciones apunta a los valores del proyecto; no guardaremos copias divergentes.
Argo CD observará rutas elegidas expresamente, sin intentar aplicar documentos,
políticas o archivos Terraform como objetos Kubernetes.

`foundation` y `clusters/sandbox/terraform` tendrán estados distintos. El primer
bootstrap del backend se documentará aparte: no puede depender de un bucket aún
inexistente. El estado de Floci jamás se reutilizará contra AWS real.

## Contrato de una aplicación

| Dato | Para qué sirve |
| --- | --- |
| Proyecto, aplicación y ambiente | Identidad, nombres, etiquetas y permisos |
| Repositorio, ruta y Dockerfile | Construir desde un repo simple o un monorepo |
| Imagen y digest; imagen de migración opcional | Elegir exactamente qué ejecutamos |
| Tipo de carga y comando | API, worker o trabajo |
| CPU, memoria, réplicas y salud | Solicitar capacidad y comprobar disponibilidad |
| Variables y referencias a secretos | Configurar sin incorporar secretos a Git |
| Dependencias | Bases, colas y endpoints que realmente utiliza |
| Política de datos | Descartar, conservar o restaurar |
| Promoción y exposición | Automática/manual y acceso interno/externo |

El chart recibe estos datos; no adivina dependencias por el nombre del proyecto.
Su esquema de valores se validará al implementarlo.

## Monorepos y repositorios por servicio

Ambos son válidos. La imagen y el despliegue pertenecen al servicio. Un monorepo
puede generar varias imágenes con el mismo SHA de origen sin desplegarlas todas.
Los cambios en el padre Maven o librerías compartidas deben activar validaciones
de sus consumidores, además de los cambios dentro de una carpeta particular.

El POC Spring ya tiene módulos bajo `services/` y `platform/`: no necesita rehacerse.
La plataforma admite Java, Node, Python u otras tecnologías con imágenes compatibles.

## Un responsable por recurso

| Recurso | Responsable del diseño nuevo |
| --- | --- |
| VPC, subredes, EKS y nodos | Terraform del ambiente |
| Backend, ECR e identidades de CI | Terraform de la base conservada |
| Instalación inicial de Argo CD | Bootstrap externo con valores versionados |
| Cargas y componentes Kubernetes | Argo CD después del bootstrap |
| Tablas y cambios de esquema | Migraciones de la aplicación |
| ALB generado desde Ingress | AWS Load Balancer Controller; Terraform crea sus permisos |
| Selección de versiones | Configuración de ambientes actualizada por promoción |
| Valores de secretos | Canal privado al almacén; Git conserva referencias |

Los recursos ALB/ECS existentes de Floci siguen bajo su Terraform actual. El nuevo
diseño no mueve sus estados ni transfiere su propiedad automáticamente.

## Ambientes y aislamiento

Primero tendremos un clúster local y un sandbox EKS compartido. Cada proyecto
tendrá namespace, configuración, credenciales y permisos propios. Un namespace
organiza; el aislamiento necesita además RBAC, permisos AWS y políticas de red.
No se alojarán cargas no confiables suponiendo que un namespace basta para aislarlas.

Producción tendrá decisiones propias de disponibilidad, identidad y datos. No
heredará la eliminación de datos de laboratorio ni compartirá sus credenciales.
No habrá Config Server global obligatorio.

La región inicial propuesta es `us-east-1`, por continuidad con el laboratorio
y las estimaciones. El grupo de nodos inicial será administrado, On-Demand y
pequeño; el tamaño se valida con consumo real, no solo con el precio.

Para el primer laboratorio sin NAT se evaluarán nodos en subredes públicas con
salida explícita, permisos y reglas de acceso restringidas; las aplicaciones no
se publican por esa decisión. No es una recomendación general para producción.
Los nodos privados necesitan una salida viable o endpoints cuyo costo también
debe calcularse. El API de EKS y el acceso del runner requieren diseño explícito.

## Ver y acceder

- Local: Docker Desktop y Argo CD muestran Kubernetes; Floci Dash muestra AWS emulado.
- Remoto: consola EKS, Argo CD y `kubectl` muestran las cargas reales.
- Primer acceso: Service interno y `kubectl port-forward`.
- Entrada HTTP posterior: Ingress y ALB compartido con rutas por aplicación.

Un Service Kubernetes ofrece descubrimiento/acceso a Pods; no equivale a un
servicio ECS. API Gateway será una práctica posterior, no un requisito del despliegue.
