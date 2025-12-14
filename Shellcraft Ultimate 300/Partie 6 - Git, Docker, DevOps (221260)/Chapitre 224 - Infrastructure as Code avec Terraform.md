# Chapitre 224 - Infrastructure as Code avec Terraform

> "L'infrastructure as code transforme les datacenters en programmes exécutables, permettant aux équipes de versionner, tester, et déployer leur infrastructure avec la même rigueur que leur code applicatif." - Mitchell Hashimoto, créateur de Terraform

## Introduction : L'infrastructure programmable

L'Infrastructure as Code (IaC) représente une révolution dans la gestion des ressources informatiques. En permettant de définir, versionner et déployer l'infrastructure comme du code, Terraform et ses équivalents transforment des semaines de configuration manuelle en déploiements reproductibles et automatisés.

Dans ce chapitre, nous explorerons les principes fondamentaux de l'IaC, la syntaxe HCL de Terraform, les patterns de déploiement, et l'intégration dans les workflows DevOps modernes.

## Section 1 : Concepts fondamentaux de l'IaC

### 1.1 Pourquoi l'Infrastructure as Code ?

**Problèmes de l'infrastructure traditionnelle :**
- Configuration manuelle sujette aux erreurs
- Différences entre environnements (dev/prod)
- Documentation souvent obsolète
- Déploiement non reproductible
- Gestion des changements complexe

**Avantages de l'IaC :**
- **Versionning** : L'infrastructure versionnée comme le code
- **Reproductibilité** : Déploiements identiques à chaque fois
- **Testabilité** : Validation automatique des configurations
- **Collaboration** : Revue et partage des changements d'infrastructure
- **Automatisation** : Intégration dans les pipelines CI/CD

### 1.2 Le cycle de vie de l'IaC

```bash
# Cycle de vie complet d'un projet IaC
echo "=== CYCLE DE VIE IAC ==="
echo "1. PLAN: Analyse des changements à apporter"
echo "2. APPLY: Application des changements"
echo "3. TEST: Validation du déploiement"
echo "4. DESTROY: Nettoyage des ressources (optionnel)"
echo ""

# Exemple avec Terraform
echo "Commandes Terraform:"
echo "terraform init     # Initialisation"
echo "terraform plan     # Planification"
echo "terraform apply    # Application"
echo "terraform destroy  # Destruction"
```

## Section 2 : Premiers pas avec Terraform

### 2.1 Installation et configuration

```bash
# Installation de Terraform
# Ubuntu/Debian
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# CentOS/RHEL
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform

# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Windows (Chocolatey)
choco install terraform

# Vérification
terraform version

# Auto-completion (optionnel)
terraform -install-autocomplete
```

### 2.2 Premier projet Terraform

```hcl
# main.tf - Configuration Terraform de base
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Configuration du provider AWS
provider "aws" {
  region = "us-east-1"
  
  # Authentification (variables d'environnement ou fichier credentials AWS)
  # AWS_ACCESS_KEY_ID et AWS_SECRET_ACCESS_KEY
}

# Création d'un VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "main-vpc"
    Environment = "dev"
  }
}

# Création d'un subnet
resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  
  tags = {
    Name = "main-subnet"
  }
}

# Security group
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = aws_vpc.main.id

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

  tags = {
    Name = "web-sg"
  }
}

# Instance EC2
resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316" # Amazon Linux 2
  instance_type = "t2.micro"
  
  subnet_id                   = aws_subnet.main.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true
  
  tags = {
    Name = "web-server"
  }
}

# Output des valeurs importantes
output "instance_public_ip" {
  description = "IP publique de l'instance EC2"
  value       = aws_instance.web.public_ip
}

output "vpc_id" {
  description = "ID du VPC créé"
  value       = aws_vpc.main.id
}
```

```bash
# Workflow Terraform de base
# 1. Initialisation
terraform init

# 2. Validation de la syntaxe
terraform validate

# 3. Planification des changements
terraform plan

# 4. Application des changements
terraform apply

# 5. Inspection de l'état
terraform show

# 6. Nettoyage (optionnel)
terraform destroy
```

## Section 3 : Langage HCL et expressions avancées

### 3.1 Variables et types de données

