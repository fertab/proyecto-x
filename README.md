## Proyecto X

### Documentación de Arquitectura

La siguiente documentación describe la arquitectura de la aplicación Proyecto X desplegada en AWS, detallando cada componente y decisión de diseño.

#### **Diagrama de arquitectura adjunto en el repositorio como 'Proyecto X.png'**
---

#### Frontend

El frontend de la aplicación se compone de dos aplicaciones: Home y Checkout, desplegadas en Amazon Elastic Container Service (ECS) para garantizar alta disponibilidad.

- **Dominio DNS de Route53 con DNS Failover**: Los usuarios se conectan al dominio DNS de Route53, configurado con DNS Failover para redirigir automáticamente el tráfico en caso de una falla en la región principal.
- **AWS WAF**: El tráfico del usuario atraviesa AWS WAF para protección contra ataques.
- **ALB (Application Load Balancer)**: El tráfico llega al ALB del frontend, distribuyendo la carga entre las instancias ECS.

##### Firewalls
- Security Group configurado con los puertos 80 y 443 para permitir el tráfico HTTP y HTTPS entre los componentes del frontend.

#### Configuración de las subredes
- Todas las subredes de la capa de Frontend y la base de datos PostgreSQL en RDS son privadas, lo que significa que no es posible acceder a ellas directamente desde Internet. Sin embargo, cuentan con un NAT Gateway configurado para permitir el acceso a Internet saliente de manera controlada.
  
- Únicamente el Application Load Balancer (ALB) tiene una subred de acceso público con un Internet Gateway configurado para que pueda ser alcanzado desde internet. El usuario después de acceder al dominio DNS de Route53 luego deberá atravesar el Web Application Firewall (WAF) el cual actúa como una capa de seguridad adicional al inspeccionar y filtrar el tráfico web, protegiendo así la aplicación de posibles ataques maliciosos antes de llegar al ALB.

##### Acceso a Backend
- Accede al Backend vía IAM Roles API Call.

## IAM Roles/Policies

#### Acceso Frontend --> Backend
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "lambda:InvokeFunction",
            "Resource": "arn:aws:lambda:region:account-id:function:nombre-de-la-funcion"
        }
    ]
}
```

#### Acceso Lambda Products --> RDS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds-db:connect"
      ],
      "Resource": "arn:aws:rds-db:region:account-id:dbuser:db-instance-id/dbusername"
    }
  ]
}
```

#### Acceso Lambdas --> AWS API Gateway

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "apigateway:Invoke"
            ],
            "Resource": "arn:aws:apigateway:region::/restapis/*"
        }
    ]
}
```
#### Acceso Lambdas --> Buckets S3

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::nombre-bucket-pagos/*",
                "arn:aws:s3:::nombre-bucket-envios/*"
            ]
        }
    ]
}
```

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

- **Lambda Functions**: Desplegadas en AWS Lambda para una ejecución sin servidor y alta disponibilidad. Las funciones de Lambda ya son altamente disponibles y escalables por diseño. No es necesario tomar medidas adicionales para garantizar la alta disponibilidad en esta capa

##### Acceso a Recursos
- **Products**: Accede a la base de datos RDS PostgreSQL vía IAM Roles API Call para obtener datos.
- **Shipping/Payments**: Acceden a AWS API Gateway vía IAM Roles API Call para interactuar con servicios externos.
- Todas las Lambda tienen acceso a los buckets S3 de Pagos y Envíos vía IAM Roles API Call.

---

#### Almacenamiento

Dos buckets de Amazon S3 son utilizados para almacenar datos de pagos y envíos.

- **Bucket S3**: Almacenamiento de metadatos de pagos y órdenes de envío.

---

#### Base de Datos RDS

La base de datos PostgreSQL en RDS se configura con una replicación de sincronización de 5ms RTT, y opera en modalidad Multi-AZ para alta disponibilidad.

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

Se implementa una estrategia de recuperación ante desastres Multi-Site Active/Active entre las regiones us-east-1 y us-east-2 de AWS que permite recuperarse en 5 minutos.

