---
id: terraform-aws-infrastructure
alias: Terraform AWS Infrastructure
type: kit
is_base: false
version: 1
tags:
  - foundation
  - infrastructure
  - terraform
description: Complete Terraform patterns for provisioning AWS infrastructure including VPC, RDS, S3, Lambda, and more with best practices for production deployments
---

# Terraform AWS Infrastructure Kit

A comprehensive kit for provisioning AWS infrastructure using Terraform. This kit provides production-ready patterns for common AWS resources with security, scalability, and cost optimization in mind.

## End State

After applying this kit, the application will have:

**Terraform Configuration:**
- Modular Terraform structure with separate modules for different resource types
- State management configured (S3 backend with DynamoDB locking)
- Variable definitions for environment-specific configurations
- Output values for resource references

**AWS Resources:**
- VPC with public/private subnets across multiple availability zones
- Security groups with least-privilege access rules
- RDS database instance with automated backups and multi-AZ support
- S3 buckets with versioning, encryption, and lifecycle policies
- IAM roles and policies for service access
- CloudWatch log groups and alarms
- Application Load Balancer with SSL termination
- Auto Scaling groups for EC2 instances
- Lambda functions with appropriate IAM roles

**Infrastructure Patterns:**
- Environment separation (dev, staging, prod)
- Tagging strategy for cost allocation
- Network isolation and security
- Backup and disaster recovery patterns
- Monitoring and alerting setup

## Implementation Principles

- **Use modules**: Organize resources into reusable modules (VPC, RDS, S3, etc.)
- **State management**: Always use remote state (S3 backend) with DynamoDB locking for team collaboration
- **Environment variables**: Use Terraform workspaces or separate variable files for different environments
- **Tag everything**: Implement consistent tagging strategy for cost tracking and resource management
- **Least privilege**: IAM roles and security groups should follow least-privilege principle
- **Idempotency**: Terraform configurations should be idempotent and safe to run multiple times
- **Version pinning**: Pin provider versions to ensure consistent deployments
- **Output values**: Expose important resource attributes as outputs for other modules/stacks
- **Variable validation**: Use variable validation to catch configuration errors early
- **Cost optimization**: Use appropriate instance types, enable auto-scaling, implement lifecycle policies

## Pattern 1: Basic Terraform Structure

Organize Terraform code into modules and environments:

```
terraform/
├── main.tf                 # Provider configuration
├── variables.tf            # Variable definitions
├── outputs.tf              # Output values
├── terraform.tfvars        # Variable values (gitignored)
├── backend.tf              # Backend configuration
├── modules/
│   ├── vpc/
│   ├── rds/
│   ├── s3/
│   └── lambda/
└── environments/
    ├── dev/
    ├── staging/
    └── prod/
```

**main.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "{{TERRAFORM_STATE_BUCKET}}"
    key            = "{{PROJECT_NAME}}/{{ENVIRONMENT}}/terraform.tfstate"
    region         = "{{AWS_REGION}}"
    encrypt        = true
    dynamodb_table = "{{TERRAFORM_LOCK_TABLE}}"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

## Pattern 2: VPC Module

Create a VPC with public and private subnets:

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
    Type = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-subnet-${count.index + 1}"
    Type = "private"
  }
}

# NAT Gateway (one per AZ for high availability)
resource "aws_eip" "nat" {
  count = length(var.availability_zones)

  domain = "vpc"
  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.project_name}-nat-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.project_name}-private-rt-${count.index + 1}"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

## Pattern 3: RDS Database Module

Provision RDS with high availability:

```hcl
# modules/rds/main.tf
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-db-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "${var.project_name}-db-subnet-group"
  }
}

resource "aws_security_group" "rds" {
  name        = "${var.project_name}-rds-sg"
  description = "Security group for RDS database"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [var.app_security_group_id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-rds-sg"
  }
}

resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-db"

  engine         = var.engine
  engine_version = var.engine_version
  instance_class = var.instance_class

  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted      = true

  db_name  = var.database_name
  username = var.database_username
  password = var.database_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = var.backup_retention_period
  backup_window          = var.backup_window
  maintenance_window     = var.maintenance_window

  multi_az               = var.multi_az
  publicly_accessible    = false
  deletion_protection    = var.deletion_protection
  skip_final_snapshot    = var.skip_final_snapshot

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  performance_insights_enabled = true
  performance_insights_retention_period = 7

  tags = {
    Name = "${var.project_name}-db"
  }
}
```

## Pattern 4: S3 Bucket Module

S3 bucket with security and lifecycle policies:

```hcl
# modules/s3/main.tf
resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name

  tags = {
    Name = var.bucket_name
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Public Access Block
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle Policy
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }
  }

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "delete-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 365
    }
  }
}

# CORS Configuration (if needed)
resource "aws_s3_bucket_cors_configuration" "main" {
  count = var.enable_cors ? 1 : 0

  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST", "DELETE"]
    allowed_origins = var.cors_allowed_origins
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}
```

## Pattern 5: Lambda Function Module

Serverless Lambda function with IAM role:

