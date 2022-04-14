---
title: Computação em Nuvem
author: Tutorial - Terraform - 

---

# Infraestrutura como código utilizando Terraform


Objetivos:

1. Entender os conceitos básicos sobre uma plataforma de gerenciamento e provisionamento de infraestrutura.
2. Introduzir conceitos básicos sobre AWS.


### Documentações das ferramentas de desenvolvimento utilizadas:

* [Terraform](https://www.terraform.io/docs/index.html)
* [AWS](https://docs.aws.amazon.com/index.html)
* [Docker](https://docs.docker.com/)
* [VScode](https://code.visualstudio.com/docs)


<!-- GETTING STARTED -->
### Pré requisitos


Como pré requisito instale o Docker em sua máquina seguindo as orientações na doc oficial, de acordo com seu Sistema Operacional.

No meu caso, instalei utilizando o comando curl:
* Docker no Linux Ubuntu 20.04
```sh
curl -SsfL https://get.docker.com | sh -
```

###  Material

Você precisará das chaves de acesso a AWS do seu usuário
  * Access Key ID
  * Secret Access Key 


## Iniciando

Vamos utilizar o Docker e o Terraform, que é apenas um binário que já está instalado na imagem que utilizaremos.

1. Crie uma pasta para iniciar o projeto.

```sh
mkdir -p IaC/terraform
cd IaC/
```


2. Subir um Container a partir da imagem com binário do Terraform.

```sh
docker run -it -v $PWD:/app -w /app --name $(basename $PWD) --entrypoint "" tiagodemay/terraform_aws:version1  sh
```

* Com estes comando você já deve estar dentro do container na pasta "/app"
* Caso você saia do container e ele pare, terá que inicia-lo novamente para acessa-lo:

Este comando serve para pegar o ID do container
```sh
docker container ls -a
```
Estes dois te colocam no terminal do container novamente

```sh
docker container start <CONTAINER ID>
docker container attach <CONTAINER ID>
```

Lembre-se que a pasta que você iniciou o projeto está sendo compartilhada como o container,

Pelo bem ou pelo mal, apagar ou alterar qualquer arquivo na pasta no seu "host" irá alterar os arquivos dentro do container e vice e versa.


## Usage
  
Para fazer a utilização deste projeto você deve seguir as seguintes etapas:

1. Criar um bucket s3 na AWS, aqui eu estou criando um backend em um bucket S3 para salvar o estado da infraestrutura, no arquivo terraform/main.tf
 
2.  Para os comandos abaixos você deve estar dentro do cointainer e passar as credenciais da sua conta na AWS

```sh
/app           # cd terraform/
/app/terraform # export AWS_ACCESS_KEY_ID="AKXXXXXXXXXXXXXXXXXX"
/app/terraform # export AWS_SECRET_ACCESS_KEY="5DXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
```

3. Vamos testar com um script simples, criando um arquivo nesta pasta chamado main.tf (app/terraform/main.tf)

```sh
# Configure the AWS Provider
provider "aws" {
  version = "~> 2.8"
  region  = "us-east-2"
}
# Criando bucket para salvar o Estado da Infraestrutura
terraform {
  backend "s3" {
    bucket  = "coloque-aqui-o-nome-do-seu-bucket"
    key     = "terraform.tsstate"
    region  = "us-east-2"
    encrypt = true
  }
}

```

4. 

* Agora dentro da pasta terraform daremos o comando para iniciar o terraform e configurar nosso backend s3
```sh
/app/terraform # terraform init
```

 
* Agora criaremos um plano no terraform, para analisar as mudanças que ele irá fazer a sua infraestrutura da AWS.
* O nome do seu arquivo de saída pode ser qualquer um escolhido por você, para que fique didático eu escolhi o nome "plano" 

```sh
/app/terraform # terraform plan -out plano
```  

* Como não há nada a ser criado ele avisa que não há mudanças a fazer.
           No changes. Infrastructure is up-to-date.

5. Vamos testar com um script simples, para subir 3 instancias basicas, adicione mais um arquivo .tf (instace.tf)
```sh
# Criação de 3 instancias nomeadas como "maq"
resource "aws_instance" "maq" {
  count = 3
  ami = "ami-0d8f6eb4f641ef691"
  instance_type = "t2.micro"
  key_name = "osm"
  tags = {
    Name = "maq${count.index}"
  }
}
```

* Na saída do seu terminal estão todas as modificações que serão feitas na sua estrutura na região escolhida da AWS.
* Um detalhe muito legal do terraform é que aqui ele demonstra tudo que será adicionado, alterado ou destruido na sua insfraestrutura com um toque para facilitar a visualização:
    1. to add     --- trás um simbolo de adição (+) em verde para demostrar o que será criado.
    2. to change  --- trás um simbolo de til (~) em amarelo para demostrar mudança de estado do recurso criado.
    3. to destroy --- trás um simbolo de subtração (-) em vermelho para demonstrar o que será excluido da infraestrutura.
* Ainda neste passo é possivel "debugar" erros de programação e analisar respostas de recursos a serem criados, antes de subir a estrutura para a AWS, basta alterar os arquivos e recursos que desejar, salvar os documentos alterados e rodar o comando novamente, até ficar satisfeito com o PLANO de criação.
* Se for a primeira vez que roda o programa ele deve mostrar:
 Plan: 3 to add, 0 to change, 0 to destroy.    

6. Então depois de analisar se é isso mesmo o que desejamos criar é só dar o comando: 

```sh
/app/terraform # terraform apply "plano"
``` 

* Na saída do seu terminal provavelmente apareceu um erro, deixamos este erro acontecer para ficar claro que o debug do comando Plan é correspondente a solução de infraestrutura e a comparação lógica do arquivo de estado, ele não confirma, por exemplo se você não possui alguma premissa para criar o recurso, neste caso a keypair não foi previamente criada.
* Aqui fica a opção: criar na própria AWS ou criar um recurso novo, enviando a sua chave.
* Para exemplificar, ante de continuar crie a Keypair na própria AWS (com o nome "osm"). 


7. Repetir passos de plan e apply. 

```sh
# terraform plan -out plano
# terraform apply "plano"
```


### E simples assim sua infraestrutura será criada.
### Verifique que as instancias foram criadas na AWS
### Verifique também que um arquivo do tipo tsstate foi criado dentro do seu bucket.


8. Vamos modificar o arquivo instance.tf e aplicar o plano e o apply novamente:

* Altere o tipo de instancia para t2.nano
* e modifique o count para 5
* Veja os resultados das modificações.


## Limpeza do ambiente

9. Agora Vamos destruir a infrainstrutura inteira.

```sh
/app/terraform # terraform destroy
```
#### Ele calcula o que será destruido e te avisa, depois de a confirmação para deletar tudo e pronto. 

```sh
aws_instance.maq[1]: Refreshing state... [id=i-0bcc9558ecae360ed]
aws_instance.maq[0]: Refreshing state... [id=i-03ee446fda0276798]
aws_instance.maq[2]: Refreshing state... [id=i-017d0e0e6cd7dacbb]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions
```

## Antes de continuar confirme se a infra criada anteriormente foi destruida completamente.

## Vamos agora trabalhar com alguns recursos diferentes.

* security_group
* providers
* s3_bucket
* output
* variaveis

### Iremos criar os arquivos: outputs.tf, security-group.tf, vars.tf e modificar os arquivos já criados, main.tf e instance.tf

1. Começaremos modificando o main.tf, pois queremos criar maquinas em 2 regioes diferentes.

O arquivo deve ficar assim:

```sh
# Configure the AWS Provider

provider "aws" {
  version = "~> 2.0"
  region = var.aws_region["N_Virginia"]
}

provider "aws" {
  alias = "us-east-2"
  version = "~> 2.0"
  region  = var.aws_region["Ohio"]
}


# Criando bucket para salvar o Estado da Infraestrutura
terraform {
  backend "s3" {
    bucket  = "coloque-aqui-o-nome-do-seu-bucket"
    key     = "terraform.tsstate"
    region  = "us-east-2"
    encrypt = true
  }
}

```
2. Modificaremos o arquivo instance.tf também, note que no segundo resource colocamos o provider 2 para uso.

```sh
# Criação de 2 instancias em cada região.


resource "aws_instance" "Nv" {
  count = 2
  ami = var.image["us-east-1"]
  instance_type = var.instance_type
  key_name = var.key_name
  tags = {
    Name = "maqNv${count.index}"
  }
  vpc_security_group_ids = ["${aws_security_group.acesso-ssh-Regiao1.id}"]
}

resource "aws_instance" "Oh" {
  provider = aws.us-east-2
  count = 2
  ami = var.image["us-eats-2"]
  instance_type = var.instance_type
  key_name = var.key_name
  tags = {
    Name = "maqOh${count.index}"
  }
  vpc_security_group_ids = ["${aws_security_group.acesso-ssh-Regiao2.id}"]
}
```

3. Note que utilizamos **variaveis** nos arquivos acima, para facilitar a edição e compreenção.
 Vamos criar o arquivos de variaveis:

```sh
#Imagens de cada região - no site https://cloud-images.ubuntu.com/locator/ec2/

variable "image" {
  description   = "The AWS AMI to create things in."
  default = {
    "us-east-1" = "ami-0885b1f6bd170450c"
    "us-eats-2" = "ami-0d8f6eb4f641ef691"
  }
}

#Definição de Regions utilizadas
variable "aws_region" {
  description   = "The AWS region to create things in."
  default       = {
    "N_Virginia"   = "us-east-1"
    "Ohio"         = "us-east-2"
  } 
}

#Key_Pair utilizado para acessar as máquinas via SSH
variable "key_name" {
  description   = "Name of AWS key pair"
  default       = "osm"
  }

#Tipo das Instancias do projeto  
variable "instance_type" {
  description   = "AWS instance type"
  default       = "t2.micro"
}

#IP liberado para acessar a máquina SSH - coloque o IP publico do seu computador.
variable "ip-ssh" {
    description = "Ip de Saida da Rede Host para acesso Remoto"
    default     = "186.232.61.4/32"
  
}

```
### Você conhece o IP (default) que está alocado acima, que IP é este?

```sh





```

4. Iremos criar o Security-Group abaixo, para definir regras de acesso, note que no segundo resource temos o provider 2 setado.


```sh
#### Arquivo de Configuração do Firewall da AWS - Security Group - Para definir Portas de acesso ####

#Security Group com nome de SSH, liberando somente a 22 para Inbound e todas para Outbound
resource "aws_security_group" "acesso-ssh-Regiao1" {
  name        = "acesso-ssh1"
  description = "Allow SSH inbound traffic"
  ingress {
    # SSH (change to whatever ports you need)
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    # Please restrict your ingress to only necessary IPs and ports.
    # Opening to 0.0.0.0/0 can lead to security vulnerabilities.
    cidr_blocks = ["${var.ip-ssh}"]  # my ip at the present time
  }
    egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
  }
  tags = {
    Name = "ssh1"
  }
}

resource "aws_security_group" "acesso-ssh-Regiao2" {
  name        = "acesso-ssh2"
  provider = aws.us-east-2
  description = "Allow SSH inbound traffic"
  ingress {
    # SSH (change to whatever ports you need)
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    # Please restrict your ingress to only necessary IPs and ports.
    # Opening to 0.0.0.0/0 can lead to security vulnerabilities.
    cidr_blocks = ["${var.ip-ssh}"]  # my ip at the present time
  }
    egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
  }
  tags = {
    Name = "ssh2"
  }
}
```

5. Por ultimo iremos criar o arquivo outputs.tf, para trabalhar algumas saídas possiveis no terminal, podemos direcionar estas saídas para um arquivo de texto se necessário.

```sh

output "instance_Ips-PublicosRegiao1" {
  value = ["${aws_instance.Nv.*.public_ip}"]
}

output "instance_idsRegiao1" {
  value = ["${aws_instance.Nv.*.id}"]
}


output "instance_Ips-PublicoRegiao2" {
  value = ["${aws_instance.Oh.*.public_ip}"]
}

output "instance_idsRegiao2" {
  value = ["${aws_instance.Oh.*.id}"]
}

```

6. 
```sh
# terraform plan -out plano
# terraform apply "plano"

```

7. Para finalizar vamos criar um Bucket no S3.

Crie um arquivo .tf

```sh

# No "Name" coloque o seu nome, pois não pode existir 2 buckets com nome iguais !!!!!!!
resource "aws_s3_bucket" "s3" {
  bucket = "tiago-demay-s3"
  acl    = "private"
  tags = {
    Name = "tiago-demay-s3"
  }
}

```



### Confira sua infra.

# Destrua tudo

# F i m ...