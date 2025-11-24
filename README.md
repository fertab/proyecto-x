## Project X

### Architecture Documentation

The following documentation describes the architecture of the Project X application deployed on AWS, detailing each component and design decision.

#### **Architecture diagram attached in the repository as 'Proyecto X.png'**
---

#### Frontend

The application frontend consists of two applications: Home and Checkout, deployed on Amazon Elastic Container Service (ECS) to ensure high availability.

- **Route53 DNS Domain with DNS Failover**: Users connect to the Route53 DNS domain, configured with DNS Failover to automatically redirect traffic in case of a failure in the primary region.
- **AWS WAF**: User traffic passes through AWS WAF for attack protection.
- **ALB (Application Load Balancer)**: Traffic arrives at the frontend ALB, distributing the load among ECS instances.

##### Firewalls
- Security Group configured with ports 80 and 443 to allow HTTP and HTTPS traffic between frontend components.

#### Subnet Configuration
- All subnets in the Frontend layer and the PostgreSQL database in RDS are private, meaning they cannot be accessed directly from the Internet. However, they have a NAT Gateway configured to allow controlled outbound Internet access.
  
- Only the Application Load Balancer (ALB) has a public access subnet with an Internet Gateway configured so it can be reached from the internet. After accessing the Route53 DNS domain, the user must then pass through the Web Application Firewall (WAF), which acts as an additional security layer by inspecting and filtering web traffic, thus protecting the application from potential malicious attacks before reaching the ALB.

##### Backend Access
- The Frontend accesses the Backend via IAM Roles API Call.

## IAM Roles/Policies Frontend

#### Frontend --> Backend Access
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
##### Route Table Configuration
- Route tables are configured to route traffic through two availability zones to ensure fault tolerance.

**Frontend Route Tables - us-east-1**
| Destination  | Target   |
|--------------|----------|
| 10.0.0.0/24  | Local    |
| 10.0.1.0/24  | Local    |
| 10.0.2.0/24  | Local    |
| 10.0.3.0/24  | Local    |

**Frontend Route Tables - us-east-2**
| Destination  | Target   |
|--------------|----------|
| 172.0.0.0/24 | Local    |
| 172.0.1.0/24 | Local    |
| 172.0.2.0/24 | Local    |
| 172.0.3.0/24 | Local    |

##### Auto Scaling
- Auto Scaling is implemented for the frontend, providing high availability and automatic scalability.

---

#### Backend

The application backend consists of three Lambda functions: Payments, Products, and Shipping.

- **Lambda Functions**: Deployed on AWS Lambda for serverless execution with high availability. Lambda functions are already highly available and scalable by design. No additional measures are necessary to ensure high availability at this layer.

##### Resource Access
- **Products**: Accesses the RDS PostgreSQL database via IAM Roles API Call to retrieve data.
- **Shipping/Payments**: Access AWS API Gateway via IAM Roles API Call to interact with external services.
- All Lambdas are accessible from the Frontend as mentioned previously, and have access to the Payments and Shipping S3 buckets via IAM Roles API Call.

## IAM Roles/Policies Backend

#### Lambda Products --> RDS Access

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

#### Lambdas --> AWS API Gateway Access

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
#### Lambdas --> S3 Buckets Access

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
---

#### Storage

Two Amazon S3 buckets are used to store payment and shipping data.

- **S3 Bucket**: Storage of payment metadata and shipping orders.

---

#### RDS Database

The PostgreSQL database in RDS is configured with 5ms RTT synchronous replication and operates in Multi-AZ mode for high availability.

- **Multi-AZ Replication**: Ensures continuous data availability in case of an availability zone failure.

#### Route Tables - RDS (us-east-1)

| Destination  | Target   |
|--------------|----------|
| 10.0.4.0/24  | Local    |
| 10.0.5.0/24  | Local    |

#### Route Tables - RDS (us-east-2)

| Destination  | Target   |
|--------------|----------|
| 172.0.4.0/24 | Local    |
| 172.0.5.0/24 | Local    |


---

#### Disaster Recovery Strategy

A **Multi-Site Active/Active** disaster recovery strategy is implemented between AWS regions us-east-1 and us-east-2, allowing recovery in 5 minutes.

