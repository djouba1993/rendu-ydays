# Rendu Ydays Second Semestre 

## Contexte
En vue de la préparation de ma certification AWS Solution Architecte Associate, j’ai travaillé sur ce projet de déployer d’infrastructure sur le cloud aws pour monter en compètence sur les services aws. De ce fait je suis parti sur du terraform pour pouvoir automatiser mon déploiement d’infrastructure. 
Pour ce faire j'ai utilisé des roles terraform.

### config.tf
Dans ce fichier j'ai mis toute la partie configuration le provider sur le quel je veux déployer mon infrastructure à savoir aws, j'ai défini la région et un backend s3 pour stocker mon fichier tfstate à distance et j'ai précisé la version de terraform à utiliser.
```
provider "aws" {
  region = "eu-west-1"
}

terraform {
  backend "s3" {
    bucket  = "ynov-pole-devops"
    key     = "ynov/ydays/devops.tfstate"
    region  = "eu-west-1"
  }

  required_version = "0.13.5"
}
```

### main.tf
La partie rassemble toute la partie configuration concernant la ressource aws_autoscaling_group qui est attaché à  mon lauch_configuration et à mon load balancer.
j'ai attaché mon lauch configuration à un block_device pour faire de la persistance de données en cas de perte de l'instance.
Pour les subnets j'ai déclaré les id au niveau des variables. Ceux sont des subnet par défault que j'ai recupéré dans le vpc default de la region eu-west-1
```
resource "aws_autoscaling_group" "pole_devops_asg" {
  lifecycle {
    create_before_destroy = true
  }

  desired_capacity          = 2
  health_check_grace_period = 300
  health_check_type         = "EC2"
  launch_configuration      = aws_launch_configuration.pole_devops.id
  max_size                  = var.nodes_number
  min_size                  = var.nodes_number
  min_elb_capacity          = var.nodes_number
  name                      = aws_launch_configuration.pole_devops.name
  vpc_zone_identifier       = var.subnet_list
  force_delete              = "false"
  wait_for_capacity_timeout = "10m"
  load_balancers            = [ aws_elb.pole_devops_elb.id ]
  
}

resource "aws_elb" "pole_devops_elb" {
  name                        = "ynov_elb"
  subnets                     = var.subnet_list
  cross_zone_load_balancing   = true
  idle_timeout                = 60
  connection_draining         = true
  connection_draining_timeout = 300
  internal                    = true

  listener {
    instance_port      = 61616
    instance_protocol  = "tcp"
    lb_port            = 61616
    lb_protocol        = "tcp"
    ssl_certificate_id = ""
  }

  health_check {
    healthy_threshold   = 5
    unhealthy_threshold = 2
    interval            = 30
    target              = "TCP:61616"
    timeout             = 5
  }

  tags = local.default_tags
}

resource "aws_launch_configuration" "pole_devops" {
  lifecycle {
    create_before_destroy = true
  }

  name_prefix          = "ynov-pole-devops"
  image_id             = var.lc_image_id
  instance_type        = var.lc_instance_type
  key_name             = var.lc_key_name
  enable_monitoring    = true
  ebs_optimized        = false

  root_block_device {
    volume_type           = "gp2"
    volume_size           = var.volume_size
    delete_on_termination = true
  }
}
```
### variables.tf
Ici on définisse nos variables en indiquant un nom, un type et une valeur qui sont appelées dans les ressources terraform que j'ai utilisées ci-dessus.
```
variable "env" {
  type    = string
  default = "preprod"
}

variable "project_name" {
  type    = string
  default = "ynov"
}

variable "subnet_list" {
  type    = list(string)
  default = ["subnet-bb1e30e1", "subnet-2a638e53"]
}

variable "lc_instance_type" {
  type    = string
  default = "t2.micro"
}

variable "lc_key_name" {
  type    = string
  default = "ynov"
}

variable "lc_image_id" {
  type    = string
  default = "ami-08962a4068733a2b6"
}

variable "nodes_number" {
  type    = number
  default = 2
}

variable "volume_size" {
  type    = number
  default = 8
}
```
