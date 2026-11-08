# Primera vuelta: seis templates de Enrollments

[Volver a la vista general](../README.md)

**Usaremos seis aplicaciones pequeñas del mismo dominio para practicar la
plataforma en Nest, Spring y Go.** Esta decisión reemplaza a Courses como primer
consumidor de la nueva plataforma. El laboratorio Courses/Floci sigue siendo
un antecedente implementado.

Decisión: 8 de noviembre de 2026. Los nombres y el alcance están elegidos; la
implementación de estos seis repositorios no se ha verificado todavía.

## Aplicaciones elegidas

| Repositorio | Tecnología | Organización interna |
| --- | --- | --- |
| `paideia-template-nest-hexagonal` | Nest | Hexagonal |
| `paideia-template-nest-layered` | Nest | Por capas |
| `paideia-template-spring-hexagonal` | Spring Boot | Hexagonal |
| `paideia-template-spring-layered` | Spring Boot | Por capas |
| `paideia-template-go-hexagonal` | Go | Hexagonal |
| `paideia-template-go-layered` | Go | Por capas |

Cada uno implementa Enrollments de forma autónoma. Son alternativas comparables,
no seis servicios que deban llamarse entre sí para completar una inscripción.
Cada aplicación tiene imagen, datos temporales y despliegue propios.

## Alcance inicial

- Un módulo de negocio: `enrollments`.
- API HTTP pequeña con el mismo contrato funcional en las seis variantes.
- Repositorio en memoria, seguro para el modelo de concurrencia del lenguaje.
- Sin base externa, archivo de datos, ORM, migraciones ni PVC.
- Sin llamadas a Courses, Config Server, discovery, colas ni pagos obligatorios.
- Configuración mínima por ambiente, healthcheck y Dockerfile por aplicación.
- Una réplica inicial por aplicación y despliegues seleccionados, no seis obligatorios.

El contrato HTTP inicial se detalla en el [contrato común](contrato-enrollments.md);
los campos de una inscripción no obligan a consultar otro servicio para esta práctica.
Se conservarán convenciones comunes de puerto configurable, cierre ordenado y
salida de logs para que la plataforma no necesite lógica por framework.

## Qué significa "en memoria"

Los datos viven dentro del proceso. Reiniciar, reemplazar un Pod, actualizar su
imagen o retirar el ambiente los pierde. No es persistencia durable.

Con dos réplicas, cada una tiene su propio repositorio: una escritura atendida
por A no aparecerá necesariamente en una lectura atendida por B. Un Service o un
ALB no sincroniza esas memorias. Practicar dos réplicas servirá para observar
orquestación y esta limitación; no se presentará como una base consistente.

La persistencia compartida y las migraciones se incorporarán en una segunda
vuelta. Los datos de esta primera vuelta son descartables por definición.

## Dos decisiones de arquitectura independientes

| Pregunta | Opciones |
| --- | --- |
| ¿Cómo se organiza el código dentro de una aplicación? | Por capas o hexagonal; agrupación por módulos de negocio |
| ¿Cómo se organiza el sistema y su despliegue? | Una aplicación con varios módulos o servicios con responsabilidades y despliegues independientes |

Tener una carpeta, varios `@Module`, un único JAR o varios paquetes no basta para
clasificar un sistema como microservicios. La cantidad de repositorios tampoco.

Estos templates serán **bases de aplicaciones organizadas por módulos, con un
único módulo inicial**. Podrán crecer agregando módulos en el mismo despliegue,
o utilizarse como base de un servicio independiente dentro de un sistema mayor.
No necesitan elegir prematuramente una etiqueta de microservicio o monolito modular.

## Organización por negocio, luego por arquitectura

Ejemplo conceptual para las variantes por capas:

```text
enrollments/
├── api/                 Entrada HTTP y contratos
├── application/         Operaciones o servicios de aplicación
├── domain/              Modelo y reglas que correspondan
└── persistence/         Implementación en memoria
```

Ejemplo conceptual para las variantes hexagonales:

```text
enrollments/
├── domain/              Modelo de negocio independiente de infraestructura
├── application/
│   ├── usecases/        Operaciones de Enrollments
│   └── ports/           Contratos necesarios, incluido el repositorio
└── adapters/
    ├── inbound/http/    Entrada HTTP
    └── outbound/memory/ Implementación del repositorio en memoria
```

Son responsabilidades equivalentes, no una obligación de copiar literalmente
los mismos nombres de carpetas entre lenguajes. La variante por capas puede tener
interfaces; no tiene que ser código acoplado para diferenciarse de la hexagonal.

## Cómo encaja en cada tecnología

| Tecnología | Punto de arranque | Módulo de negocio |
| --- | --- | --- |
| Nest | `main.ts` y módulo raíz | `EnrollmentsModule`, providers y API de módulo explícita |
| Spring Boot | Una clase `@SpringBootApplication` | Paquete `enrollments` con API y componentes internos |
| Go | Un ejecutable, por ejemplo `cmd/api` | Paquetes bajo `internal/enrollments`, ensamblados en el arranque |

Spring no exige varios módulos Maven/Gradle para organizar un monolito modular.
Spring Modulith puede ayudar después a verificar límites y documentar módulos,
pero no será una dependencia obligatoria del primer template.
[Módulos de Spring Modulith](https://docs.spring.io/spring-modulith/reference/fundamentals.html).

En Nest un módulo agrupa proveedores y controla qué expone al contenedor de
inyección; no es una frontera de despliegue por sí mismo. Una API HTTP también
puede participar como microservicio sin utilizar transportes de mensajería de Nest.
[Módulos Nest](https://docs.nestjs.com/modules).

En Go un módulo de `go.mod` es una unidad de distribución/dependencias del lenguaje,
distinta del módulo funcional Enrollments. No necesitamos varios `go.mod` para
organizar responsabilidades de negocio.
[Organización de proyectos Go](https://go.dev/doc/modules/layout).

## Recorrido de infraestructura ajustado

1. Piloto propuesto: `paideia-template-nest-layered`, por continuidad con Nest.
2. Kubernetes local: Deployment, Service, configuración, salud y acceso HTTP.
3. Argo CD: reconciliar la versión elegida desde Git.
4. EKS temporal: preparar identidad/red, desplegar y comprobar down/up.
5. GitHub Actions: construir/promover sin despertar un clúster retirado.
6. Repetir con una variante Spring y una Go; luego con las variantes hexagonales.

Las seis comparten la infraestructura común. Solo cambian construcción, imagen,
comando y parámetros necesarios. No se exige ejecutar las seis simultáneamente.
El tamaño del nodo y los ejemplos de costo se validarán contra las cargas elegidas.