![image](https://github.com/fertab/proyecto-x/assets/8042545/6ddb7350-8fc5-49c3-b246-82449b3aaf18)


- This disaster recovery plan is the fastest in restoring the system during a disaster recovery event.
- Multi-site is a one-to-one copy of the infrastructure that is located and runs in another region (or AZ), known as an active-active configuration.
- It offers the best RTO (Recovery Time Objective) and RPO (Recovery Point Objective), as no downtime is expected and little or no data loss should be experienced, which is in line with what is required by Project X.
- A DNS service that supports weighted routing, such as Route 53, can be used to route production traffic to different sites that provide the same Project X application.
- During failover, it is possible to quickly increase computing capacity using Autoscaling or changing instance sizes to a larger size.
- Several AWS services, such as RDS, offer a multi-AZ feature that allows provisioning resources in a different location for a more fault-tolerant configuration.

---

#### Monitoring and Notifications

Region health is monitored using Amazon CloudWatch, with notifications sent to users through Amazon SNS.

- **Amazon CloudWatch**: Continuous monitoring of region metrics and events.
- **Amazon SNS**: Critical event notifications sent to subscribed users.

---

#### Personal Health Dashboard Subscription

Users are subscribed to AWS Personal Health Dashboard to receive information about the health status of the infrastructure and the region.

---

#### Availability Zones

Both the Frontend and the PostgreSQL database in RDS are distributed across two availability zones (AZs) to ensure fault tolerance and high availability.

#### NACLs (Network Access Control List)
- A NACL has been implemented between the Frontend and the PostgreSQL database in RDS to restrict access from the Frontend to RDS and vice versa.

**Frontend - us-east-1**:
- **Inbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 10.0.4.0/23 (RDS Subnet)
  - Destination IP range: 10.0.0.0/22 (All Frontend subnets)
- **Outbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 10.0.0.0/22 (All Frontend subnets)
  - Destination IP range: 10.0.4.0/23 (RDS Subnet)

**RDS - us-east-1**:
- **Inbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 10.0.0.0/22 (All Frontend subnets)
  - Destination IP range: 10.0.4.0/23 (RDS Subnet)
- **Outbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 10.0.4.0/23 (RDS Subnet)
  - Destination IP range: 10.0.0.0/22 (All Frontend subnets)

**Frontend - us-east-2**:
- **Inbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 172.0.4.0/23 (RDS Subnet)
  - Destination IP range: 172.0.0.0/22 (All Frontend subnets)
- **Outbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 172.0.0.0/22 (All Frontend subnets)
  - Destination IP range: 172.0.4.0/23 (RDS Subnet)

**RDS - us-east-2**:
- **Inbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 172.0.0.0/22 (All Frontend subnets)
  - Destination IP range: 172.0.4.0/23 (RDS Subnet)
- **Outbound**:
  - Rule number: 100
  - Type: Deny
  - Protocol: All
  - Source port: All
  - Destination port: All
  - Source IP range: 172.0.4.0/23 (RDS Subnet)
  - Destination IP range: 172.0.0.0/22 (All Frontend subnets)

---

This documentation provides a detailed description of the Project X application architecture on AWS, including each component and design decision made to ensure the application's availability, scalability, and security.

---

## Future improvement recommendations and comments

1. The statement indicates **"Database: PostgreSQL, which is deployed on RDS. All applications use the same RDS cluster."**
   
   As a best practice, I do not consider that there should be communication between the Frontend and the database, but rather that it should be done through the Backend.
   For this reason, a NACL was configured between the Frontend and the database, to not allow traffic between these layers.
   If this were desired in any case, a Security Group would be configured in RDS with port 5432, the NACL would be removed, and access would be configured from the Route Tables of each layer, which would allow traffic.

2. As a recommendation, I would suggest implementing **CloudFront** as a content distribution layer to improve the delivery of static and dynamic content, optimizing website load speed and reducing the load on origin servers. Additionally, configuring an effective caching strategy would help maximize CloudFront's performance benefits.

3. Furthermore, to ensure data security and availability, I recommend using **AWS Backup**.
   - With AWS Backup it will be possible to automatically backup critical resources, including the Frontend, Backend, and the RDS database.
   - This will provide an additional layer of data protection, facilitating restoration in case of data loss or system failure, providing peace of mind and security for the cloud infrastructure.

4. **Continuous Deployment**
   - I also recommend using code repositories such as **AWS CodeCommit** to store the application source code.
   - This will provide centralized version control and a change history for the code.

- **Pipeline:**
  - It is possible to configure a continuous deployment pipeline using **AWS CodePipeline** which can be integrated with **AWS CodeBuild** that will allow compiling and testing the application, and then deploying it automatically.

5. **Infrastructure as Code**
- Finally, I recommend deploying the infrastructure through IaC (Infrastructure as Code) with tools like Terraform, which allows defining all infrastructure as code, meaning that infrastructure changes can be made in a controlled and documented manner. This helps reduce errors, minimize downtime, and improve infrastructure security.

- Only as an example, I show below a template that serves to deploy the Frontend, Backend, and Database. This does not have exact data, but rather contains generic data.

## Frontend

```
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "frontend_vpc" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "subnet_us_east_1a" {
  vpc_id                  = aws_vpc.frontend_vpc.id
  cidr_block              = "10.0.0.0/24"
  availability_zone       = "us-east-1a"
}

resource "aws_subnet" "subnet_us_east_1b" {
  vpc_id                  = aws_vpc.frontend_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1b"
}

resource "aws_route_table" "frontend_route_table" {
  vpc_id = aws_vpc.frontend_vpc.id
}

resource "aws_route" "route_10_0_0" {
  route_table_id         = aws_route_table.frontend_route_table.id
  destination_cidr_block = "10.0.0.0/24"
  gateway_id             = "local"
}

resource "aws_route" "route_10_0_1" {
  route_table_id         = aws_route_table.frontend_route_table.id
  destination_cidr_block = "10.0.1.0/24"
  gateway_id             = "local"
}

resource "aws_route" "route_10_0_2" {
  route_table_id         = aws_route_table.frontend_route_table.id
  destination_cidr_block = "10.0.2.0/24"
  gateway_id             = "local"
}

resource "aws_route" "route_10_0_3" {
  route_table_id         = aws_route_table.frontend_route_table.id
  destination_cidr_block = "10.0.3.0/24"
  gateway_id             = "local"
}

resource "aws_security_group" "frontend_sg" {
  vpc_id = aws_vpc.frontend_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_autoscaling_group" "frontend_asg" {
  launch_configuration = aws_launch_configuration.frontend_lc.name
  vpc_zone_identifier  = [aws_subnet.subnet_us_east_1a.id, aws_subnet.subnet_us_east_1b.id]
  min_size             = 2
  max_size             = 4
  desired_capacity     = 2
}

resource "aws_launch_configuration" "frontend_lc" {
  image_id        = "ami-12345678" # Change to a valid AMI
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.frontend_sg.name]
  key_name        = "your-key-name"
}
```

## Backend

```
provider "aws" {
  region = "us-east-1"
}

resource "aws_lambda_function" "example" {
  filename      = "lambda_function_payload.zip"
  function_name = "example_lambda_function"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "lambda_function.handler"
  runtime       = "python3.8"
}

resource "aws_iam_role" "lambda_exec" {
  name = "example_lambda_role"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

resource "aws_iam_policy_attachment" "lambda_exec_policy" {
  name       = "example_lambda_policy_attachment"
  roles      = [aws_iam_role.lambda_exec.name]
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_api_gateway_rest_api" "example" {
  name = "example_api_gateway"
}

resource "aws_api_gateway_resource" "example" {
  rest_api_id = aws_api_gateway_rest_api.example.id
  parent_id   = aws_api_gateway_rest_api.example.root_resource_id
  path_part   = "example"
}

resource "aws_api_gateway_method" "example" {
  rest_api_id   = aws_api_gateway_rest_api.example.id
  resource_id   = aws_api_gateway_resource.example.id
  http_method   = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_integration" "example" {
  rest_api_id             = aws_api_gateway_rest_api.example.id
  resource_id             = aws_api_gateway_resource.example.id
  http_method             = aws_api_gateway_method.example.http_method
  integration_http_method = "POST"
  type                    = "AWS_PROXY"
  uri                     = aws_lambda_function.example.invoke_arn
}

resource "aws_lambda_permission" "example" {
  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.example.function_name
  principal     = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_rest_api.example.execution_arn}/*/*"
}
```

## Database

```
provider "aws" {
  region = "us-east-1"
}

resource "aws_db_subnet_group" "example" {
  name       = "example-db-subnet-group"
  subnet_ids = ["subnet-xxxxxxxx", "subnet-xxxxxxxx"]
}

resource "aws_db_instance" "example" {
  identifier             = "example-db-instance"
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "postgres"
  engine_version         = "12.5"
  instance_class         = "db.t2.micro"
  name                   = "example-db"
  username               = "admin"
  password               = "password"
  db_subnet_group_name   = aws_db_subnet_group.example.name
  multi_az               = true

  tags = {
    Name = "example-db-instance"
  }
}

resource "aws_route_table" "example" {
  vpc_id = "vpc-xxxxxxxx"

  route {
    cidr_block = "10.0.4.0/24"
    gateway_id = "local"
  }

  route {
    cidr_block = "10.0.5.0/24"
    gateway_id = "local"
  }
}

resource "aws_route_table_association" "example" {
  subnet_id      = "subnet-xxxxxxxx"
  route_table_id = aws_route_table.example.id
}

resource "aws_route_table_association" "example" {
  subnet_id      = "subnet-xxxxxxxx"
  route_table_id = aws_route_table.example.id
}


```


# Fernando Taboada
