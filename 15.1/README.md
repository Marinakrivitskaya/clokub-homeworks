# Домашнее задание к занятию "15.1. Организация сети"

Настроить Production like сеть в рамках одной зоны с помощью terraform. Модуль VPC умеет автоматически делать все что есть в этом задании. Но мы воспользуемся более низкоуровневыми абстракциями, чтобы понять, как оно устроено внутри.

1. Создать VPC.

- Используя vpc-модуль terraform, создать пустую VPC с подсетью 172.31.0.0/16.
- Выбрать регион и зону.

2. Публичная сеть.

- Создать в vpc subnet с названием public, сетью 172.31.32.0/19 и Internet gateway.
- Добавить RouteTable, направляющий весь исходящий трафик в Internet gateway.
- Создать в этой приватной сети виртуалку с публичным IP и подключиться к ней, убедиться что есть доступ к интернету.

3. Приватная сеть.

- Создать в vpc subnet с названием private, сетью 172.31.96.0/19.
- Добавить NAT gateway в public subnet.
- Добавить Route, направляющий весь исходящий трафик private сети в NAT.

4. VPN.

- Настроить VPN, соединить его с сетью private.
- Создать себе учетную запись и подключиться через нее.
- Создать виртуалку в приватной сети.
- Подключиться к ней по SSH по приватному IP и убедиться, что с виртуалки есть выход в интернет.

Документация по AWS-ресурсам:

- [Getting started with Client VPN](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/cvpn-getting-started.html)

Модули terraform

- [VPC](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc)
- [Subnet](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/subnet)
- [Internet Gateway](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/internet_gateway)

Решение.
1. Создать VPC.
```
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 3.28”
    }
  }
}
```
```
# Configure the AWS Provider 

provider "aws" {
  region = "eu-south-1”
}
```
```
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = “my-vpc“
  cidr = "172.31.0.0/16"

  azs = ["eu-south-1a", "eu-south-1b", "eu-south-1c"]

  tags = {
    Terraform = "true"
    Environment = “dev”
  }
}
```
2. Публичная сеть.
```
 resource "aws_subnet" "public" {
    vpc_id       = module.vpc.vpc_id
    cidr_block = "172.31.32.0/19"

  tags = {
    Name = "public"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = module.vpc.vpc_id

  tags = {
    Name = “gw”
  }
}

resource "aws_route_table" “route” {
  vpc_id = module.vpc.vpc_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
 tags = {
    Name = “route”
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.route.id
}
```
<img width="491" alt="Screenshot 2021-08-24 at 15 38 24" src="https://user-images.githubusercontent.com/67638098/130626834-4b8f9726-0ca9-4ce3-8d38-8e4cbfbb0863.png">


3. Приватная сеть.
```
resource "aws_subnet" "private" {
  vpc_id     = module.vpc.vpc_id
  cidr_block = "172.31.96.0/19"

  tags = {
    Name = "private"
  }
}

resource "aws_eip" "ip_for_nat" {
  vpc      = true
}

resource "aws_nat_gateway"  "private_subnet” {
  allocation_id = aws_eip.ip_for_nat.id
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "gw NAT"
  }
  depends_on = [aws_internet_gateway.gw]
}


resource "aws_route_table" "private_route" {
  vpc_id = module.vpc.vpc_id

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.private_subnet.id
  }
  tags = {
    Name = "private route" 
  }
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_route_table.private_route.id
}

```
<img width="483" alt="Screenshot 2021-08-24 at 16 45 25" src="https://user-images.githubusercontent.com/67638098/130638121-5ceae4c5-8de8-4cb7-a202-c2c3e237f968.png">

4. VPN.
```
resource "aws_ec2_client_vpn_endpoint" "my_vpn_endpoint" {
  description            = "terraform-clientvpn-endpoint"
  server_certificate_arn = "arn:aws:acm:eu-south-1:475618787347:certificate/0e256650-c9d6-4b16-9ab0-d38bd9b45bfc"
  client_cidr_block      = "10.0.0.0/16"

  authentication_options {
    type                       = "certificate-authentication"
    root_certificate_chain_arn = "arn:aws:acm:eu-south-1:475618787347:certificate/18b0cb8e-7544-4bb4-b1e5-95a714b32130"
  }

  connection_log_options {
    enabled               = false
  }
}

resource "aws_ec2_client_vpn_network_association" "vpn_to_private__association" {
  client_vpn_endpoint_id = aws_ec2_client_vpn_endpoint.my_vpn_endpoint.id
  subnet_id              = aws_subnet.private.id
}

resource "aws_ec2_client_vpn_authorization_rule" "default_vpn_authorization_rule" {
  client_vpn_endpoint_id = aws_ec2_client_vpn_endpoint.my_vpn_endpoint.id
  target_network_cidr    = aws_subnet.private.cidr_block
  authorize_all_groups   = true
}
```
