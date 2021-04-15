+++
title = "Refatorando projetos de Terraform"
description = "Ou: como fazer cirurgias em statefiles"
author = "Marcelo Andrade"
date = "2021-04-15"
tags = ["terraform"]
categories = ["terraform"]
[[images]]
  src = "img/reusable/toy-operation.jpg"
  alt = "Operation"
  stretch = ""
+++

Então, você fez algumas configurações em Terraform enquanto está aprendendo.

Agora aprendeu que o ideal é agrupar as configurações comuns em **Módulos** para garantir reusabilidade de código. É hora de "empacotá-las".

Só que você já usou seu código algumas vezes. E aí, o que acontece?

## Criação do módulo 

Fiz um exemplo bem simples: um código que cria um grupo e um repositório de Gitlab. Vou criar um módulo para ele.

```hcl
# modulo/gitlab-crap/main.tf
# 
# Um grupo é uma 'pasta' lógica no Gitlab para agrupar repositórios.
# Não há equivalente no Github; o mais próximo é a maneira como repositórios são criados sob Organizations.
#
resource "gitlab_group" "cluster_group" {
  # Nome é o que será exibido. Pode ser uma string 'complicada'.
  name        = var.cluster_id
  
  # Path é a parte que vai na URL. Precisa ser mais simples.
  path        = var.cluster_id
  
  # O grupo é criado na 'raiz' ou em outro grupo? No meu caso, é em outro grupo.
  parent_id   = var.gitlab_parent_id
  
  visibility_level = "private"
  description = "Cluster ${var.cluster_id}"
}

#
# O repositório git é chamado de 'Project' no Gitlab.
#
resource "gitlab_project" "cluster" {
  # Nome é o que será exibido. Pode ser uma string 'complicada'.
  name        = var.cluster_id

  # Aqui vai o grupo em que o repositório vai ser criado. Podia ser parent_id, né? Só que não é.
  namespace_id = gitlab_group.cluster_group.id

  visibility_level = "private"
  description = "Cluster ${var.cluster_id}"
}
```

Ok. Do jeito que está, o código obviamente é bem simples, apenas para exemplificar o problema.

Todo código Terraform tem pelo menos um módulo; o principal, que é onde você executa Terraform init, plan, apply é chamado de módulo raiz ou **Root Module**. 

> Dica: O conceito de *Root Module* pode parecer bobagem, mas a terminologia é importante se você pretende fazer a prova de certificação Hashicorp Terraform Associate.

A ideia é migrar esse código para um módulo. Ok, sem mistérios.

Criar um módulo é só combinar o arquivo acima com um arquivo de definição de variáveis, que serão as **inputs** do módulo:

```
# modulo/gitlab-crap/variables.tf
variable cluster_id {
  type = string
  description = "Id do cluster no formato spx/dfy."
}

variable gitlab_parent_id {
  type = string
  description = "Id do Grupo do Gitlab em que a estrutura será criada."
}
```

Como estamos usando as versões mais novas do Terraform, também precisamos da definição das versões de providers exigidas:

```
terraform {
  required_providers {
    gitlab = {
      source = "gitlabhq/gitlab"
      version = "~>3.6.0"
    }
  }
}
```

Este requisito é particularmente necessário porque o Terraform não encontrará o **provider** gitlab automaticamente por não ser 'fornecido' pela Hashicorp.

## Ajuste do módulo raiz

O código do módulo raiz agora é resumido a:

```
terraform {
  required_providers {
    gitlab = {
      source = "gitlabhq/gitlab"
      version = "~>3.6.0"
    }
  }
}

provider "gitlab" {
}

data "gitlab_group" "workgroup" {
  full_path = "CDNPS/incubus"
}

module "gitlab_crap" {

  # 
  source = "/srv/terraform/modules/gitlab-crap"
  cluster_id = terraform.workspace
  gitlab_parent_id = data.gitlab_group.workgroup.group_id

}
```

Sim, você precisa duplicar as configurações de required_providers em todos os módulos - é a maneira que o Teraform encontra para garantir que o programa funcionará.

> A atribuição '~>' no campo *version* do *map* associado ao *provider* gitlab indica que podemos usar qualquer nova *patch release* da *minor version* especificada (ou seja, 3.6.*). Enfatizando: se a versão 3.6.1 for lançada, meu código poderá fazer uso dessa versão, mas se a atualização for 3.7.0, o Terraform **a ignorará**.]

## O que vai acontecer?

Okay. Meu código já foi modularizado. Eu já rodei o código antes, e coisas foram criadas. 

* O que eu gostaria que acontecesse? Que **nada** fosse modificado, já que efetivamente eu apenas movi código de lugar, não modifiquei nada da minha infraestrutura.

* O que vai acontecer?

Isso aqui:

```
# Gerar arquivo das modificações
$ terraform plan -out plan

# Como ler o plan do Terraform? Com o comando show -json e jq.
$ terraform show -json plan | jq -r '.resource_changes[] | "\(.change.actions[]):\(.address)"'
delete:gitlab_group.cluster_group
delete:gitlab_project.cluster
create:module.gitlab-crap.gitlab_group.cluster_group
create:module.gitlab-crap.gitlab_project.cluster
```

Logo, em vez de simplesmente 'renomear' os *resources* criados, o Terraform irá destruir tudo que existia e recriar a mesma coisa sob o novo nome de resource, agora referenciando o *module*.

Observe que são os mesmos nomes, exceto pelo prefixo.

## O que fazer?

No meu exemplo bobo, eu podia simplesmente ignorar e deixar rolar.

Mas preferi parar o trabalho gastar um tempo escrevendo este post para você! Então vamos operar no *Statefile*, para impedir essa devastação na infraestrutura.

```
# Este comando exibe o que está no statefile:
$ terraform state list
data.gitlab_group.workgroup
gitlab_group.cluster_group
gitlab_project.cluster
```

Vamos mover os recursos (não precisamos mexer no data) para colocá-los sob o módulo. Basta replicar a estrutura exibida pelo *Plan*.

```
$ terraform state mv gitlab_project.cluster module.gitlab-crap.gitlab_project.cluster
Move "gitlab_project.cluster" to "module.gitlab-crap.gitlab_project.cluster"
Successfully moved 1 object(s).

$ terraform state mv gitlab_group.cluster_group module.gitlab-crap.gitlab_group.cluster_group
Move "gitlab_group.cluster_group" to "module.gitlab-crap.gitlab_group.cluster_group"
Successfully moved 1 object(s).
```

Aquela verificada para ver o state:

```
$ terraform state list
data.gitlab_group.workgroup
module.gitlab-crap.gitlab_group.cluster_group
module.gitlab-crap.gitlab_project.cluster
```

Será que funcionou?

```
$ terraform plan
module.gitlab_terraform_pipelines.gitlab_group.cluster_group: Refreshing state... [id=11914]
module.gitlab_terraform_pipelines.gitlab_project.cluster: Refreshing state... [id=19325]

No changes. Infrastructure is up-to-date.

This means that Terraform did not detect any differences between your
configuration and real physical resources that exist. As a result, no
actions need to be performed.
```

Win!

---

Como o post não era de shell script, eu preferi clareza over shell script quality, mas poderíamos ter feito assim:

```
$ for i in $( terraform state list | tail -n+2); do \
>   terraform state mv {,module.gitlab-crap.}$i
> done
```