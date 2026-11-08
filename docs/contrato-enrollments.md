# Contrato inicial de Enrollments

[Primera vuelta de templates](templates-iniciales.md) · [Vista general](../README.md)

**Las seis variantes ofrecen el mismo comportamiento mínimo.** Este es el
contrato de referencia para implementarlas; no afirma que las APIs ya existan.
La organización por capas/hexagonal y el lenguaje no cambian el contrato HTTP.

## Operaciones

| Método y ruta | Resultado esperado |
| --- | --- |
| `GET /health` | `200`, cuerpo `{"status":"ok"}` |
| `POST /enrollments` | `201`, inscripción creada |
| `GET /enrollments` | `200`, arreglo de inscripciones, vacío al iniciar |
| `GET /enrollments/{id}` | `200`, inscripción encontrada; `404` si no existe |

Entrada de creación:

```json
{
  "studentId": "fe4c8135-ea41-44f6-acaa-bb3f631af36a",
  "courseId": "4773031f-b05e-4adb-9da6-a64ee55713da"
}
```

Salida de creación o consulta individual, con valores generados por la aplicación:

```json
{
  "id": "dd6fd2b5-dff7-4d23-8cba-a729a97ecff2",
  "studentId": "fe4c8135-ea41-44f6-acaa-bb3f631af36a",
  "courseId": "4773031f-b05e-4adb-9da6-a64ee55713da",
  "createdAt": "2026-11-08T15:00:00.000Z"
}
```

## Reglas iniciales

- `studentId` y `courseId` son obligatorios y deben ser UUID válidos.
- La aplicación genera `id` y fecha UTC en formato ISO 8601.
- Datos inválidos o un identificador de ruta mal formado producen `400`.
- La combinación alumno/curso no se repite dentro del mismo proceso: `409`.
- No se consulta la existencia del alumno o curso en otro servicio.
- No hay pagos, cancelación, autenticación ni paginación en esta vuelta.
- Los códigos HTTP y el modelo de éxito son comunes. El contrato inicial no
  exige un texto exacto de error ni un orden específico de la lista.

La creación y la comprobación de duplicados deben ser coherentes dentro del
proceso, considerando la concurrencia de cada lenguaje. Las variantes se probarán
con creación, lectura, duplicado y entrada inválida.

## Contrato de ejecución para la infraestructura

| Propiedad | Convención inicial |
| --- | --- |
| Entrada | HTTP; escuchar en `0.0.0.0` dentro del contenedor |
| Puerto | Variable `PORT`, valor predeterminado `8080` |
| Salud | `/health`, sin depender de bases o servicios externos |
| Registro | Salida estándar y salida de errores del proceso |
| Terminación | Manejar el cierre del proceso sin requerir un servidor adicional |
| Almacenamiento | Solo memoria, una instancia del repositorio por proceso |
| Réplicas | Una inicialmente; dos tienen datos independientes |
| Datos iniciales | Vacío; reiniciar o actualizar pierde las inscripciones |
| Publicación | Imagen por template, versión inmutable y digest registrado |

El puerto del contenedor no será un puerto distinto por tecnología: Docker y
Kubernetes permiten reutilizar `8080` en cargas separadas. Los puertos publicados
en el computador sí deben distinguirse. Para el piloto se propone `28001:8080`,
independiente del `28080` usado por el ALB de Floci.

## Primer template

`paideia-template-nest-layered`, con esta organización prevista:

```text
src/
├── main.ts
├── app.module.ts
├── health/
│   └── health.controller.ts
└── enrollments/
    ├── enrollments.module.ts
    ├── api/
    │   ├── enrollments.controller.ts
    │   └── create-enrollment.dto.ts
    ├── application/
    │   └── enrollments.service.ts
    ├── domain/
    │   └── enrollment.ts
    └── persistence/
        └── in-memory-enrollments.repository.ts
```

El controller traduce HTTP; el servicio ejecuta la operación; el repositorio
guarda los datos del proceso. La imagen y el contrato se verifican antes de
incorporar Kubernetes, Argo CD y el despliegue remoto. Los nombres anteriores
no indican que estos archivos se hayan generado todavía.
