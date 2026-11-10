# Decisiones de diseño

[Volver a la vista general](../README.md)

**Este registro explica las elecciones vigentes y cuándo revisarlas.** Acordar
una decisión no significa que su implementación ya exista.

Fecha base: 8 de noviembre de 2026.

Actualización del 9 de noviembre de 2026: se separa la automatización reutilizable
de platform antes de implementar los primeros workflows. D06 reemplaza la propuesta
inicial de tres repositorios, que agrupaba automatizaciones y charts en platform.

| ID | Decisión | Razón y consecuencia | Revisar cuando… |
| --- | --- | --- | --- |
| D01 | Kubernetes como destino común, local y EKS | Aprovecha imágenes existentes y permite practicar una plataforma reutilizable | Otra carga requiera un servicio distinto |
| D02 | AWS real con sesiones temporales | EKS permanente supera USD 10–20/mes | Exista presupuesto/necesidad de servicio continuo |
| D03 | GitOps con Argo CD dentro de cada clúster | El estado deseado vive en Git; el clúster se reconstruye con bootstrap externo | Haya varios clústeres o una operación permanente |
| D04 | Terraform administra AWS; Argo CD administra cargas | Cada recurso tiene un propietario y un flujo identificable | Se introduzca un controlador de infraestructura con alcance explícito |
| D05 | Promoción automática/manual independiente de up/down | Un merge puede preparar un despliegue sin encender EKS | Cambie la política de ambientes |
| D06 | Cuatro repositorios: terraform, platform, automations y environments | Separa módulos AWS, plataforma Kubernetes, automatización reutilizable y composición concreta; las automatizaciones sirven también fuera de Kubernetes | La división produzca duplicación real o gestión innecesaria |
| D07 | Monorepos y repositorios por servicio admitidos | La aplicación es la unidad desplegable; no se rehace el POC Spring | Equipos, permisos o ciclos justifiquen separar un repositorio |
| D08 | Workflows y charts reutilizables, con versiones elegidas | Evita copiar pipelines y actualizaciones involuntarias de todos los consumidores | Aparezca una segunda implementación que revele límites del contrato |
| D09 | Un sandbox compartido, namespaces por proyecto | Reduce el costo del plano de control y simplifica aprendizaje | Se necesite aislamiento fuerte, producción o capacidad diferente |
| D10 | Estado y artefactos con ciclo separado del clúster | Son necesarios para reconstrucción y trazabilidad; pueden conservar costo | Haya una política explícita de purga o archivo |
| D11 | Nuevo sandbox con datos de prueba descartables | Permite retirar recursos; esquema y datos de ejemplo se recrean | Un proyecto tenga datos que deban sobrevivir |
| D12 | Secretos fuera de Git y fuente privada fuera de EKS | Reconstruir no depende de un secreto perdido con el clúster | Se requiera rotación o un gestor diferente |
| D13 | Compartir componentes según alcance | Argo/observabilidad por clúster; Config Server específico del proyecto | Permisos, versiones o impacto de fallos requieran separar |
| D14 | Primera vuelta con seis templates de Enrollments; un piloto a la vez | Compara Nest, Spring y Go con variantes por capas/hexagonal, en memoria y sin migraciones | La primera reconstrucción esté comprobada y se incorpore persistencia |
| D15 | GitHub Pages se mantiene | El sitio no necesita EKS | Sus requisitos reales cambien |
| D16 | Floci y ECS permanecen como laboratorios | Conserva el aprendizaje y evita mezclar estados de emulación con AWS | Se retiren deliberadamente esos laboratorios |
| D17 | Mantener el nombre paideia-infra-terraform | Contiene módulos específicos de esa herramienta; un nombre genérico no hace portable la implementación | Se adopte realmente otra herramienta de infraestructura |

## Alcance inicial de las decisiones

- GitHub Actions con OIDC para AWS; las sesiones no se basan en claves AWS permanentes.
- Helm para plantillas comunes; no añadimos otro sistema de plantillas al mismo tiempo.
- Argo CD instalado por nosotros; EKS Auto Mode y EKS Capabilities no son requisitos iniciales.
- Nodos administrados On-Demand inicialmente; Spot se estudia después.
- Región propuesta `us-east-1`; tamaño y salida de red deben validarse antes del apply.
- Sin RDS, NAT, ALB, API Gateway, Vault ni Grafana obligatorios en el primer despliegue.
- La política de promoción se prueba en manual y se habilita en automático por aplicación.

## Decisiones aún abiertas

| Pendiente | Cuándo resolver |
| --- | --- |
| Acceso a EKS y cuotas del plan gratuito | Antes de crear recursos remotos |
| Recursos/versiones finales, red y acceso del runner | Al preparar el primer plan AWS |
| Tamaño real del nodo y límites de las cargas | Con requests y consumo observados |
| Retención de imágenes, logs, respaldos y datos | Antes de activar sus operaciones automáticas |
| Esquema exacto de valores/políticas y permisos entre repositorios | Al implementar charts y promoción |
| Dominio, HTTPS y acceso a interfaces administrativas | Al añadir entrada externa permanente |

No se elegirá una política de producción a partir de una simplificación del
sandbox. Si una restricción de cuenta bloquea EKS, revisaremos el camino remoto;
no consideraremos confirmado ese acceso ni cambiaremos de plan por este documento.

## Cómo cambiar una decisión

Registrar el problema observado, alternativas, efecto en costos/datos y la nueva
decisión. Actualizar las páginas afectadas y conservar el antecedente cuando
ayude a entender el cambio. Las prácticas comprobadas se registran en
[Estado y recorrido](estado-y-recorrido.md), no como promesas de este registro.
