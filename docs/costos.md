# Costos y presupuesto

[Volver a la vista general](../README.md)

**Presupuesto de referencia: USD 10–20 al mes para el laboratorio. La estrategia
es practicar localmente y usar EKS durante sesiones cortas.** Un clúster permanente
no encaja en ese importe.

Referencias consultadas: **8 de noviembre de 2026**. Región de estimación:
**us-east-1**, Linux y On-Demand. Son precios de referencia y cálculos de diseño,
sin créditos, impuestos ni garantía de una factura final. Antes del primer apply
se verifican región, recursos concretos, capacidad disponible y precios vigentes.

## Qué cuesta aunque no reciba peticiones

| Recurso | Referencia | Con 730 horas de existencia |
| --- | --- | --- |
| EKS, soporte estándar | USD 0,10/clúster-hora | USD 73,00, sin nodos ni aplicaciones |
| Nodo EC2 t3.medium de ejemplo | USD 0,0416/hora | USD 30,37, sin disco ni otros cargos |
| Una IPv4 pública | USD 0,005/hora | USD 3,65 por dirección |
| ALB | USD 0,0225/hora de base | USD 16,43, más capacidad consumida y otros cargos |
| NAT Gateway, referencia de precio horario | USD 0,045/hora | USD 32,85, más procesamiento, IPv4 y transferencia aplicables |
| EBS gp3, tarifa de referencia | USD 0,08/GB-mes de almacenamiento | 20 GB: USD 1,60, sin IOPS/throughput adicionales |
| ECR privado, almacenamiento de referencia | USD 0,10/GB-mes | 1 GB: USD 0,10, más cargos aplicables de uso/transferencia |

