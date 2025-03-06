# Desafio DevOps - VExpenses

### Repositório para entrega do desafio técnico do processo seletivo de estágio em DevOps da VExpenses

### Para o desafio técnico foi enviado um arquico Terraform, uma ferramenta de IaC (Infrastructure as Code), que permite a construção, manutenção e versionamento de infraestura de forma segura e eficiente. Além disso, ele traz vantagens em relação à automação, reprodutibilidade e escalabilidade.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### Essa primeira parte do código define a AWS como provedor da nossa infraestutura, uma vez que o Terraform suporta múltiplos provedores, sendo possível, inclusive utilizar mais de um se necessário.

### Vale destacar que estamos utilizando a região "us-east-1" que é uma das mais barata para criação de recursos na AWS, garantindo assim uma infraestutura mais custo-eficiente.

```hcl
variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}
```

### O código acima está criando variáveis para o nome do projeto e do candidato, que serão usadas futuramente para definir os nomes dos recursos da nossa infraestutura de forma dinâmica

```hcl
resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}
```

### Aqui estamos definindo recursos para gerar e gerenciar um par de chaves SSH, que serão usadas para acessar nossa instância EC2.

### Utilizamos o provedor TLS, com o algoritmo RSA, para gerar uma chave privada de 2048 bits, garantindo assim conexões SSH de forma segura.

```hcl
resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}
```

### Nessa parte, estamos criando uma VPC (Virtual Private Cloud), que é uma rede isolada dentro da AWS, onde, por padrão, permite que os recursos dentro dela se comuniquem uns com os outros.

### Definimos o intervalo de endereços IP da rede (mantendo os primeirso 16 bits fixos), habilitamos a resolução de nomes DNS dentro da VPC e nomeamos dinamicamente a VPC utilizando as variáveis criadas anteriormente

```hcl
resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}
```

### Aqui estamos criando uma subnet, que trata-se de uma divisão lógica dentro de uma VPC, cuja função é segmentar a rede, separando recursos para trazer melhor organização e controle de acesso.

### Estamos associando a subnet com a VPC já criada, definindo a faixa de endereços IPs e sua zona de disponibilidade.

```hcl
resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}
```

### No trecho de código acima, estamos criando um gateway para a nossa VPC, permitindo que os recursos dentro dela tenham acesso à internet

```hcl
resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}
```

### O recurso acima trata-se da criação de uma tabela de rotas dentro da nossa VPC, que permite que o tráfego seja corretamente direcionado.

### Definimos também a rota padrão (0.0.0.0/0), que direciona todo o tráfego para o Internet Gateway e permite que os recursos da VPC acessem a internet.

```hcl
resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}
```

### Aqui estamos associando a subnet que criamos à nossa tabela de rotas, fazendo com que os recursos dentro dela sigam as regras estabelecidas pela tabela de rotas e permitindo também que os recursos da subnet acessem a internet

```hcl
resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  # Regras de entrada
  ingress {
    description      = "Allow SSH from anywhere"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  # Regras de saída
  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}
```

### Aqui estamos criando um Security Group, que funciona como um firewall, controlando o tráfego de entrada e saída para as instâncias da nossa VPC

### Definimos o nome dinamicamente e associamos à VPC. Criamos a regra de conexões SSH na porta 22 vindas de qualquer origem e também permitimos todo o tráfego de saída, ou seja, qualquer instância dentro da VPC pode se comunicar com a internet.

```hcl
data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}
```

### O código acima busca automaticamente a última imagem disponível do Debian 12 (64 bits, HVM), filtrando a busca pelo nome, tipo de virtualização e proprietário oficial, garantindo sempre a versão mais recente, evintando assim, atualizações manuais.

```hcl
resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}
```

### Aqui estamos criando uma instância EC2 com 20GB de armazenamento, endereço IP público e script de inicialização (user_data) que atualiza o sistema.

### Vale ressaltar que garantimos também a exclusão automática do disco ao deletar a instância

### Utilizando o tamanho t2.micro conseguimos utilizar a camada gratuita da AWS e com tudo citado temos uma VM Debian pronta para ser utilizada.

```hcl
output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

### Basicamente, nesses dois últimos trechos do código, estamos exibindo a chave privada SSH usada para acessar nossa instância e o endereço IP público da EC2 criada.