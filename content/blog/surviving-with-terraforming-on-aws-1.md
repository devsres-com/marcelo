+++
title = "Sobrevivendo com Terraform na AWS - parte 1"
description = "Como começar do zero com Terraform na AWS"
author = "Marcelo Andrade"
date = "2021-03-11"
tags = ["terraform", "aws"]
categories = ["terraform", "aws"]
[[images]]
  src = "img/reusable/valhein-dropoff.png"
  alt = "Minecraft"
  stretch = ""
+++

# Sobrevivendo com Terraform na AWS

Então, um pássaro preto gigante te jogou em um terreno desconhecido e inóspito conhecido como **AWS** e a única coisa que você tem é um terminal Linux e o binário **terraform** da Hashicorp. O que você faz?

(Ou talvez não tenha sido um pássaro preto; talvez tenha sido o seu gerente de TI que te incumbiu de construir tudo que será necessário para o início das atividades na AWS. Mas a ideia do pássaro preto gigante parece mais legal).

## Interlude 
### IAM e um usuário administrativo não-"root"

Na "língua nativa AWS", o usuário **root** é aquele com o e-mail que criou a conta. Em ambientes empresariais complexos, teríamos uma estrutura de [*Organizations*](https://aws.amazon.com/pt/organizations/) com várias contas encadeadas e possivelmente integração com alguma base de usuários locais usando esta ou aquela estratégia de autenticação. Vamos abstrair isso; o usuário **root** é o gerente de TI que criou a conta e cadastrou o cartão de crédito corporativo sem limites (ou não!) da empresa e te entregou **uma** única coisa:

* Um usuário do IAM com seu nome.

É com ele (e com uma senha temporária) que você irá logar a primeira (e última!) vez na Console Web AWS. Afinal, nós vivemos para a tela preta do console.

* Autentique-se na Console usando a página customizada de login, o alias da conta ou simplesmente o ID se nada disso tiver sido feito.
* Busque por **IAM**, e clique no seu usuário.
* Clique em **Credenciais de Segurança** 

Aqui temos **duas** atividades para fazer:

* Crie uma chave de acesso. É um par ID e chave de acesso que você usará para "acessar programaticamente" a AWS usando o cliente de linha de comando (e, consequentemente, o terraform). Você pode baixar o .CSV se não estiver pronto para usá-la ainda.

* Faça o upload de sua chave pública SSH para poder usá-la com o [*AWS Code Commit*](https://aws.amazon.com/pt/codecommit/). Escreva em algum lugar o "*ID da chave do SSH", que é um conjunto de letras em sentido - vamos precisar dela mais na frente.

(**Nota**: você obviamente não precisa usar o CodeCommit se preferir usar o Github, mas né, a título de laboratório, vamos nos ater às coisas AWS).

Com isso, você está pronto para abandonar o excessivamente colorido universo WWW e partir para o que realmente importa.

### Baixando e configurando os programas essenciais

Vamos precisar de dois programas para nos divertir com este trabalho:

* [Cliente AWS](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-install)
* [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli) 

Os dois links acima constam com procedimentos para instalação da última versão, que na data deste post, são aws-cli/2.1.30 e terraform /0.14.7.

### Configurando o aws cli

Para configurar o AWS cli, basta executar:

```
$ aws configure
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: sa-east-1
Default output format [None]: json
```

(Não, eu não esqueci minhas access keys aqui, esse é o [exemplo da página oficial](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html))

Será que está tudo ok? Teste vendo a si mesmo:

```bash
aws iam list-users --query "Users[?UserName=='the-ops-hero']"
[
    {
        "Path": "/",
        "UserName": "the-ops-hero",
        "UserId": "AIDQWERJLATCU123CR7M3UJ",
        "Arn": "arn:aws:iam::446926690069:user/the-ops-hero",
        "CreateDate": "2021-03-11T03:12:47+00:00",
        "PasswordLastUsed": "2021-03-11T03:13:42+00:00"
    }
]
```

(Essa query syntax chama-se [JMESPATH](https://jmespath.org/) e, se quiser usar adequadamente o AWS cli, é bom ir se [acostumando com ela](https://gist.github.com/magnetikonline/6a382a4c4412bbb68e33e137b9a74168)!)

### Configurando o Terraform

É criar o diretório e configurar o provider AWS: 
```bash
$ mkdir terraform-boostrap
$ cd terraform-bootstrap
$ cat << 'EOF' > provider.aws.tf
provider "aws" {

  region = "sa-east-1"

}
EOF

```

# Traçando uma estratégia inicial

Aqui, definimos os pré-requisitos para trabalhar com Terraform com o mínimo de profissionalismo. Quais são?

* Precisamos **versionar** as coisas que criamos;
* Precisamos de um lugar para guardar o nosso '*statefile*' que não seja na nossa máquina local;
* Precisamos de uma estratégia para lidar com *concorrência* de execuções.

Vamos atacar cada um desses pontos progressivamente.

## Requisito 1: Versionando o código Terraform usando CodeCommit

Temos à nossa disposição o serviço [*AWS Code Commit*](https://aws.amazon.com/pt/codecommit/), que é um servidor de repositórios Git mantido pela AWS.

(Precisamos usar ele? Não, você pode usar Github ou qualquer outra coisa se preferir, mas para manter tudo homogêneo, vamos usá-lo!)

Então a primeira coisa a fazer é criar o tal repositório... Usando Terraform!

Vamos começar a confeccionar o nosso arquivo terraform-bootstrap.tf assim:

```bash
# ./terraform-bootstrap.tf
# Qualquer nome que termine com .tf será processado. A maioria das pessoas usa main.tf, mas eu gosto de ser diferente.

# Repositório GIT para servir de base para a Infraestrutura de Terraform:
resource "aws_codecommit_repository" "terraform-bootstrap" {
  repository_name = "terraform-git"
  description     = "Comecei na AWS agora, este repositorio tem tudo que preciso para comecar a rodar terraform"

  tags = {
    Name        = "terraform-dev-sres"
    CreatedBy   = "terraform"
    Environment = "production"
  }

}
```
Não esqueça de customizar as tags de acordo com seu gosto. Na AWS, **tags** são tudo.

Vamos criar nosso primeiro recurso?
```bash
$ terraform plan -out terraform-bootstrap.plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_codecommit_repository.terraform-bootstrap will be created
  + resource "aws_codecommit_repository" "terraform-bootstrap" {
      + arn             = (known after apply)
      + clone_url_http  = (known after apply)
      + clone_url_ssh   = (known after apply)
      + description     = "Comecei na AWS agora, este repositorio tem tudo que preciso para comecar a rodar terraform"
      + id              = (known after apply)
      + repository_id   = (known after apply)
      + repository_name = "terraform-git"
      + tags            = {
          + "CreatedBy"   = "terraform"
          + "Environment" = "production"
          + "Name"        = "terraform-dev-sres"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

This plan was saved to: terraform-bootstrap.plan

To perform exactly these actions, run the following command to apply:
    terraform apply "terraform-bootstrap.plan"
``` 

Não criamos nada ainda, executamos um plan apenas para dar uma olhada. Vamos criar esse troço!

```bash
$ terraform apply terraform-bootstrap.plan
aws_codecommit_repository.terraform-bootstrap: Creating...
aws_codecommit_repository.terraform-bootstrap: Creation complete after 1s [id=terraform-bootstrapx]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate
```

Agora, vamos usar o tal repositório! 

... como dou clone?

Duas opções:

* Pede informações usando a AWS CLI:

```bash
# Se você preferir, pode executar sem o --query para ver toods os campos.
$ aws codecommit get-repository --repository-name=terraform-bootstrap --query repositoryMetadata.cloneUrlSsh --output text
ssh://git-codecommit.sa-east-1.amazonaws.com/v1/repos/terraform-git
```
* Cria um output no Terraform
```bash
# arquivo ./outputs.tf

# Você pode imprimir todas as informações
output "codecommit-terraform-bootstrap" {
        value = aws_codecommit_repository.terraform-bootstrap
}

# Ou apenas filtrar pela url:
output "codecommit-terraform-bootstrap-url-ssh" {
        value = aws_codecommit_repository.terraform-bootstrap.clone_url_ssh
}
```

Para ver a saída do output, será necesário um *refresh*:

```bash
$ terraform refresh
aws_codecommit_repository.terraform-bootstrap: Refreshing state... [id=terraform-bootstrapx]

Outputs:

codecommit-terraform-bootstrap = "ssh://git-codecommit.sa-east-1.amazonaws.com/v1/repos/terraform-git"
```

Agora é só dar o git clone, certo?

```bash
$ REPO=$( aws codecommit get-repository --repository-name=terraform-bootstrap --query repositoryMetadata.cloneUrlSsh --output text )
$ git clone $REPO
Cloning into 'terraform-bootstrap'...
Warning: Permanently added the RSA host key for IP address '52.94.206.167' to the list of known hosts.
marcelo@git-codecommit.sa-east-1.amazonaws.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

Putz, não rolou. Mas eu fiz o upload da minha chave no IAM, certo? Deveria funcionar, certo?

**Não**. 

[RTFM](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html#setting-up-ssh-unixes-keys), em especial a parte 8:

> On your local machine, use a text editor to create a config file in the ~/.ssh directory, and then add the following lines to the file, where the value for User is the SSH key ID you copied earlier:

```bash
Host git-codecommit.*.amazonaws.com
  User APKAEIBAERJR2EXAMPLE
  IdentityFile ~/.ssh/codecommit_rsa
```

Ah, lembra daquele tal "Id da chave SSH", **ele** é o seu usuário no CodeCommit, não seu usuário do IAM, o que faz todo sentido, pensando direitinho.

Configure seu ~/.ssh/config e tente novamente que vai funcionar.

### Interlude - git bureaucracy

Eu normalmente aplico um 'git clone' no repo vazio e mudo o diretório .git de lugar:

```bash
$ git clone $REPO
mv terraform-git/.git ./
git add . -A
git commit -m 'yay funcinou'
git push 
```

Mas eu reconheço que isso é feio, então talvez esses comandos mais verboses agradem mais:

```bash
$ git init 
$ git remote add origin $REPO
$ git pull origin
# ... ao fim de tudo # 
$ git push --set-upstream origin master
```

Não vou me incomodar com git aqui, ok? 

Não esqueça de colocar um [.gitignore](https://raw.githubusercontent.com/github/gitignore/master/Terraform.gitignore) apropriado no seu repositório.

## Requisito 2: Armazenando estado do Terraform usando S3

Se você olhar no diretório corrente, vai encontrar o seguinte:
```bash
$ ls *.tfstate*
terraform.tfstate  terraform.tfstate.backup
``` 

Se sua máquina morrer agora (não estou agourando!), é como se o seu Terraform não tivesse existido. As coisas que ele criou irão persistir, mas esse arquivo **é** o Terraform; ele que mapeia o que ele criou, o que ele controla, quais são os atributos e o que fazer com eles.

Esse arquivo **não** pode ficar em uma máquina local.

Onde colocar? 

Estando na AWs, o melhor lugar para colocar (também chamado de '*backend*') é o [AWS S3](https://www.terraform.io/docs/language/settings/backends/s3.html), o serviço de armazenamento de objetos da AWS.

Como fazer isso?

Bem, primeiro, precisamos de um Bucket S3. E qual a melhor maneira de criar isso? Usando Terraform, é claro!

### Criando Buckets S3 com Terraform
```bash
# bucket.tf. Não precisa ser outro arquivo, mas pode ser se você quiser.

resource "aws_s3_bucket" "terraform-bootstrap" {
  # não se esqueça de colocar o seu nome, nomes de buckets são globais na aws.
  bucket = "terraform-bootstrap-dev-sres"
  acl    = "private"

  versioning {
    enabled = true
  }

  tags = {
    Name        = "terraform-bootstrap"
    Environment = "production"
  }
}
```

Aqui, não esqueça das **Tags** e de habilitar o **versioning**, que vai te proteger se alguem decidir excluir ou sobrescrever seu programa com algum copy/paste mal feito! 

```bash
# vou suprimir as saídas destes comandos porque são altamente irrelevantes.
$ terraform plan -out terraform-bootstrap.pan
...
$ terraform apply terraform-bootstrap.plan
...
```
Ok! Temos um bucket!

```bash
$ aws s3api list-buckets --query Buckets[].Name --output text
terraform-bootstrap-dev-sres
```

### Configurando o bucket S3 como Backend

Como configurar este bucket como backend?

```bash
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-bootstrap-dev-sres"
    key            = "terraform-bootstrap/tfstate"
    region         = "sa-east-1"
  }
}
```

Após modificaçõçes de **backend**, será necessário aplicar novamente a operação *init*. Durante essa operação, ele irá perguntar se você quer copiar o arquivo de estado existente para o novo backend. A resposta é um óbvio sim:

```bash
$ terraform init

Initializing the backend...
Backend configuration changed!

Terraform has detected that the configuration specified for the backend
has changed. Terraform will now check for existing state in the backends.


Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "s3" backend to the
  newly configured "s3" backend. No existing state was found in the newly
  configured "s3" backend. Do you want to copy this state to the new "s3"
  backend? Enter "yes" to copy and "no" to start with an empty state.

  Enter a value: yes

$ terraform plan -out planfile
...
$ terraforma apply planfile
...
```

Após as últimas operações, podemos observar nosso arquivo de estados seguro em seu bucket s3:

```bash
$ aws s3 ls terraform-bootstrap-dev-sres/terraform-bootstrap/
2021-03-10 23:18:56      18466
2021-03-10 23:52:00      22110 tfstate
```

E com isso completamos os dois primeiros requisitos.

## Requisito 3: Usando dynamodb para controlar a concorrência de execução

Se você executar um '*terraform apply*' sem um plan, ele vai solicitar um prompt de confirmação para você:

```bash
$ terraform apply
Acquiring state lock. This may take a few moments...
...
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Se você abrir outro terminal e tentar executar o mesmo programa, receberá um erro:

```bash
terraform plan
Acquiring state lock. This may take a few moments...

Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        7ec4834e-f004-abd5-6b20-31ba68ba07e7
  Path:      terraform-bootstrap=dev-sres/terraform-bootstrap/tfstate
  Operation: OperationTypeApply
  Who:       the-ops-hero@localhost
  Version:   0.14.7
  Created:   2021-03-11 05:06:10.5604146 +0000 UTC
  Info:


Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

Isso porque o terraform aplica um *lock* para evitar execução concorrente, que pode ser devastadora em alguns casos.

Obviamente, se alguém baixar o código do seu repositório git e executá-lo em outra máquina, ele conseguirá. Alguém eventualmente receberá um erro porque algo criado já existe.

Para impedir isso, é importante usar um sistema de controle de concorrência centralizado. A melhor maneira de fazer isso na AWS é usando uma tabela Dynamodb.

Qual a melhor maneira de criar uma tabela DynamoDB? Usando Terraform, oras!

### Criando uma tabela Dynamodb

Essa etapa não é que seja difícil, mas falta informação.

A primeira delas está escondida lá embaixo na [documentação oficial](https://www.terraform.io/docs/language/settings/backends/s3.html#dynamodb-state-locking):
> dynamodb_table - (Optional) Name of DynamoDB Table to use for state locking and consistency. The table must have a primary key named LockID with type of string. If not configured, state locking will be disabled.

Então sabemos que a nossa tabela de DynamoDB precisa de um campo LockID. Se não existir, o state locking vai ser desativado.

O segundo detalhe é o 'billing_mode'. Temos as opções 'on-demand' e 'provisioned'. Em 'on-demand', ele está livre para consumir recursos que ele quiser. Em 'provisioned', você precisa fazer as contas. A parte importante é: **apenas provisioned está incluso no free tier**. Se você estiver apenas brincando de laboratório na AWS e não quer cobrança, use este.

Então, se usar 'provisioned', vamos precisar especificar read e write capacity. Mas quanto? 

O 'free tier' nos garante 25 de cada, além de 25GB de espaço. Na documentação oficial para criação de tabelas, ele relaciona ambos com 20 unidades. Se você quiser ler mais a respeito, [siga em frente](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html). Eu arbitrariamente dividi por 10 e coloquei 2 read requests.
```bash
# dynamodb.tf
# resource "aws_dynamodb_table" "terraform-dev-sres" {
  name           = "terraform-bootstrap-dev-sres"
  billing_mode   = "PROVISIONED"
  read_capacity  = 2
  write_capacity = 2
  hash_key       = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Name        = "terraform-bootstrap-dev-sres"
    CreatedBy   = "terraform"
    Environment = "production"
  }
}
```
### Configurando a tabela DynamoDB para controlar concorrência

Após um combo plan/apply, estamos aptos a modificar nosso backend novamente:
```bash
terraform {
  backend "s3" {
    bucket         = "terraform-bootstrap-dev-sres"
    key            = "terraform-bootstrap/tfstate"
    region         = "sa-east-1"
    dynamodb_table = "terraform-dev-sres"
  }
}
```

Novamente, toda modificação de 'backend' demanda um novo init:

```bash
$ terraform init
... mude alguma coisa em algum lugar...
$ terraform apply 
...
  Enter a value: 
```

**Não** dê yes.

Execute o seguinte comando para olhar a sua tabela:

```bash
$ aws dynamodb scan --table-name terraform-dev-sres
{
    "Items": [
        {
            "LockID": {
                "S": "terraform-bootstrap-dev-sres/terraform-bootstrap/tfstate"
            },
            "Info": {
                "S": "{\"ID\":\"c2869c54-ff40-2824-ffee-1a409a834d6e\",\"Operation\":\"OperationTypeApply\",\"Info\":\"\",\"Who\":\"the-ops-hero@localhost\",\"Version\":\"0.14.7\",\"Created\":\"2021-03-11T05:21:19.5567361Z\",\"Path\":\"terraform--bootstrap-dev-sres/terraform-bootstrap/tfstate\"}"
            }
        },
        {
            "Digest": {
                "S": "4cca273212ee805d228e8d662d4f4c6d"
            },
            "LockID": {
                "S": "terraform-bootstrap-dev-sres/terraform-bootstrap/tfstate-md5"
            }
        },
        {
            "Digest": {
                "S": "413f17b8b57764a4c7339837f75e7516"
            },
            "LockID": {
                "S": "terraform-bootstrap-dev-sres/terraform-bootstrap/tfstatex-md5"
            }
        }
    ],
    "Count": 3,
    "ScannedCount": 3,
    "ConsumedCapacity": null
}
``` 

Se você der um 'terraform plan' em outra janela, é exatamente isso que você vai ver:

```bash
$ terraform plan
Acquiring state lock. This may take a few moments...

Error: Error locking state: Error acquiring the state lock: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        c2869c54-ff40-2824-ffee-1a409a834d6e
  Path:      terraform-bootstrap-dev-sres/terraform-bootstrap/tfstate
  Operation: OperationTypeApply
  Who:       the-ops-hero@localhost
  Version:   0.14.7
  Created:   2021-03-11 05:21:19.5567361 +0000 UTC
  Info:


Terraform acquires a state lock to protect the state from being written
by multiple users at the same time. Please resolve the issue above and try
again. For most commands, you can disable locking with the "-lock=false"
flag, but this is not recommended.
```

Confirme as mudanças.

Suba seu código para o git.

Sua aventura Terraform com AWS agora está **pronta para começar**!









