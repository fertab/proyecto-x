### Documentación de Arquitectura

La siguiente documentación describe la arquitectura de la aplicación Proyecto X desplegada en AWS, detallando cada componente y decisión de diseño.

---

#### Frontend

El frontend de la aplicación se compone de dos aplicaciones: Home y Checkout, desplegadas en Amazon Elastic Container Service (ECS) para garantizar alta disponibilidad.

- **Dominio DNS de Route53 con DNS Failover**: Los usuarios se conectan al dominio DNS de Route53, configurado con DNS Failover para redirigir automáticamente el tráfico en caso de una falla en la región principal.
- **AWS WAF**: El tráfico del usuario atraviesa AWS WAF para protección contra ataques.
- **ALB (Application Load Balancer)**: El tráfico llega al ALB del frontend, distribuyendo la carga entre las instancias ECS.

##### Configuración de Security Groups
- Puertos 80 y 443 abiertos para permitir el tráfico HTTP y HTTPS entre los componentes del frontend.

##### Configuración de Tablas de Ruta
- Las tablas de ruta están configuradas para enrutar el tráfico a través de dos zonas de disponibilidad para garantizar tolerancia a fallos.

**Tablas de Ruta Frontend - us-east-1**
| Destination  | Target   |
|--------------|----------|
| 10.0.0.0/24  | Local    |
| 10.0.1.0/24  | Local    |
| 10.0.2.0/24  | Local    |
| 10.0.3.0/24  | Local    |

**Tablas de Ruta Frontend - us-east-2**
| Destination  | Target   |
|--------------|----------|
| 172.0.0.0/24 | Local    |
| 172.0.1.0/24 | Local    |
| 172.0.2.0/24 | Local    |
| 172.0.3.0/24 | Local    |

##### Auto Scaling
- Se implementa Auto Scaling para el frontend, proporcionando alta disponibilidad y escalabilidad automática.

---

#### Backend

El backend de la aplicación consiste en tres funciones Lambda: Payments, Products y Shipping.

- **Lambda Functions**: Desplegadas en AWS Lambda para una ejecución sin servidor y alta disponibilidad.

##### Acceso a Recursos
- **Products Lambda**: Accede a la base de datos RDS PostgreSQL vía IAM Roles API Call para obtener datos.
- **Shipping y Payments Lambda**: Acceden a AWS API Gateway vía IAM Roles API Call para interactuar con servicios externos.
- Todas las Lambda tienen acceso a los buckets S3 de Pagos y Envíos vía IAM Roles API Call.

---

#### Almacenamiento

Dos buckets de Amazon S3 son utilizados para almacenar datos de pagos y envíos.

- **Bucket S3**: Almacenamiento de metadatos de pagos y órdenes de envío.

---

#### Base de Datos RDS

La base de datos PostgreSQL de RDS se configura con una replicación de sincronización de 5ms RTT, y opera en modalidad Multi-AZ para alta disponibilidad.

- **Replicación Multi-AZ**: Garantiza la disponibilidad continua de datos en caso de falla de una zona de disponibilidad.

#### Tablas de Ruta - RDS (us-east-1)

| Destination  | Target   |
|--------------|----------|
| 10.0.4.0/24  | Local    |
| 10.0.5.0/24  | Local    |

#### Tablas de Ruta - RDS (us-east-2)

| Destination  | Target   |
|--------------|----------|
| 172.0.4.0/24 | Local    |
| 172.0.5.0/24 | Local    |


---

#### Estrategia de Disaster Recovery

Se implementa una estrategia de recuperación ante desastres Multi-Site Active/Active entre las regiones us-east-1 y us-east-2 de AWS.

---

#### Monitoreo y Notificaciones

La salud de la región se monitorea utilizando Amazon CloudWatch, con notificaciones enviadas al usuario a través de Amazon SNS.

- **Amazon CloudWatch**: Monitoreo continuo de métricas y eventos de la región.
- **Amazon SNS**: Notificaciones de eventos críticos enviadas a los usuarios suscritos.

---

#### Subscripción a Personal Health Dashboard

Los usuarios están suscritos al Personal Health Dashboard de AWS para recibir información sobre el estado de salud de la infraestructura y la región.

---

#### Zonas de Disponibilidad

Tanto el Frontend como la base de datos RDS PostgreSQL están distribuidos en dos zonas de disponibilidad (AZs) para garantizar la tolerancia a fallos y la alta disponibilidad.

#### NACL's (Network Access Control List) - Se ha implementado un NACL entre el Frontend y la base de datos RDS PostgreSQL para restringir el acceso desde el Frontend hacia RDS y viceversa.

**Frontend - us-east-1**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.4.0/24 (Subred RDS)
  - Rango de IP de destino: 10.0.0.0/22 (Todas las subredes Frontend)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 10.0.4.0/24 (Subred RDS)

**RDS - us-east-1**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 10.0.4.0/24 (Subred RDS)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.4.0/24 (Subred RDS)
  - Rango de IP de destino: 10.0.0.0/22 (Todas las subredes Frontend)

**Frontend - us-east-2**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.4.0/24 (Subred RDS)
  - Rango de IP de destino: 172.0.0.0/22 (Todas las subredes Frontend)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 172.0.4.0/24 (Subred RDS)

**RDS - us-east-2**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 172.0.4.0/24 (Subred RDS)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.4.0/24 (Subred RDS)
  - Rango de IP de destino: 172.0.0.0/22 (Todas las subredes Frontend)

---

Esta documentación proporciona una descripción detallada de la arquitectura de la aplicación Proyecto X en AWS, incluyendo cada componente y decisión de diseño tomada para garantizar la disponibilidad, escalabilidad y seguridad de la aplicación.