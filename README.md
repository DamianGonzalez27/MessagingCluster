# Messaging Cluster - Infraestructura y Plataforma de Mensajería Híbrida

Este repositorio representa la infraestructura base para el aprovisionamiento y gobierno de nuestro Ecosistema de Mensajería Híbrido, diseñado para operar de forma aislada dentro de una cuenta satélite (Spoke) dedicada exclusivamente a la ingesta y distribución de eventos distribuidos. La solución orquesta de forma sinérgica Apache Kafka y RabbitMQ, proveyendo las bases de infraestructura necesarias para soportar Arquitecturas Orientadas a Eventos (EDA) de alta disponibilidad.

**Segmentación Estratégica de Casos de Uso**

El diseño del clúster evita la adopción ciega de una única tecnología, segmentando quirúrgicamente las herramientas según el patrón operativo requerido por el negocio:

- **RabbitMQ (Message Broker):** Configurado para la gestión de mensajería transaccional de alta fidelidad, enrutamiento complejo mediante exchanges personalizados y distribución de tareas asíncronas con garantías de entrega (AMQP protocol).
- **Apache Kafka (Distributed Commit Log):** Orientado al procesamiento y streaming de eventos masivos en tiempo real, persistencia de registros de auditoría inmutables y habilitación de patrones de integración coreográficos de alta velocidad entre múltiples microservicios.

**Características de Diseño e Infraestructura como Código**

- **Aislamiento de Entorno (Blast Radius Control):** Implementación empaquetada mediante contenedores y redes virtuales aisladas, asegurando que el tráfico de mensajería core de la organización esté protegido y segregado de las cargas de trabajo de cómputo tradicional.
- **Habilitador de Plataforma:** Funciona como un componente de autoservicio listo para producción; las células de desarrollo pueden conectarse a este clúster utilizando nuestro Async Framework heredando de forma nativa la topología de red y los estándares de seguridad de la plataforma.
- **Observabilidad:** Exports y endpoints de métricas (Prometheus) disponibles tanto en despliegues Kubernetes como en las instancias Docker Compose para ambientes bajos. Se incluyen exporters para Kafka y RabbitMQ.
- **Persistencia y Ciclo de Vida de Datos:** Todos los servicios críticos declaran volúmenes nombrados o PV/PVC en Kubernetes para asegurar durabilidad y recuperación. En ambientes bajos se usan volúmenes Docker nombrados para simular persistencia.
- **Seguridad Perimetral y Credenciales:** Evitar credenciales por defecto; los despliegues Kubernetes usan Secrets para credenciales de RabbitMQ. Las redes Docker están segmentadas en redes internas (`*_backend`) y de gestión (`*_mgmt`) para limitar la visibilidad de puertos de broker.
- **Modernización Kafka (KRaft):** Los manifiestos Kubernetes y las imágenes de Docker Compose están preparados para operar Kafka en modo KRaft (sin Zookeeper), facilitando la modernización operativa.

## Estructura del proyecto

- `infra/`
  - `vpc-template.yaml`: VPC spoke, subnets privadas para producción y entornos bajos, attachment a Transit Gateway y una tabla de ruteo única.
  - `kubernetes-cluster.yaml`: Plantilla CloudFormation para crear un cluster EKS productivo (control plane gestionado por AWS y un Nodegroup de 3 nodos worker).
  - `ec2-develop-cluster.yaml`: Plantilla CloudFormation para instancias EC2 Ubuntu en ambientes dev/qa con Docker y Docker Compose instalados.

- `configs/`
  - `kubernetes-config.yaml`: Recursos para preparar un cluster EKS (Ingress, StorageClass, ServiceAccounts) y anotaciones básicas para observabilidad.
  - `kafka-config.yaml`: Despliegue KRaft-ready de Kafka en Kubernetes con StatefulSet y exporter para Prometheus.
  - `rabbitmq-config.yaml`: StatefulSet de RabbitMQ con credenciales en `Secret`, PV/PVC y exporter para métricas.
  - `docker-compose.yaml`: Composiciones aisladas para dev/qa con redes internas, volúmenes nombrados, y exporters para observabilidad.

## Despliegue (resumen operativo)

1. Desplegar la VPC y recursos base: crear stack con `infra/vpc-template.yaml`.
2. Desplegar EKS: crear stack con `infra/kubernetes-cluster.yaml` y esperar a que el cluster esté activo.
3. Preparar cluster: aplicar `configs/kubernetes-config.yaml`.
4. Desplegar Kafka y RabbitMQ en Kubernetes: aplicar `configs/kafka-config.yaml` y `configs/rabbitmq-config.yaml`.
5. Ambientes bajos: arrancar `configs/docker-compose.yaml` en las instancias EC2 (dev/qa). Asegúrate de proporcionar las contraseñas a través de variables de entorno (`RABBITMQ_DEV_PASSWORD`, `RABBITMQ_QA_PASSWORD`).

## Notas operativas

- Los manifiestos Kubernetes incluyen anotaciones para Prometheus (`prometheus.io/scrape`) y despliegan exporters para Kafka y RabbitMQ que exponen métricas en puertos estándar (9308 para kafka-exporter, 9419 para rabbitmq-exporter).
- En Docker Compose las redes `*_backend` son internas (`internal: true`) para que sólo contenedores conectados a esa red puedan alcanzar los puertos de broker. Los puertos de gestión (UI) sí están expuestos en la red `*_mgmt` para tareas operacionales.
- KRaft en Kafka requiere coordinación de `controller.quorum.voters` y configuración de listeners. Los manifiestos incluidos muestran una configuración básica que debe revisarse y adaptarse para producción (cluster IDs, storage, límites y recursos, políticas de anti-affinity, y backups de metadatos).
- No uses credenciales por defecto en producción. Los Secrets incluidos son ejemplos; en AWS se recomienda usar AWS Secrets Manager o SSM Parameter Store y conectar usando IRSA o una integración segura.

Si quieres, puedo:
- Añadir plantillas de `ServiceMonitor` para Prometheus Operator.
- Generar roles y políticas IAM más restrictivas para los nodos EKS.
- Proveer scripts de bootstrap para inicializar IDs KRaft y realizar tests de reinicio/recuperación.