```hcl
# modules/lambda/main.tf
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = var.source_file
  output_path = "${path.module}/lambda_function.zip"
}

resource "aws_iam_role" "lambda" {
  name = "${var.function_name}-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy" "lambda" {
  name = "${var.function_name}-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      var.additional_policy_statements...
    ]
  })
}

resource "aws_lambda_function" "main" {
  filename         = data.archive_file.lambda_zip.output_path
  function_name    = var.function_name
  role            = aws_iam_role.lambda.arn
  handler         = var.handler
  runtime         = var.runtime
  timeout         = var.timeout
  memory_size     = var.memory_size

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = var.environment_variables
  }

  tags = {
    Name = var.function_name
  }
}

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.log_retention_days
}
```

## Pattern 6: Application Load Balancer

ALB with SSL termination:

```hcl
# modules/alb/main.tf
resource "aws_lb" "main" {
  name               = "${var.project_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  enable_deletion_protection = var.enable_deletion_protection
  enable_http2              = true
  enable_cross_zone_load_balancing = true

  tags = {
    Name = "${var.project_name}-alb"
  }
}

resource "aws_lb_target_group" "main" {
  name     = "${var.project_name}-tg"
  port     = var.target_port
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = var.health_check_path
    protocol            = "HTTP"
    matcher             = "200"
  }

  stickiness {
    enabled = true
    type    = "lb_cookie"
  }

  tags = {
    Name = "${var.project_name}-tg"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

## Pattern 7: Auto Scaling Group

EC2 Auto Scaling with launch template:

```hcl
# modules/autoscaling/main.tf
resource "aws_launch_template" "main" {
  name_prefix   = "${var.project_name}-"
  image_id      = var.ami_id
  instance_type = var.instance_type

  vpc_security_group_ids = [var.security_group_id]

  user_data = base64encode(var.user_data)

  iam_instance_profile {
    name = aws_iam_instance_profile.main.name
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "${var.project_name}-instance"
    }
  }
}

resource "aws_autoscaling_group" "main" {
  name                = "${var.project_name}-asg"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = var.target_group_arns
  health_check_type   = "ELB"
  health_check_grace_period = 300

  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity

  launch_template {
    id      = aws_launch_template.main.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "${var.project_name}-asg"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.project_name}-scale-up"
  autoscaling_group_name = aws_autoscaling_group.main.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = 1
  cooldown               = 300
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "${var.project_name}-scale-down"
  autoscaling_group_name = aws_autoscaling_group.main.name
  adjustment_type        = "ChangeInCapacity"
  scaling_adjustment     = -1
  cooldown               = 300
}
```

## Pattern 8: CloudWatch Alarms

Monitoring and alerting:

```hcl
# modules/monitoring/main.tf
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.project_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period               = "300"
  statistic            = "Average"
  threshold            = "80"
  alarm_description    = "This metric monitors ec2 cpu utilization"
  alarm_actions        = [var.sns_topic_arn]

  dimensions = {
    AutoScalingGroupName = var.autoscaling_group_name
  }
}

resource "aws_cloudwatch_metric_alarm" "low_cpu" {
  alarm_name          = "${var.project_name}-low-cpu"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period               = "300"
  statistic            = "Average"
  threshold            = "20"
  alarm_description    = "This metric monitors ec2 cpu utilization"
  alarm_actions        = [var.sns_topic_arn]

  dimensions = {
    AutoScalingGroupName = var.autoscaling_group_name
  }
}
```

## Best Practices

1. **State Management**: Always use S3 backend with DynamoDB locking for team collaboration
2. **Modularity**: Break infrastructure into reusable modules
3. **Environment Separation**: Use workspaces or separate directories for dev/staging/prod
4. **Tagging**: Implement consistent tagging for cost allocation and resource management
5. **Security**: Use least-privilege IAM policies, encrypt data at rest and in transit
6. **High Availability**: Deploy across multiple AZs, use multi-AZ for RDS
7. **Backups**: Enable automated backups with appropriate retention periods
8. **Monitoring**: Set up CloudWatch alarms for critical metrics
9. **Cost Optimization**: Use appropriate instance types, enable auto-scaling, implement lifecycle policies
10. **Version Control**: Store Terraform code in version control, review changes before applying

## Common Pitfalls

- **State file conflicts**: Always use remote state with locking
- **Hardcoded values**: Use variables and tfvars files
- **Missing dependencies**: Use `depends_on` for explicit resource ordering
- **Cost surprises**: Review resource sizes and enable cost alerts
- **Security misconfigurations**: Review security groups and IAM policies carefully
- **State drift**: Use `terraform plan` regularly to detect configuration drift
- **Provider version conflicts**: Pin provider versions in `required_providers`

## Verification Criteria

After generation, verify:
- ✓ Terraform state is stored in S3 with DynamoDB locking
- ✓ VPC is created with public and private subnets across multiple AZs
- ✓ Security groups follow least-privilege principle
- ✓ RDS instance has automated backups and encryption enabled
- ✓ S3 buckets have versioning, encryption, and lifecycle policies
- ✓ IAM roles have minimal required permissions
- ✓ CloudWatch alarms are configured for critical metrics
- ✓ All resources are properly tagged
- ✓ `terraform plan` shows no unexpected changes
- ✓ Infrastructure can be destroyed and recreated cleanly