```hcl
# variables.tf - Définition des variables
variable "environment" {
  description = "Environnement de déploiement"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "L'environnement doit être dev, staging ou prod."
  }
}

variable "instance_count" {
  description = "Nombre d'instances à créer"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Le nombre d'instances doit être entre 1 et 10."
  }
}

variable "instance_types" {
  description = "Types d'instances disponibles"
  type        = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
}

variable "allowed_cidrs" {
  description = "CIDR blocks autorisés"
  type        = list(string)
  default     = ["10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16"]
}

variable "tags" {
  description = "Tags à appliquer aux ressources"
  type        = map(string)
  default = {
    Project     = "MyProject"
    Owner       = "DevOps Team"
    ManagedBy   = "Terraform"
  }
}

# Utilisation des variables dans main.tf
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_types[var.environment]
  
  tags = merge(var.tags, {
    Name        = "web-server-${count.index + 1}"
    Environment = var.environment
  })
}

# Variables depuis la ligne de commande
# terraform apply -var="environment=prod" -var="instance_count=3"

# Variables depuis un fichier
# terraform apply -var-file="prod.tfvars"
```

```hcl
# prod.tfvars - Valeurs pour l'environnement de production
environment   = "prod"
instance_count = 3

# Variables depuis l'environnement
# export TF_VAR_environment="staging"
# export TF_VAR_aws_region="us-west-2"
```

### 3.2 Data sources et références

```hcl
# Data sources pour récupérer des informations existantes
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["main-vpc"]
  }
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.existing.id]
  }
  
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

# Utilisation des data sources
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  subnet_id     = data.aws_subnets.private.ids[0]
  
  tags = {
    Name         = "web-server"
    Environment  = var.environment
    Owner        = data.aws_caller_identity.current.account_id
    Region       = data.aws_region.current.name
  }
}

# Références entre ressources
resource "aws_eip" "web" {
  instance = aws_instance.web.id
  vpc      = true
  
  tags = {
    Name = "web-eip"
  }
}

# Références implicites
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = aws_vpc.main.id  # Référence à une autre ressource
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = var.allowed_cidrs
  }
}
```

### 3.3 Métaprogrammation et dynamic blocks

```hcl
# Dynamic blocks pour répétitions conditionnelles
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = aws_vpc.main.id

  # Règles d'entrée dynamiques
  dynamic "ingress" {
    for_each = var.ingress_rules
    
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }

  # Règles de sortie dynamiques
  dynamic "egress" {
    for_each = var.egress_rules
    
    content {
      from_port   = egress.value.from_port
      to_port     = egress.value.to_port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
    }
  }

  tags = var.tags
}

# Variables pour les règles dynamiques
variable "ingress_rules" {
  description = "Règles d'entrée pour le security group"
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  ]
}

# Count et for_each pour ressources multiples
resource "aws_instance" "web" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  subnet_id = element(data.aws_subnets.private.ids, count.index)
  
  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# For_each pour maps
resource "aws_s3_bucket" "buckets" {
  for_each = var.buckets
  
  bucket = each.value.name
  acl    = each.value.acl
  
  versioning {
    enabled = each.value.versioning
  }
  
  tags = merge(var.tags, each.value.tags)
}

variable "buckets" {
  description = "Configuration des buckets S3"
  type = map(object({
    name       = string
    acl        = string
    versioning = bool
    tags       = map(string)
  }))
  default = {
    "app-logs" = {
      name       = "myapp-logs"
      acl        = "private"
      versioning = true
      tags = {
        Purpose = "logging"
      }
    }
    "backups" = {
      name       = "myapp-backups"
      acl        = "private"
      versioning = true
      tags = {
        Purpose = "backup"
      }
    }
  }
}
```

## Section 4 : Modules et réutilisabilité

### 4.1 Création de modules

```hcl
# modules/vpc/main.tf
variable "cidr_block" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "name" {
  description = "Name of the VPC"
  type        = string
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  
  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  
  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.this.id
}

output "igw_id" {
  description = "ID of the created Internet Gateway"
  value       = aws_internet_gateway.this.id
}
```

```hcl
# modules/vpc/variables.tf
variable "azs" {
  description = "Availability zones"
  type        = list(string)
  default     = []
}

variable "private_subnets" {
  description = "Private subnet CIDR blocks"
  type        = list(string)
  default     = []
}

variable "public_subnets" {
  description = "Public subnet CIDR blocks"
  type        = list(string)
  default     = []
}

# modules/vpc/subnets.tf
resource "aws_subnet" "private" {
  count             = length(var.private_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index % length(var.azs)]
  
  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Type = "private"
  })
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index % length(var.azs)]
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Type = "public"
  })
}

# modules/vpc/outputs.tf
output "private_subnet_ids" {
  description = "IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}
```

### 4.2 Utilisation de modules