Fuentes de cada tarifa: [EKS](https://aws.amazon.com/eks/pricing/),
[referencia EC2 publicada por AWS](https://aws.amazon.com/blogs/aws-cloud-financial-management/a-finops-guide-to-comparing-containers-and-serverless-functions-for-compute/),
[IPv4 y NAT](https://aws.amazon.com/vpc/pricing/),
[ALB](https://aws.amazon.com/elasticloadbalancing/pricing/),
[EBS](https://aws.amazon.com/ebs/pricing/) y
[ECR](https://aws.amazon.com/ecr/pricing/).

El nodo es un ejemplo contable, no una selección final ni garantía de capacidad
para todo el POC. Un nodo de 4 GB debe alojar también Kubernetes, Argo CD y sus
componentes. Se medirán recursos antes de aumentar aplicaciones o réplicas.
Las T3 pueden tener cargos adicionales por CPU excedente según su configuración.

## Un ejemplo pequeño para entender las horas

Supongamos EKS estándar, un nodo con la tarifa anterior, una IPv4 y un disco raíz
gp3 de 20 GB. Sin ALB, NAT, RDS ni observabilidad administrada.

```text
EKS                 0,10000 USD/h
EC2                 0,04160 USD/h
Una IPv4            0,00500 USD/h
Disco prorrateado    1,60 / 730 USD/h
Subtotal            ≈ 0,14879 USD/h
```

| Horas de ambiente al mes | Subtotal ilustrativo |
| --- | --- |
| 4 h | USD 0,60 |
| 20 h | USD 2,98 |
| 40 h | USD 5,95 |
| 80 h | USD 11,90 |
| 730 h | USD 108,62 |

**Estos subtotales no son el costo total.** Se suman base conservada, otras IP,
transferencia, peticiones, logs, respaldos, redondeos propios de cada servicio,
consumo durante creación/limpieza y cualquier recurso adicional.
El prorrateo del disco usa 730 horas como simplificación del modelo; AWS aplica
la unidad y reglas de facturación de cada servicio.

El tiempo de una sesión incluye más que escribir código: provisionar, probar,
corregir y retirar también ocupa recursos. Se revisará el inventario creado por
los módulos, incluidos recursos auxiliares, antes de usar el ejemplo como presupuesto.

## Qué cambia al practicar balanceadores

El ALB añade su tarifa base y capacidad LCU. El ejemplo oficial usa USD 0,008 por
LCU-hora. Un ALB público también añade direcciones públicas: con dos IPv4, solo
base e IP representan aproximadamente USD 23,73 por 730 horas, antes de LCUs,
transferencia y nodos. [Precios de balanceo](https://aws.amazon.com/elasticloadbalancing/pricing/).

Por eso el primer acceso remoto será `port-forward`. La práctica ALB será explícita
y temporal. Compartirlo entre aplicaciones reduce duplicación, pero no elimina
sus cargos de existencia ni de capacidad.

## Qué puede seguir cobrando después de down

| Recurso conservado | Cómo lo controlaremos |
| --- | --- |
| ECR | Retención limitada, protegiendo imágenes activas y de rollback |
| S3 de estado/respaldos | Versionado, tamaño y retención; no borrar el estado para ahorrar centavos |
| EBS y snapshots | Inventario de volúmenes retenidos y política por proyecto |
| Parámetros, secretos y KMS | Revisar tipo, nivel, claves y uso; no asumir que todo es gratuito |
| Logs/métricas externos | Retención explícita y evitar servicios opcionales sin necesidad |
| RDS u otra base persistente | Presupuesto propio; mantener datos fuera de EKS no elimina ese costo |
| IPv4, balanceadores o NAT olvidados | Down verifica su liberación y muestra cualquier recurso pendiente |

El costo de S3 también depende de peticiones y versiones conservadas.
[Precios S3](https://aws.amazon.com/s3/pricing/).
Parameter Store estándar no tiene el mismo modelo que Secrets Manager o claves
KMS: se verificarán los cargos de la combinación elegida.
[Comparación de almacenamiento de configuración y secretos](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html).

Argo CD instalado por nosotros no añade una tarifa de servicio AWS propia, pero
consume capacidad del nodo. GitHub Actions, almacenamiento de artefactos y minutos
tienen límites/cargos según cuenta y repositorios; se revisan además de AWS.
[Facturación de Actions](https://docs.github.com/en/billing/concepts/product-billing/github-actions).

## Decisiones para el primer presupuesto

1. Un sandbox compartido, retirado fuera de sesiones; no un EKS por POC.
2. Nodo pequeño On-Demand para aprender sin interrupciones Spot inicialmente.
3. Templates en memoria: sin PostgreSQL, migraciones ni volúmenes de aplicación
   en la primera vuelta. El disco raíz del nodo sigue existiendo. RDS se pospone.
4. Sin NAT por defecto: elegir una red viable y calcular cualquier alternativa.
5. Sin ALB hasta su práctica; acceso por port-forward.
6. Sin Grafana, API Gateway, Prometheus administrado ni EKS Capabilities obligatorios.
7. Estado/imágenes mínimos conservados y reporte al finalizar cada down.

Con estos límites, sesiones cortas parecen compatibles con USD 10–20 según el
modelo, pero falta comprobar el despliegue real. No se fijará una cantidad de
horas "seguras" sin medir la factura y los recursos adicionales.

## Spot y créditos

Spot se evaluará después como opción de nodos EC2, junto con tolerancia a
interrupciones y disponibilidad. No reduce el precio del plano de control EKS.
La modalidad Fargate Spot estudiada para ECS no debe asumirse disponible en EKS.

La cuenta indicada tiene plan gratuito. La habilitación concreta de EKS y cuotas
sigue pendiente. Un crédito es saldo que cubre consumo elegible, no ausencia de
costo ni autorización para todas las funciones. No asumiremos elegibilidad Spot.
[Planes de cuenta AWS](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html).

## Control al iniciar y cerrar una sesión

Antes: estimación concreta, región/cuenta y recursos identificados. Durante:
tiempo de sesión y revisión de recursos realmente creados. Después: down y
verificación del inventario restante y su retención. Las alarmas ayudan a detectar
desvíos, pero la información puede llegar con demora y no establece un límite duro.

Dominios y necesidades futuras de producción no están incluidos en el ejemplo.
La conversión a CLP depende del tipo de cambio y del medio de pago; estas cifras
se mantienen en USD para compararlas con las tarifas oficiales.
