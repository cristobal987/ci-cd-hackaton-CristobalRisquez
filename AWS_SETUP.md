# Configuración de AWS para CI/CD

Este archivo contiene las instrucciones para configurar los recursos de AWS necesarios para el pipeline de CI/CD.

## Recursos de AWS requeridos

### 1. Amazon ECR (Elastic Container Registry)
```bash
# Crear repositorio ECR
aws ecr create-repository --repository-name ci-cd-hackathon-app --region us-east-1
```

### 2. Amazon ECS (Elastic Container Service)

#### Crear Cluster
```bash
aws ecs create-cluster --cluster-name ci-cd-hackathon-cluster --capacity-providers FARGATE
```

#### Crear Log Group
```bash
aws logs create-log-group --log-group-name /ecs/ci-cd-hackathon-task --region us-east-1
```

#### Crear IAM Roles

##### Execution Role
```bash
# Crear trust policy para execution role
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Crear execution role
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust-policy.json

# Adjuntar policy predefinida
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

##### Task Role (opcional, para acceso a otros servicios de AWS)
```bash
aws iam create-role --role-name ecsTaskRole --assume-role-policy-document file://trust-policy.json
```

### 3. Application Load Balancer (opcional pero recomendado)

#### Crear VPC y subnets (si no existen)
```bash
# Obtener VPC por defecto
aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text

# Obtener subnets públicas
aws ec2 describe-subnets --filters "Name=vpc-id,Values=YOUR_VPC_ID" --query "Subnets[?MapPublicIpOnLaunch==\`true\`].SubnetId" --output text
```

#### Crear Security Group
```bash
# Crear security group para ALB
aws ec2 create-security-group --group-name ci-cd-hackathon-alb-sg --description "Security group for CI/CD Hackathon ALB" --vpc-id YOUR_VPC_ID

# Permitir tráfico HTTP
aws ec2 authorize-security-group-ingress --group-id YOUR_ALB_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0

# Crear security group para ECS tasks
aws ec2 create-security-group --group-name ci-cd-hackathon-ecs-sg --description "Security group for CI/CD Hackathon ECS tasks" --vpc-id YOUR_VPC_ID

# Permitir tráfico desde ALB
aws ec2 authorize-security-group-ingress --group-id YOUR_ECS_SG_ID --protocol tcp --port 3000 --source-group YOUR_ALB_SG_ID
```

#### Crear Application Load Balancer
```bash
aws elbv2 create-load-balancer --name ci-cd-hackathon-alb --subnets YOUR_SUBNET_1 YOUR_SUBNET_2 --security-groups YOUR_ALB_SG_ID
```

#### Crear Target Group
```bash
aws elbv2 create-target-group --name ci-cd-hackathon-tg --protocol HTTP --port 3000 --vpc-id YOUR_VPC_ID --target-type ip --health-check-path /
```

#### Crear Listener
```bash
aws elbv2 create-listener --load-balancer-arn YOUR_ALB_ARN --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=YOUR_TG_ARN
```

### 4. Registrar Task Definition
```bash
# Actualizar task-definition.json con los ARNs correctos, luego:
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

### 5. Crear ECS Service
```bash
aws ecs create-service \
  --cluster ci-cd-hackathon-cluster \
  --service-name ci-cd-hackathon-service \
  --task-definition ci-cd-hackathon-task:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[YOUR_SUBNET_1,YOUR_SUBNET_2],securityGroups=[YOUR_ECS_SG_ID],assignPublicIp=ENABLED}" \
  --load-balancers targetGroupArn=YOUR_TG_ARN,containerName=ci-cd-hackathon-container,containerPort=3000
```

## Secrets de GitHub

Configura estos secrets en tu repositorio de GitHub (Settings > Secrets and variables > Actions):

- `AWS_ACCESS_KEY_ID`: Tu AWS Access Key ID
- `AWS_SECRET_ACCESS_KEY`: Tu AWS Secret Access Key

## Variables a actualizar

En los archivos creados, reemplaza las siguientes variables con tus valores reales:

- `YOUR_ACCOUNT_ID`: Tu AWS Account ID
- `YOUR_VPC_ID`: El ID de tu VPC
- `YOUR_SUBNET_1`, `YOUR_SUBNET_2`: IDs de tus subnets
- `YOUR_ALB_SG_ID`: ID del security group del ALB
- `YOUR_ECS_SG_ID`: ID del security group de ECS
- `YOUR_ALB_ARN`: ARN del Application Load Balancer
- `YOUR_TG_ARN`: ARN del Target Group

## Alternativa: Elastic Beanstalk

Si prefieres usar Elastic Beanstalk (más simple), descomenta la sección correspondiente en el archivo deploy.yml y crea una aplicación de Beanstalk:

```bash
# Crear aplicación
aws elasticbeanstalk create-application --application-name ci-cd-hackathon-app

# Crear environment
aws elasticbeanstalk create-environment \
  --application-name ci-cd-hackathon-app \
  --environment-name ci-cd-hackathon-env \
  --solution-stack-name "64bit Amazon Linux 2 v5.8.4 running Node.js 18"
```

## Comandos útiles

### Para probar localmente con Docker:
```bash
# Construir imagen
docker build -t ci-cd-hackathon-app .

# Ejecutar contenedor
docker run -p 3000:3000 ci-cd-hackathon-app

# Probar health check
docker exec CONTAINER_ID node healthcheck.js
```

### Para debugging:
```bash
# Ver logs de ECS service
aws logs get-log-events --log-group-name /ecs/ci-cd-hackathon-task --log-stream-name ecs/ci-cd-hackathon-container/TASK_ID

# Ver status del service
aws ecs describe-services --cluster ci-cd-hackathon-cluster --services ci-cd-hackathon-service
```