```hcl
# main.tf - Utilisation du module VPC
module "vpc" {
  source = "./modules/vpc"
  
  name       = "myapp"
  cidr_block = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  tags = {
    Environment = "dev"
    Project     = "myapp"
  }
}

# Utilisation des outputs du module
resource "aws_security_group" "web" {
  name_prefix = "web-"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  
  subnet_id                   = module.vpc.public_subnet_ids[0]
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true
  
  tags = {
    Name = "web-server"
  }
}
```

### 4.3 Registres de modules

```hcl
# Utilisation d'un module depuis le Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}

# Utilisation d'un module depuis Git
module "web_server" {
  source = "git::https://github.com/terraform-aws-modules/terraform-aws-ec2-instance.git?ref=v3.0.0"
  
  name                   = "web-server"
  instance_type          = "t2.micro"
  key_name               = "my-key"
  monitoring             = true
  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = module.vpc.public_subnets[0]
  
  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```

## Section 5 : Gestion d'état et collaboration

### 5.1 Backend pour l'état Terraform

```hcl
# Configuration du backend S3
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Backend local (défaut)
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

# Backend Azure
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state"
    storage_account_name = "terraformstate"
    container_name       = "tfstate"
    key                  = "dev.terraform.tfstate"
  }
}

# Backend GCS (Google Cloud Storage)
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "dev"
  }
}

# Remote backend Terraform Cloud
terraform {
  backend "remote" {
    hostname     = "app.terraform.io"
    organization = "my-org"
    
    workspaces {
      name = "dev-workspace"
    }
  }
}
```

### 5.2 Workspaces et environnements

```bash
# Gestion des workspaces
terraform workspace list          # Lister les workspaces
terraform workspace select dev    # Sélectionner un workspace
terraform workspace new staging   # Créer un nouveau workspace
terraform workspace delete old    # Supprimer un workspace

# Configuration par workspace
# main.tf
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = lookup(var.instance_types, terraform.workspace, "t2.micro")
  
  tags = {
    Name        = "web-server-${terraform.workspace}"
    Environment = terraform.workspace
  }
}

# terraform.workspace contient le nom du workspace actuel
# Permet de différencier les configurations par environnement
```

```hcl
# Variables par environnement
variable "instance_types" {
  description = "Types d'instance par environnement"
  type        = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
}

# Configuration conditionnelle
resource "aws_instance" "web" {
  count = terraform.workspace == "prod" ? 2 : 1
  
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_types[terraform.workspace]
  
  tags = {
    Name        = "web-server-${terraform.workspace}-${count.index}"
    Environment = terraform.workspace
  }
}
```

### 5.3 Locking et collaboration

```bash
# Verrouillage automatique avec DynamoDB
# Création de la table DynamoDB pour les locks
aws dynamodb create-table \
    --table-name terraform-locks \
    --attribute-definitions AttributeName=LockID,AttributeType=S \
    --key-schema AttributeName=LockID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

# Terraform détecte automatiquement le locking avec DynamoDB
# lorsqu'un backend S3 est configuré avec dynamodb_table

# Forcer le déverrouillage (en cas de problème)
terraform force-unlock LOCK_ID

# Lister les locks actuels
aws dynamodb scan --table-name terraform-locks
```

## Section 6 : Tests et validation

### 6.1 Tests unitaires avec Terratest

```go
// test/terraform_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformBasic(t *testing.T) {
    t.Parallel()
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    
    terraform.InitAndApply(t, terraformOptions)
    
    // Vérifications
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
    
    instancePublicIP := terraform.Output(t, terraformOptions, "instance_public_ip")
    assert.NotEmpty(t, instancePublicIP)
}

func TestTerraformSecurityGroup(t *testing.T) {
    t.Parallel()
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/security-group",
        Vars: map[string]interface{}{
            "vpc_id": "vpc-12345",
            "ingress_rules": []map[string]interface{}{
                {
                    "from_port":   80,
                    "to_port":     80,
                    "protocol":    "tcp",
                    "cidr_blocks": []string{"0.0.0.0/0"},
                },
            },
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    
    terraform.InitAndApply(t, terraformOptions)
    
    // Vérifier que le security group a été créé avec les bonnes règles
    securityGroupID := terraform.Output(t, terraformOptions, "security_group_id")
    assert.NotEmpty(t, securityGroupID)
}
```

```bash
# Exécution des tests Terratest
cd test
go mod init terraform-tests
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/stretchr/testify/assert

go test -v
```

### 6.2 Validation et linting

