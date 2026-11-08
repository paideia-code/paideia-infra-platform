# Glosario breve

[Volver a la vista general](../README.md)

Consulta solo el término que necesites. Las equivalencias entre herramientas
sirven para orientarse; no significan que tengan todas las mismas funciones.

| Término | Explicación |
| --- | --- |
| Imagen | Paquete ejecutable con aplicación y dependencias |
| Contenedor | Ejecución concreta de una imagen |
| Registro / ECR | Lugar donde se publican y descargan imágenes |
| Tag | Nombre legible de una versión de imagen; debe mantenerse inmutable según nuestra política |
| Digest | Identificador del contenido exacto de una imagen |
| CI | Comprobaciones y construcción que acompañan cambios de código |
| CD | Proceso que entrega una versión a un ambiente |
| Promover | Elegir una imagen ya construida como versión deseada de un ambiente |
| GitOps | Expresar el estado deseado en Git y reconciliarlo con el ambiente |
| Argo CD | Herramienta que compara Git y Kubernetes y sincroniza sus diferencias |
| Bootstrap | Preparación inicial para que el resto del sistema pueda operar |
| Clúster | Conjunto de componentes que proporciona un Kubernetes operativo |
| Plano de control | Componentes que coordinan Kubernetes; en EKS AWS los administra y cobra por el clúster |
| Nodo | Máquina donde Kubernetes puede ejecutar cargas |
| Pod | Unidad de ejecución Kubernetes que contiene uno o varios contenedores relacionados |
| Deployment | Mantiene réplicas de Pods de una aplicación y gestiona sus actualizaciones |
| Réplica | Otra ejecución de la misma aplicación; no es automáticamente una copia de su base de datos |
| Service Kubernetes | Dirección/descubrimiento estable hacia un conjunto de Pods |
| Job | Trabajo que debe terminar, por ejemplo una migración |
| CronJob | Definición de trabajos Kubernetes con una programación |
| Namespace | Agrupación de recursos Kubernetes; el aislamiento necesita controles adicionales |
| ConfigMap | Configuración no secreta para cargas Kubernetes |
| Secret | Recurso Kubernetes para valores sensibles; base64 no permite publicarlos de forma segura |
| PVC / volumen | Solicitud de almacenamiento / almacenamiento; su retención depende de la configuración |
| Readiness | Comprobación de si una aplicación está lista para recibir tráfico |
| Liveness | Comprobación usada para decidir si necesita reiniciar el contenedor |
| Helm / chart / values | Herramienta de plantillas / paquete de recursos / parámetros de una instalación |
| Ingress | Reglas de entrada HTTP que requieren un controlador para funcionar |
| ELB / ALB | Servicio de balanceo AWS / su tipo orientado a HTTP y HTTPS |
| Listener | Entrada de un balanceador, con protocolo y puerto |
| Target Group | Grupo de destinos entre los que un balanceador dirige tráfico |
| EKS | Kubernetes administrado de AWS |
| ECS / task definition / task | Orquestador AWS / descripción de ejecución / ejecución concreta |
| Servicio ECS | Mantiene tareas ECS según una cantidad deseada; no es un Service Kubernetes |
| Fargate | Cómputo administrado para cargas compatibles, sin administrar sus servidores |
| Spot | Capacidad con descuento que puede ser interrumpida; no reduce el cargo del plano de control EKS |
| Terraform module | Conjunto reutilizable de configuración Terraform con entradas y salidas |
| Terraform state | Registro que vincula configuración y recursos; debe conservarse y protegerse |
| Plan / apply | Vista de cambios propuestos / ejecución de cambios de Terraform |
| Foundation | Base conservada entre sesiones, con estado independiente del sandbox |
| Up / down | En este diseño: crear / retirar infraestructura temporal, no solo arrancar/detener Pods |
| OIDC / IAM / RBAC | Identidad federada / permisos AWS / permisos Kubernetes |
| Migración / Flyway | Cambio versionado del esquema / herramienta que lo aplica y registra |
| Rollback | Volver a una versión anterior; no revierte automáticamente una migración de datos |
| Monorepo | Un repositorio con varias aplicaciones o componentes |
| Floci | Emulador local de APIs AWS; no demuestra costos ni seguridad de una cuenta real |

## Tres diferencias que conviene recordar

1. Un recurso declarado en un archivo todavía no está creado hasta que una
   herramienta lo aplica. Una task definition registrada tampoco inicia una tarea.
2. Tener dos réplicas mantiene dos ejecuciones; el escalado automático cambia la
   cantidad según reglas. Son prácticas diferentes.
3. Retirar un proceso, retirar un clúster y eliminar sus datos son operaciones
   diferentes. Cada una tiene consecuencias y costos propios.
