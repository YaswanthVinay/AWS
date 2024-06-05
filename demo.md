### Overview

We will build a cloud infrastructure and CI/CD pipeline for a Notification Service built in the Node.js ecosystem using AWS. The infrastructure includes the Notification API and Email Sender services deployed on AWS ECS Fargate, using Terraform for IaC. We will implement a CI/CD pipeline using GitHub Actions.

### Steps

1. **Fork the Repository and Set Up Local Development**
2. **Provision AWS Infrastructure with Terraform**
3. **Set Up Docker and Amazon ECR**
4. **Deploy Services to AWS ECS Fargate**
5. **Set Up CI/CD with GitHub Actions**
6. **Prepare Screencast and Submit Pull Request**

### 1. Fork the Repository and Set Up Local Development

1. Fork the repo [PearlThoughts/DevOps-Assessment](https://github.com/PearlThoughts/DevOps-Assessment) to your GitHub account.

2. Clone the forked repo:
   ```bash
   git clone https://github.com/your-username/DevOps-Assessment.git
   cd DevOps-Assessment
   ```

3. Install dependencies:
   ```bash
   npm install
   ```

4. Test the Notification API locally:
   ```bash
   npx nx serve pt-notification-service
   ```

### 2. Provision AWS Infrastructure with Terraform

1. Install Terraform:
   ```bash
   brew install terraform
   ```

2. Create a `terraform` directory and navigate into it:
   ```bash
   mkdir terraform
   cd terraform
   ```

3. Create Terraform configuration files (`main.tf`, `variables.tf`, `outputs.tf`):

#### `main.tf`
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_ecr_repository" "notification_api" {
  name = "notification-api"
}

resource "aws_ecr_repository" "email_sender" {
  name = "email-sender"
}

resource "aws_ecs_cluster" "main" {
  name = "notification-cluster"
}

resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecs_task_execution_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action    = "sts:AssumeRole",
      Effect    = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })

  managed_policy_arns = [
    "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
  ]
}

resource "aws_ecs_task_definition" "notification_api" {
  family                   = "notification-api"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  container_definitions = jsonencode([{
    name      = "notification-api"
    image     = "${aws_ecr_repository.notification_api.repository_url}:latest"
    essential = true
    portMappings = [{
      containerPort = 3000
      hostPort      = 3000
    }]
  }])
  memory = "512"
  cpu    = "256"
}

resource "aws_ecs_task_definition" "email_sender" {
  family                   = "email-sender"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  execution_role_arn       = aws_iam_role.ecs_task_execution_role.arn
  container_definitions = jsonencode([{
    name      = "email-sender"
    image     = "${aws_ecr_repository.email_sender.repository_url}:latest"
    essential = true
    portMappings = [{
      containerPort = 3000
      hostPort      = 3000
    }]
  }])
  memory = "512"
  cpu    = "256"
}

resource "aws_ecs_service" "notification_api" {
  name            = "notification-api-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.notification_api.arn
  desired_count   = 1
  launch_type     = "FARGATE"
  network_configuration {
    subnets         = ["subnet-xxxxxxxx"]
    assign_public_ip = true
  }
}

resource "aws_ecs_service" "email_sender" {
  name            = "email-sender-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.email_sender.arn
  desired_count   = 1
  launch_type     = "FARGATE"
  network_configuration {
    subnets         = ["subnet-xxxxxxxx"]
    assign_public_ip = true
  }
}
```

#### `variables.tf`
```hcl
variable "aws_region" {
  default = "us-east-1"
}
```

#### `outputs.tf`
```hcl
output "ecr_repository_url_notification_api" {
  value = aws_ecr_repository.notification_api.repository_url
}

output "ecr_repository_url_email_sender" {
  value = aws_ecr_repository.email_sender.repository_url
}
```

4. Initialize and apply Terraform configuration:
   ```bash
   terraform init
   terraform apply
   ```

### 3. Set Up Docker and Amazon ECR

1. **Dockerize the Services:**
   Create Dockerfiles for both services.

#### `Dockerfile` for Notification API
```dockerfile
FROM node:14
WORKDIR /app
COPY . .
RUN npm install
CMD ["npx", "nx", "serve", "pt-notification-service"]
```

#### `Dockerfile` for Email Sender
```dockerfile
FROM node:14
WORKDIR /app
COPY . .
RUN npm install
CMD ["npx", "nx", "serve", "pt-email-sender"]
```

2. **Build and Push Docker Images:**
   ```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

   docker build -t notification-api .
   docker tag notification-api:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/notification-api:latest
   docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/notification-api:latest

   docker build -t email-sender .
   docker tag email-sender:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/email-sender:latest
   docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/email-sender:latest
   ```

### 4. Deploy Services to AWS ECS Fargate

This step is covered in the Terraform `main.tf` file where the ECS services are defined and deployed. Ensure that the subnets specified in the `network_configuration` section exist and are accessible.

### 5. Set Up CI/CD with GitHub Actions

1. **Create GitHub Actions Workflow:**
   Create a `.github/workflows/ci-cd.yml` file:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '14'

      - name: Install dependencies
        run: npm install

      - name: Build Docker images
        run: |
          docker build -t notification-api .
          docker tag notification-api:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/notification-api:latest

          docker build -t email-sender .
          docker tag email-sender:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/email-sender:latest

      - name: Login to Amazon ECR
        run: echo "${{ secrets.AWS_ACCESS_KEY_ID }}" | docker login --username AWS --password-stdin ${{ secrets.AWS_SECRET_ACCESS_KEY }}

      - name: Push Docker images to ECR
        run: |
          docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/notification-api:latest
          docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/email-sender:latest

      - name: Deploy to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          cluster: notification-cluster
          service: notification-api-service
          task-definition: arn:aws:ecs:us-east-1:<account-id>:task-definition/notification-api
          container-name: notification-api
          container-image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/notification-api:latest

      - name: Deploy Email Sender to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          cluster: notification-cluster
          service: email-sender-service
          task-definition: arn:aws:ecs:us-east-1:<account-id>:task-definition/email-sender
          container-name: email-sender
          container-image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/email-sender:latest
```

2. **Configure Secrets:**
   - Go to the GitHub repository settings.
   - Add the following secrets:
     - `AWS_ACCESS_KEY_ID`
     - `AWS_SECRET_ACCESS_KEY`

### 6. Prepare Screencast and Submit Pull Request

1. **Record Screencast:**
   Use a

 tool like OBS Studio to record the following:
   - Forking the repository.
   - Setting up and running the application locally.
   - Provisioning infrastructure using Terraform.
   - Building and pushing Docker images.
   - Deploying services on AWS ECS Fargate.
   - Setting up and running GitHub Actions CI/CD pipeline.

2. **Submit Pull Request:**
   - Commit your changes and push them to your forked repository.
   - Open a pull request to the original repository.
   - Include a link to the screencast in the pull request description.

### Final Steps

Once everything is complete, provide the pull request link and the screencast link:

- **Pull Request Link:** [Insert PR link here]
- **Screencast Link:** [Insert Screencast link here]

Feel free to ask if you have any questions or need further assistance!