```bash
# Validation de la syntaxe
terraform validate

# Formatage automatique
terraform fmt -recursive

# Vérification du formatage
terraform fmt -check -recursive

# Installation de tflint pour linting avancé
# https://github.com/terraform-linters/tflint

# Création d'un fichier de configuration tflint
cat > .tflint.hcl << 'EOF'
config {
  module = true
}

plugin "aws" {
    enabled = true
    version = "0.17.0"
    source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_required_version" {
    enabled = true
}

rule "terraform_required_providers" {
    enabled = true
}
EOF

# Exécution de tflint
tflint --config .tflint.hcl

# Intégration dans les pipelines CI/CD
# .github/workflows/terraform.yml
name: Terraform

on: [push, pull_request]

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      
    - name: Terraform Format
      run: terraform fmt -check -recursive
      
    - name: Terraform Validate
      run: terraform validate
      
    - name: TFLint
      uses: terraform-linters/setup-tflint@v2
      with:
        tflint_version: latest
        
    - name: Run TFLint
      run: tflint --config .tflint.hcl
```

### 6.3 Tests d'intégration avec InSpec

```ruby
# test/integration/default/controls/terraform.rb
control 'terraform-vpc' do
  impact 1.0
  title 'Test VPC Terraform'
  desc 'Vérifie que le VPC Terraform a été créé correctement'
  
  vpc_id = input('vpc_id')
  
  describe aws_vpc(vpc_id) do
    it { should exist }
    its('cidr_block') { should cmp '10.0.0.0/16' }
    its('state') { should eq 'available' }
  end
end

control 'terraform-instance' do
  impact 1.0
  title 'Test Instance EC2 Terraform'
  desc 'Vérifie que l\'instance EC2 a été créée correctement'
  
  instance_id = input('instance_id')
  
  describe aws_ec2_instance(instance_id) do
    it { should exist }
    it { should be_running }
    its('instance_type') { should eq 't2.micro' }
  end
end
```

```bash
# Exécution des tests InSpec
inspec exec test/integration/default \
    --input vpc_id="$(terraform output -raw vpc_id)" \
    --input instance_id="$(terraform output -raw instance_id)"
```

## Section 7 : Patterns avancés et best practices

### 7.1 Pattern Landing Zone

```hcl
# Structure pour une landing zone multi-comptes
# accounts.tf
variable "accounts" {
  description = "Configuration des comptes AWS"
  type = map(object({
    email     = string
    role_name = string
  }))
  default = {
    "dev" = {
      email     = "dev@example.com"
      role_name = "DevAdmin"
    }
    "prod" = {
      email     = "prod@example.com" 
      role_name = "ProdAdmin"
    }
  }
}

# Création des comptes
resource "aws_organizations_account" "accounts" {
  for_each = var.accounts
  
  name  = each.key
  email = each.value.email
  
  role_name = each.value.role_name
}

# network.tf - Configuration réseau de base
module "vpc" {
  for_each = aws_organizations_account.accounts
  
  source = "./modules/vpc"
  providers = {
    aws = aws.account[each.key]
  }
  
  name = "${each.key}-vpc"
  cidr = "10.${index(keys(var.accounts), each.key)}.0.0/16"
}

# security.tf - Politiques de sécurité communes
resource "aws_organizations_policy" "security" {
  name = "SecurityBaseline"
  
  content = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Action = [
          "s3:*",
          "ec2:*"
        ]
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:RequestedRegion" = ["us-east-1", "eu-west-1"]
          }
        }
      }
    ]
  })
}
```

### 7.2 Gestion des secrets

```hcl
# Intégration avec AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = "prod/database/credentials"
}

locals {
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_credentials.secret_string)
}

resource "aws_db_instance" "database" {
  username = local.db_credentials.username
  password = local.db_credentials.password
  
  # Autres configurations...
}

# Intégration avec HashiCorp Vault
data "vault_generic_secret" "db_credentials" {
  path = "secret/database/prod"
}

resource "aws_db_instance" "database" {
  username = data.vault_generic_secret.db_credentials.data["username"]
  password = data.vault_generic_secret.db_credentials.data["password"]
  
  # Autres configurations...
}
```

### 7.3 Automatisation des déploiements

