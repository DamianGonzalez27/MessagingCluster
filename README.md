# Messaging Cluster Infraestructura

Este proyecto gestiona la infraestructura necesaria para una cuenta AWS spoke que implementa un cluster de mensajería con Kafka y RabbitMQ.

## Estructura del proyecto

- `infra/`
  - `vpc-template.yaml`: define la VPC spoke, cuatro subnets privadas (2 para producción, 1 para dev y 1 para QA), el attachment al Transit Gateway y una única tabla de ruteo.
  - `kubernetes-cluster.yaml`: despliega un cluster EKS preparado para producción usando las subnets de producción.
  - `ec2-develop-cluster.yaml`: crea instancias EC2 Ubuntu Server para los ambientes bajos (dev y QA).

- `configs/`
  - `kubernetes-config.yaml`: contiene la configuración del cluster Kubernetes, incluyendo el controlador de ingress y la estructura básica para Kafka y RabbitMQ.
  - `kafka-config.yaml`: describe el despliegue de un cluster Kafka en Kubernetes con Zookeeper y un StatefulSet de brokers.
  - `rabbitmq-config.yaml`: describe el despliegue de un cluster RabbitMQ en Kubernetes con StatefulSet y servicios.
  - `docker-compose.yaml`: define la implementación de Kafka y RabbitMQ para las máquinas EC2 de los ambientes bajos (dev y QA).

## Uso

1. Despliega la infraestructura base con los templates de CloudFormation.
2. Aplica `kubernetes-config.yaml` en el cluster EKS.
3. Aplica `kafka-config.yaml` y `rabbitmq-config.yaml` para levantar los servicios en Kubernetes.
4. Usa `docker-compose.yaml` en las instancias EC2 de dev y QA para arrancar los servicios locales.

## Notas

- Todos los templates son parametrizables y exportan outputs para permitir su reutilización en otros stacks.
- `vpc-template.yaml` centraliza el enrutamiento en una sola tabla que apunta todo el tráfico hacia el Transit Gateway.
- El cluster EKS usa una configuración gestionada con un grupo de nodos de tres instancias.
- Las instancias EC2 para desarrollo y QA se despliegan con Ubuntu Server y Docker preinstalado.
