# Tienda de Perritos - DevOps en AWS EKS

Guía paso a paso para desplegar una aplicación web CRUD (tienda de alimentos para perros) en Amazon EKS usando EC2 como máquina de build, Docker, CloudFormation, Kubernetes y GitHub Actions.

---

## Índice

1. [Arquitectura del proyecto](#1-arquitectura-del-proyecto)
2. [Estructura del repositorio](#2-estructura-del-repositorio)
3. [Crear roles IAM en AWS](#3-crear-roles-iam-en-aws)
4. [Lanzar una instancia EC2 (máquina de build)](#4-lanzar-una-instancia-ec2-máquina-de-build)
5. [Configurar EC2: AWS CLI, Docker, kubectl, git](#5-configurar-ec2-aws-cli-docker-kubectl-git)
6. [Clonar el repositorio en EC2](#6-clonar-el-repositorio-en-ec2)
7. [Provisionar infraestructura con CloudFormation](#7-provisionar-infraestructura-con-cloudformation)
8. [Conectar kubectl al cluster EKS](#8-conectar-kubectl-al-cluster-eks)
9. [Build y push de imágenes Docker a ECR](#9-build-y-push-de-imágenes-docker-a-ecr)
10. [Desplegar aplicación en EKS](#10-desplegar-aplicación-en-eks)
11. [Configurar GitHub Actions (CI/CD automático)](#11-configurar-github-actions-cicd-automático)
12. [Verificar el despliegue](#12-verificar-el-despliegue)
13. [Autoescalado con HPA](#13-autoescalado-con-hpa)

---

## 1. Arquitectura del proyecto

```
                     GitHub
                   ┌─────────┐
                   │  Push a │
                   │  deploy │
                   └────┬────┘
                        │
               GitHub Actions
                   ┌────┴────┐
                   │  Build  │
                   │ & Push  │
                   └────┬────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
        ECR-Frontend  ECR-Backend  ECR-DB
              │         │         │
              └─────────┼─────────┘
                        ▼
                    EKS Cluster
                  ┌────────────┐
                  │  Service   │
                  │ LoadBalancer│
                  └──────┬─────┘
                         ▼
                   Internet 🌍
```

**Componentes:**
- **Frontend:** Nginx sirve HTML+JS (puerto 80)
- **Backend:** API REST Node.js + Express (puerto 3001)
- **Base de datos:** MySQL 8 (puerto 3306)
- **Infraestructura:** VPC, ECR, EKS, NAT Gateway — todo definido en CloudFormation
- **CI/CD:** GitHub Actions build + push + deploy automático

---

## 2. Estructura del repositorio

```
ev3-devops/
├── .github/workflows/deploy.yml       # Pipeline CI/CD (GitHub Actions)
├── 1_CloudFormation/
│   └── duoc-devops-act3.yaml           # Plantilla CloudFormation (VPC, ECR, EKS)
├── 2_CloudShell/
│   └── deploy-tienda-perritos/         # Aplicación + Kubernetes
│       ├── deploy.sh                   # Script de deploy manual
│       ├── frontend/                   # Frontend en Nginx (Dockerfile, index.html, app.js)
│       ├── backend/                    # Backend Node.js (Dockerfile, server.js, package.json)
│       ├── db/                         # MySQL 8 (Dockerfile, init.sql)
│       └── k8s/                        # Manifiestos Kubernetes
│           ├── namespace.yaml
│           ├── mysql-secret.yaml
│           ├── mysql-pvc.yaml
│           ├── mysql-deployment.yaml
│           ├── mysql-service.yaml
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── backend-hpa.yaml
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           └── frontend-hpa.yaml
└── README.md
```

---

## 3. Crear roles IAM en AWS

Antes de ejecutar CloudFormation necesitas dos roles IAM.

> **Nota para entornos AWS Academy (Learner Lab):** Los roles IAM ya se crean automáticamente en cada lab con nombres largos y aleatorios (por ejemplo, `c209059a5310053l15329600t1w535152-LabEksClusterRole-IOAyOzsyWCfL`). En ese caso, **no debes crear los roles manualmente**: ve directamente a **IAM > Roles**, identifica los roles con "EksCluster" y "EksNodeRole" en su nombre, y copia esos nombres para usarlos en el paso 7. Asegúrate de no incluir espacios al inicio o al final del nombre al pegarlo.

### 3.1. EKS Cluster Role

Ve a **IAM > Roles > Create role**:
- **Trusted entity type:** AWS Service
- **Use case:** EKS
- **Permissions:** `AmazonEKSClusterPolicy`
- **Name:** `EksClusterRole` (o el que prefieras)

### 3.2. EKS Node Group Role

- **Trusted entity type:** AWS Service
- **Use case:** EC2
- **Permissions:**
  - `AmazonEKSWorkerNodePolicy`
  - `AmazonEKS_CNI_Policy`
  - `AmazonEC2ContainerRegistryReadOnly`
- **Name:** `EksNodeRole` (o el que prefieras)

Guarda ambos nombres, los necesitarás en el paso 7.

---

## 4. Lanzar una instancia EC2 (máquina de build)

Esta EC2 la usarás para ejecutar los comandos de build y deploy.

1. Ve a **EC2 > Launch instance**
2. Configura:
   - **Name:** `devops-builder`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** `t3.medium` (mínimo 2 vCPU, 4 GB RAM)
   - **Key pair:** Crea o selecciona una existente
   - **Network settings:**
     - VPC por defecto (luego usaremos la VPC creada por CloudFormation)
     - **Security group:** permite SSH (22) desde tu IP
     - **IAM role:** (opcional por ahora, pero puedes crear un rol con permisos `AdministratorAccess` para simplificar)
   - **Storage:** 20 GB gp3

### 4.1. (Opcional) Crear rol IAM para la EC2

Ve a **IAM > Roles > Create role**:
- **Trusted entity type:** AWS Service
- **Use case:** EC2
- **Permissions:** `AdministratorAccess` (para este lab; en producción limita permisos)
- **Name:** `EC2-Admin-Role`

Luego asigna ese rol a tu EC2:  
**EC2 > Instancias > seleccionas la instancia > Actions > Security > Modify IAM role**

Si asignas el rol a la EC2, **no necesitas** configurar `aws configure` manualmente (paso 5.2) porque las credenciales se obtienen automáticamente del rol.

---

## 5. Configurar EC2: AWS CLI, Docker, kubectl, git

Conéctate por SSH a la EC2 y ejecuta:

### 5.1. Actualizar sistema e instalar paquetes

```bash
sudo yum update -y
sudo yum install -y git docker
```

### 5.2. Configurar AWS CLI

Si **no** asignaste un rol IAM a la EC2:

```bash
aws configure
# AWS Access Key ID: tu access key
# AWS Secret Access Key: tu secret key
# Default region: us-east-1
# Default output: json
```

Si **asignaste un rol IAM**, saltas este paso (las credenciales se obtienen automáticamente).

Verifica que funciona:

```bash
aws sts get-caller-identity
```

### 5.3. Instalar kubectl

```bash
curl -O https://s3.us-east-1.amazonaws.com/amazon-eks/1.35.0/2025-01-01/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

### 5.4. Iniciar Docker y agregar usuario al grupo

```bash
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user
```

Cierra sesión y vuelve a conectarte para que el grupo surta efecto:

```bash
exit
# vuelve a conectarte con ssh
```

Verifica:

```bash
docker ps
```

---

## 6. Clonar el repositorio en EC2

```bash
git clone <URL_DE_TU_REPOSITORIO> ev3-devops
cd ev3-devops
```

---

## 7. Provisionar infraestructura con CloudFormation

Desde la EC2, ejecuta:

```bash
aws cloudformation create-stack \
  --stack-name devops-infra \
  --template-body file://1_CloudFormation/duoc-devops-act3.yaml \
  --parameters ParameterKey=EksClusterRoleName,ParameterValue=EksClusterRole \
               ParameterKey=EksNodeRoleName,ParameterValue=EksNodeRole \
  --capabilities CAPABILITY_IAM
```

Reemplaza `EksClusterRole` y `EksNodeRole` con los nombres que creaste (o copiaste desde IAM) en el paso 3.

Espera a que termine (aproximadamente 12-15 minutos):

```bash
aws cloudformation wait stack-create-complete --stack-name devops-infra
```

Puedes verificar el progreso desde la consola de AWS en **CloudFormation > Stacks > devops-infra > Events > Timeline view**. Cuando todos los recursos aparezcan en verde (completados) y el node group esté en estado Activo, el stack está listo.

**Lo que crea esta plantilla:**

| Recurso | Nombre | Detalle |
|---|---|---|
| VPC | `devopsvpc` | 10.0.0.0/16 |
| Subnets públicas | 2 en AZ A y B | 10.0.1.0/24, 10.0.2.0/24 |
| Subnets privadas | 4 (2 por AZ) | 10.0.11-12-21-22.0/24 |
| Internet Gateway | `devops-igw` | Para subnets públicas |
| NAT Gateway | `devops-nat` | Para salida a internet desde privadas |
| ECR repos | `tienda-frontend`, `tienda-backend`, `tienda-db` | Almacén de imágenes Docker |
| EKS cluster | `devopseks` | Kubernetes 1.35 |
| Node group | `devops-nodegroup` | t3.large, 1-3 nodos |
| Addons | vpc-cni, coredns, kube-proxy, pod-identity, CloudWatch | |

---

## 8. Conectar kubectl al cluster EKS

```bash
aws eks update-kubeconfig --region us-east-1 --name devopseks
```

Verifica la conexión:

```bash
kubectl get nodes
```

Deberías ver 1 nodo (o más si el escalado automático ya actuó).

---

## 9. Build y push de imágenes Docker a ECR

### 9.1. Login a ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com
```

### 9.2. Build, tag y push

```bash
cd 2_CloudShell/deploy-tienda-perritos

ECR_URL=$(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com

# Frontend
docker build -t tienda-frontend ./frontend
docker tag tienda-frontend:latest $ECR_URL/tienda-frontend:eks-v1
docker push $ECR_URL/tienda-frontend:eks-v1

# Backend
docker build -t tienda-backend ./backend
docker tag tienda-backend:latest $ECR_URL/tienda-backend:eks-v1
docker push $ECR_URL/tienda-backend:eks-v1

# DB
docker build -t tienda-db ./db
docker tag tienda-db:latest $ECR_URL/tienda-db:eks-v1
docker push $ECR_URL/tienda-db:eks-v1
```

---

## 10. Desplegar aplicación en EKS

### 10.1. Reemplazar placeholder de ECR en manifiestos

Los manifiestos YAML usan `{{ECR_URL}}` como placeholder. Sustitúyelo:

```bash
ECR_URL=$(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com

find ./k8s -type f -name "*.yaml" -exec sed -i "s|{{ECR_URL}}|$ECR_URL|g" {} \;
```

### 10.2. Instalar Metrics Server (necesario para HPA)

El Metrics Server es requerido para que el autoescalado horizontal (HPA) pueda leer el uso de CPU de los pods. Sin este componente, los HPA quedarán en estado `unknown` y no escalarán.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verifica que esté listo antes de continuar:

```bash
kubectl rollout status deployment/metrics-server -n kube-system --timeout=120s
```

### 10.3. Aplicar manifiestos

```bash
# Namespace
kubectl apply -f ./k8s/namespace.yaml

# MySQL
kubectl apply -f ./k8s/mysql-secret.yaml
kubectl apply -f ./k8s/mysql-deployment.yaml
kubectl apply -f ./k8s/mysql-service.yaml

# Esperar que MySQL esté listo antes de iniciar el backend
kubectl rollout status deployment/tienda-db -n tienda --timeout=300s

# Backend
kubectl apply -f ./k8s/backend-deployment.yaml
kubectl apply -f ./k8s/backend-service.yaml
kubectl rollout status deployment/tienda-backend -n tienda --timeout=300s

# Frontend
kubectl apply -f ./k8s/frontend-deployment.yaml
kubectl apply -f ./k8s/frontend-service.yaml
kubectl rollout status deployment/tienda-frontend -n tienda --timeout=300s

# HPA
kubectl apply -f ./k8s/frontend-hpa.yaml
kubectl apply -f ./k8s/backend-hpa.yaml
```

### 10.4. Orden de despliegue

```
1. Namespace        → crea el espacio de nombres "tienda"
2. MySQL Secret     → define la contraseña en base64
3. MySQL Deployment → 1 pod con MySQL 8
4. MySQL Service    → headless service para resolución DNS interna
5. Backend          → 2 pods con la API Node.js
6. Backend Service  → ClusterIP (solo interno)
7. Frontend         → 2 pods con Nginx
8. Frontend Service → LoadBalancer (público, expuesto a internet)
9. HPA              → autoescalado por CPU
```

### 10.5. Obtener la URL pública

```bash
kubectl get svc tienda-frontend -n tienda -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Abre esa URL en el navegador. Puede demorar unos minutos en quedar efectivamente habilitada incluso después de que el script la muestre. Deberías ver la tienda de perritos con productos precargados.

---

## 11. Configurar GitHub Actions (CI/CD automático)

El pipeline se activa automáticamente con cada `push` a la rama `deploy`.

### 11.1. Agregar secrets al repositorio

Ve a **Settings > Secrets and variables > Actions > New repository secret** y agrega:

| Secret | Valor |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access Key de AWS |
| `AWS_SECRET_ACCESS_KEY` | Secret Key correspondiente |
| `AWS_SESSION_TOKEN` | Session Token (requerido al usar credenciales temporales de AWS Academy) |

### 11.2. Flujo del pipeline

El archivo `.github/workflows/deploy.yml` hace lo siguiente:

1. **Checkout** del código
2. **Configure AWS Credentials** — usa los secrets para autenticarse
3. **Login to Amazon ECR** — autentica Docker contra ECR
4. **Build, Tag, Push Frontend** — `docker build ./frontend`, taggea y pushea
5. **Build, Tag, Push Backend** — `docker build ./backend`, taggea y pushea
6. **Build, Tag, Push DB** — `docker build ./db`, taggea y pushea
7. **Update Kubeconfig** — conecta kubectl al cluster `devopseks`
8. **Deploy to EKS** — reemplaza `{{ECR_URL}}` y aplica todos los YAML

> **Nota:** El pipeline aplica los manifiestos de forma secuencial pero sin esperar a que cada componente esté listo. Si el backend falla al conectar con MySQL en el primer intento, Kubernetes lo reiniciará automáticamente hasta que MySQL esté disponible. El Metrics Server debe instalarse manualmente (paso 10.2) antes de que los HPA funcionen correctamente.

### 11.3. Cómo usarlo

```bash
git checkout -b deploy
git push origin deploy
```

Cada push a `deploy` ejecuta el pipeline completo: build, push a ECR y deploy a EKS.

---

## 12. Verificar el despliegue

```bash
# Pods
kubectl get pods -n tienda

# Servicios
kubectl get svc -n tienda

# HPA
kubectl get hpa -n tienda

# Logs de un pod específico
kubectl logs -n tienda deployment/tienda-frontend
kubectl logs -n tienda deployment/tienda-backend

# Health check del backend
curl http://<BACKEND_POD_IP>:3001/api/health
```

**Salida esperada (pods):**

```
NAME                              READY   STATUS    RESTARTS   AGE
tienda-backend-xxxxxxxxx-yyyy     1/1     Running   0          2m
tienda-backend-xxxxxxxxx-zzzz     1/1     Running   0          2m
tienda-db-xxxxxxxxx-wwww          1/1     Running   0          3m
tienda-frontend-xxxxxxxxx-vvvv    1/1     Running   0          1m
tienda-frontend-xxxxxxxxx-uuuu    1/1     Running   0          1m
```

---

## 13. Autoescalado con HPA

El proyecto incluye autoescalado horizontal basado en CPU:

| Deployment | Min réplicas | Max réplicas | CPU target |
|---|---|---|---|
| Frontend | 2 | 6 | 60% |
| Backend | 2 | 10 | 70% |

```bash
kubectl get hpa -n tienda
```

Para probar el escalado, genera carga:

```bash
# Instala hey (herramienta de benchmark)
sudo yum install -y golang
go install github.com/rakyll/hey@latest

# Genera carga contra el LoadBalancer
hey -n 10000 -c 50 http://<LOAD_BALANCER_DNS>/
```

Luego monitorea:

```bash
kubectl get hpa -n tienda -w
```

---

## Resumen de comandos útiles

```bash
# EC2 (build machine)
ssh -i tu-key.pem ec2-user@<IP_EC2>

# Infraestructura
aws cloudformation create-stack \
  --stack-name devops-infra \
  --template-body file://1_CloudFormation/duoc-devops-act3.yaml \
  --parameters ParameterKey=EksClusterRoleName,ParameterValue=EksClusterRole \
               ParameterKey=EksNodeRoleName,ParameterValue=EksNodeRole \
  --capabilities CAPABILITY_IAM
aws cloudformation delete-stack --stack-name devops-infra

# EKS
aws eks update-kubeconfig --region us-east-1 --name devopseks
kubectl get nodes
kubectl get pods -n tienda
kubectl get svc -n tienda
kubectl logs -n tienda deployment/tienda-frontend

# Docker
docker build -t tienda-frontend ./frontend
docker tag tienda-frontend:latest <ECR_URL>/tienda-frontend:eks-v1
docker push <ECR_URL>/tienda-frontend:eks-v1

# GitHub Actions
git checkout -b deploy
git add .
git commit -m "deploy: nueva versión"
git push origin deploy
```