```bash
#!/bin/bash
# Script d'automatisation complet pour Terraform

set -euo pipefail

# Configuration
ENVIRONMENT="${1:-dev}"
WORKSPACE="$ENVIRONMENT"
TF_DIR="infrastructure/$ENVIRONMENT"

# Fonction de logging
log() {
    echo "[$(date +%s)] $*" >&2
}

# Fonction de vérification des prérequis
check_prerequisites() {
    log "Vérification des prérequis..."
    
    if ! command -v terraform >/dev/null 2>&1; then
        log "ERREUR: Terraform n'est pas installé"
        exit 1
    fi
    
    if ! command -v aws >/dev/null 2>&1; then
        log "ERREUR: AWS CLI n'est pas installé"
        exit 1
    fi
    
    # Vérification des credentials AWS
    if ! aws sts get-caller-identity >/dev/null 2>&1; then
        log "ERREUR: Credentials AWS non configurés"
        exit 1
    fi
    
    log "Prérequis OK"
}

# Fonction de sélection du workspace
select_workspace() {
    local workspace="$1"
    
    cd "$TF_DIR"
    
    if ! terraform workspace select "$workspace" 2>/dev/null; then
        log "Création du workspace $workspace"
        terraform workspace new "$workspace"
    fi
    
    log "Workspace $workspace sélectionné"
}

# Fonction de déploiement
deploy_infrastructure() {
    local action="${1:-plan}"
    
    log "Déploiement infrastructure - Action: $action"
    
    # Validation
    terraform validate
    
    # Formatage
    terraform fmt -check
    
    case "$action" in
        "plan")
            terraform plan -out=tfplan
            ;;
        "apply")
            if [[ -f "tfplan" ]]; then
                terraform apply tfplan
            else
                terraform apply -auto-approve
            fi
            ;;
        "destroy")
            terraform destroy -auto-approve
            ;;
        *)
            log "ERREUR: Action inconnue: $action"
            exit 1
            ;;
    esac
}

# Fonction de tests post-déploiement
run_post_deployment_tests() {
    log "Exécution des tests post-déploiement..."
    
    # Récupération des outputs Terraform
    local vpc_id=$(terraform output -raw vpc_id 2>/dev/null || echo "")
    local instance_ip=$(terraform output -raw instance_public_ip 2>/dev/null || echo "")
    
    if [[ -n "$vpc_id" ]]; then
        log "Test VPC: $vpc_id"
        # Tests AWS...
    fi
    
    if [[ -n "$instance_ip" ]]; then
        log "Test instance: $instance_ip"
        # Tests de connectivité...
    fi
}

# Fonction de rollback
rollback_deployment() {
    local backup_state="$1"
    
    log "Rollback vers l'état: $backup_state"
    
    if [[ -f "$backup_state" ]]; then
        cp "$backup_state" terraform.tfstate
        terraform refresh
        log "Rollback terminé"
    else
        log "ERREUR: État de sauvegarde non trouvé: $backup_state"
        exit 1
    fi
}

# Fonction principale
main() {
    local action="${2:-plan}"
    
    check_prerequisites
    select_workspace "$ENVIRONMENT"
    
    case "$action" in
        "deploy"|"apply")
            # Sauvegarde de l'état actuel
            cp terraform.tfstate "terraform.tfstate.backup.$(date +%Y%m%d_%H%M%S)" 2>/dev/null || true
            
            deploy_infrastructure "apply"
            run_post_deployment_tests
            ;;
        "plan")
            deploy_infrastructure "plan"
            ;;
        "destroy")
            deploy_infrastructure "destroy"
            ;;
        "rollback")
            local backup_file="${3:-terraform.tfstate.backup}"
            rollback_deployment "$backup_file"
            ;;
        *)
            echo "Usage: $0 <environment> [deploy|plan|destroy|rollback [backup_file]]"
            exit 1
            ;;
    esac
    
    log "Opération terminée avec succès"
}

# Exécution
main "$@"
```

## Conclusion : Terraform comme langage d'infrastructure

Terraform transforme l'infrastructure en code exécutable, permettant de définir, versionner et déployer des environnements complexes avec la même rigueur que le code applicatif. Sa syntaxe déclarative et ses capacités de modularisation en font l'outil de référence pour l'Infrastructure as Code moderne.

Dans le prochain chapitre, nous explorerons d'autres outils IaC comme Ansible, CloudFormation, et les patterns de déploiement multi-cloud, complétant notre panorama des pratiques DevOps.

---

**Exercice pratique :** Créez une infrastructure complète avec Terraform qui inclut :
1. Réseau VPC avec sous-réseaux publics/privés
2. Instances EC2 avec load balancer
3. Base de données RDS
4. Buckets S3 avec policies
5. Utilisation de modules pour la réutilisabilité
6. Tests et validation automatiques

**Challenge avancé :** Développez un framework IaC multi-cloud qui :
- Supporte AWS, Azure et GCP simultanément
- Implémente des patterns de déploiement avancés
- Gère les secrets et configurations sensibles
- Fournit des rapports de conformité automatiques
- S'intègre dans des pipelines GitOps

**Réflexion :** Comment Terraform change-t-il fondamentalement l'approche de la gestion d'infrastructure, et quels sont les impacts sur la productivité des équipes d'exploitation et la fiabilité des systèmes ?