![image](https://github.com/fertab/proyecto-x/assets/8042545/6ddb7350-8fc5-49c3-b246-82449b3aaf18)


- Este plan de recuperación ante desastres es el más rápido en la restauración del sistema durante un evento de recuperación ante desastres.
- Multi-site es una copia uno a uno de la infraestructura que se encuentra y se ejecuta en otra región (o AZ), conocida como configuración activo-activo.
- Ofrece el mejor RTO (Recovery Time Objective) y RPO (Recovery Point Objective), ya que no se espera tiempo de inactividad y se debe experimentar poca o ninguna pérdida de datos, lo cual es acorde a lo requerido por el Proyecto X.
- Se puede utilizar un servicio DNS que admita el enrutamiento ponderado, como Route 53, para enrutar el tráfico de producción a diferentes sitios que brindan la misma aplicación de Proyecto X.
- Durante la conmutación por error, es posibe aumentar rápidamente la capacidad informática utilizando Autoscaling o cambiando el tamaño de las instancias a un tamaño mayor.
- Varios servicios en AWS, como RDS, ofrecen una función multi-AZ que permite aprovisionar recursos en una ubicación diferente para una configuración más tolerante a fallas.

---

#### Monitoreo y Notificaciones

La salud de la región se monitorea utilizando Amazon CloudWatch, con notificaciones enviadas al usuario a través de Amazon SNS.

- **Amazon CloudWatch**: Monitoreo continuo de métricas y eventos de la región.
- **Amazon SNS**: Notificaciones de eventos críticos enviadas a los usuarios suscritos.

---

#### Subscripción a Personal Health Dashboard

Los usuarios están suscriptos al Personal Health Dashboard de AWS para recibir información sobre el estado de salud de la infraestructura y la región.

---

#### Zonas de Disponibilidad

Tanto el Frontend como la base de datos PostgreSQL en RDS están distribuidos en dos zonas de disponibilidad (AZs) para garantizar la tolerancia a fallos y la alta disponibilidad.

#### NACL's (Network Access Control List)
- Se ha implementado un NACL entre el Frontend y la base de datos PostgreSQL en RDS para restringir el acceso desde el Frontend hacia RDS y viceversa.

**Frontend - us-east-1**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.4.0/23 (Subred RDS)
  - Rango de IP de destino: 10.0.0.0/22 (Todas las subredes Frontend)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 10.0.4.0/23 (Subred RDS)

**RDS - us-east-1**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 10.0.4.0/23 (Subred RDS)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 10.0.4.0/23 (Subred RDS)
  - Rango de IP de destino: 10.0.0.0/22 (Todas las subredes Frontend)

**Frontend - us-east-2**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.4.0/23 (Subred RDS)
  - Rango de IP de destino: 172.0.0.0/22 (Todas las subredes Frontend)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 172.0.4.0/23 (Subred RDS)

**RDS - us-east-2**:
- **Entrada**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.0.0/22 (Todas las subredes Frontend)
  - Rango de IP de destino: 172.0.4.0/23 (Subred RDS)
- **Salida**:
  - Número de regla: 100
  - Tipo: Denegar
  - Protocolo: Todos
  - Puerto de origen: Todos
  - Puerto de destino: Todos
  - Rango de IP de origen: 172.0.4.0/23 (Subred RDS)
  - Rango de IP de destino: 172.0.0.0/22 (Todas las subredes Frontend)

---

Esta documentación proporciona una descripción detallada de la arquitectura de la aplicación Proyecto X en AWS, incluyendo cada componente y decisión de diseño tomada para garantizar la disponibilidad, escalabilidad y seguridad de la aplicación.

---

## Recomendaciones

1. En el enunciado indica **"Database: PostgreSQL, la cual está desplegada sobre RDS. Todas las aplicaciones usan el mismo cluster de RDS."**
   
   Como buena práctica no considero que exista comunicación entre el Frontend y la base de datos, sino que esta se haga a través del Backend.
   Por ese motivo se configuró un NACL entre el Frontend y la base de datos, para no permitir el tráfico entre estas capas.
   Si de cualquier forma se quisiera esto, se configuraría un Security Group en RDS con el puerto 5432, se removería el NACL y se configuraría los accesos desde las Route Tables de cada capa, lo que permitiría el tráfico.

2. Como recomendación sugeriría implementar **CloudFront** como una capa de distribución de contenido para mejorar la entrega de contenido estático y dinámico, optimizando la velocidad de carga del sitio web y reduciendo la carga en los servidores de origen. Además, configurar una estrategia de caché efectiva ayudaría a maximizar los beneficios de rendimiento de CloudFront.

3. Además recomiendo, para garantizar la seguridad y disponibilidad de los datos, recomiendo el uso de **AWS Backup**.
   - Con AWS Backup será posible respaldar de manera automatizada los recursos críticos, incluyendo el Frontend, el Backend y la base de datos en RDS.
   - Esto proporcionará una capa adicional de protección de los datos, facilitando la restauración en caso de pérdida de información o falla del sistema, brindando tranquilidad y seguridad para la infraestructura en la nube.

4. **Despliegue Continuo**
   - Por último recomiendo el uso de repositorios de código como **AWS CodeCommit** para almacenar el código fuente de las aplicaciones.
   - Esto proporcionará un control de versiones centralizado y un historial de cambios para el código.

- **Pipeline:**
  - Es posible configurar un pipeline de despliegue continuo utilizando **AWS CodePipeline** el cual puede estar integrado con **AWS CodeBuild** que permitirá compilar y probar la aplicación, para luego desplegarla automáticamente.

# Fernando Taboada
