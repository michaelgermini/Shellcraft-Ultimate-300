# Chapitre 247 - Infrastructure as Code avec Terraform

## Table des matières
- [Introduction](#introduction)
- [Configuration Terraform de base](#configuration-terraform-de-base)
- [Modules et réutilisabilité](#modules-et-réutilisabilité)
- [State management](#state-management)
- [Conclusion](#conclusion)

## Introduction

Terraform permet de définir et gérer l'infrastructure comme du code, rendant les déploiements reproductibles et versionnés.

## Configuration Terraform de base

**Configuration de base** :
```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "Web Server"
  }
}

# variables.tf
variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "ami_id" {
  description = "AMI ID"
  type        = string
}

variable "instance_type" {
  description = "Instance type"
  default     = "t3.micro"
}

# outputs.tf
output "instance_ip" {
  value = aws_instance.web.public_ip
}
```

## Modules et réutilisabilité

**Module réutilisable** :
```hcl
# modules/web-server/main.tf
variable "instance_type" {
  type = string
}

variable "ami_id" {
  type = string
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
}

output "instance_id" {
  value = aws_instance.web.id
}

# Utilisation du module
module "web_server" {
  source = "./modules/web-server"
  
  instance_type = "t3.medium"
  ami_id        = "ami-12345678"
}
```

## State management

**Gestion du state** :
```bash
#!/bin/bash
# Gestion du state Terraform

# Initialiser avec backend S3
init_with_backend() {
    terraform init \
        -backend-config="bucket=my-terraform-state" \
        -backend-config="key=terraform.tfstate" \
        -backend-config="region=us-east-1"
}

# Importer une ressource existante
import_resource() {
    local resource_type="$1"
    local resource_id="$2"
    
    terraform import "$resource_type" "$resource_id"
}
```

## Conclusion

Terraform transforme la gestion d'infrastructure en processus versionné et reproductible.

