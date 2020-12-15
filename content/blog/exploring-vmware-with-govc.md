+++
title = "GOVC"
description = "Como explorar o VMWare pela linha de comando"
author = "Marcelo Andrade"
date = "2020-12-15"
tags = ["vmware","govc"]
categories = ["vmware", "govc"]
[[images]]
  src = "img/reusable/hammer-screw.jpg"
  alt = "If all you have is a hammer, everything looks like a nail."
  stretch = ""
+++

O projeto [**govmomi**](https://github.com/vmware/govmomi) oferece uma biblioteca em Go Para interação com as APIs VMWare Vsphere (ESXi ou vCenter). O programa [**govc**](https://github.com/vmware/govmomi/tree/master/govc) é o binário que te permite executar coisas na linha de comando.

Eu nunca implantei VMWare do zero; conheço um emaranhado de conceitos usando a técnica que todo ultrageneralista usa: sair aprendendo uma série de coisas desconexas sobre o objeto de estudo que são o suficiente para desempenhar determinada tarefa, torcendo para que, algum dia, eu possa voltar àquele assunto e completar o que falta.

Ironicamente, tem funcionado.

O objetivo de hoje? Descobrir tudo que existe no ambiente e onde estão minhas coisas já criadas.

Por que isso? Para implementar a automação com [Terraform](https://www.terraform.io/). Eu preciso de um monte de informações, e **me recuso** a usar interfaces gráficas coloridas. Isso é contra os princípios de todo bom #Ops.

# Abrindo a caixa de ferramentas

A minha caixa de ferramentas, na verdade, vem com um único martelo. Portanto, daqui para frente, tudo é prego.

Existem diversas opções viáveis para administrar VMWare a partir da linha de comando. Não é o que eu estou procurando. Eu quero algo que seja capaz de me trazer as informações necessárias sobre o meu ambiente para que eu possa viabilizar o uso do Terraform, já que as estruturas de dados (objetos '*data*') que ele pode importar precisam que eu especifique nomes e IDs. 

Para usar o **govc**, é necessário configurar algumas variáveis de ambiente e um conjunto de credenciais:

```
# # Deixe de ser preguiçoso e importe a CA auto assinada na sua máquina em vez de usar GOVC_INSECURE
# export GOVC_INSECURE=true

# export GOVC_URL=https://endereco-ip-do-seu-vcenter:443

# # Credenciais. Não preciso dizer que você não deve fazer isso em uma máquina com root compartilhado...
# export GOVC_USERNAME=$USER
# export GOVC_PASSWORD=Cdmp1980@69
```
Peça para os administradores VMWare criarem um usuário com 'readonly' no cluster. Não precisamos de mais nada além disso para fazer a exploração.

Existem maneiras mais inteligentes de fazer a autenticação, como, por exemplo, com um token. Mas eu ainda estou esperando isso aqui acontecer:

```
# https://github.com/vmware/govmomi/issues/1824
6 Feb 2020
Some more info on the auth methods here: kubernetes/kubernetes#63209
We should put some of that in the govmomi docs.
```
Apenas para referência, é possível também usar algumas das variáveis abaixo:
* GOVC_TLS_CA_CERTS
* GOVC_TLS_KNOWN_HOSTS
* GOVC_TLS_HANDSHAKE_TIMEOUT
* GOVC_DATACENTER
* GOVC_DATASTORE
* GOVC_NETWORK
* GOVC_RESOURCE_POOL
* GOVC_HOST

GOVC_DATACENTER, por exemplo, me economizaria todas as vezes em que eu precisei passar o atributo *-dc* nos comandos abaixo.

Mas vamos ao que importa.

# govc find

Como descobrir TUDO que eu preciso sobre minha infraestrutura contando apenas com minhas credenciais e sabendo que, "lá dentro", tem máquinas minhas?

Basicamente usando o comando **find** do *govc*. 

Este comando conta com um parâmetro *-type* que será a chave para as descobertas.

```
# govc find 
...
The '-type' flag value can be a managed entity type or one of the following aliases:

  a    VirtualApp
  c    ClusterComputeResource
  d    Datacenter
  f    Folder
  g    DistributedVirtualPortgroup
  h    HostSystem
  m    VirtualMachine
  n    Network
  o    OpaqueNetwork
  p    ResourcePool
  r    ComputeResource
  s    Datastore
  w    DistributedVirtualSwitch
...

```

Então vamos lá.

# Fase 1: o mínimo necessário

## Quantos datacenters estão disponíveis no Vcenter para mim?

```
# govc find . -type Datacenter
/DATACENTER-PE
```

Um. Ótimo. Menos tapetes para procurar sujeira embaixo.

Quer informações sobre o que tem pela frente?

```
# govc datacenter.info -dc DATACENTER-PE
Name:                DATACENTER-PE
  Path:              /DATACENTER-PE
  Hosts:             9
  Clusters:          3
  Virtual Machines:  90
  Networks:          6
  Datastores:        9
```

## Quantos clusters de processamento diferentes existem?

Direto e reto:

```
# govc find -dc DATACENTER-PE -type ClusterComputeResource
/DATACENTER-PE/host/cc_pool01
/DATACENTER-PE/host/cc_pool02
/DATACENTER-PE/host/cc_pool03
```

## Quantas redes diferentes existem?

Mais um direto:

```
# govc find -dc DATACENTER-PE -type  Network
./network/kubernetes_cluster_producao
./network/kubernetes_cluster_homologacao
./network/gerencia
./network/internet
./network/backup
./network/kubernetes_cluster_lab
```

## Quantos datastore clusters diferentes existem?

Nós não usamos os *Datastores* diretamente; fazemos uso do conceito de [*Datastore Clusters*](https://docs.vmware.com/en/VMware-Validated-Design/5.0/com.vmware.vvd.sddc-design.doc/GUID-90F8BCEB-F917-450F-A5D9-3C860446EF84.html).

Aqui confesso que vou ficar devendo um *find* melhor: o melhor que consegui foi listar todos os *Datastores*; o nome do *DatastoreCluster* é o nome da pasta em que o *Datastore* está, como visto abaixo:

```
# govc find -dc DATACENTER-PE -type ClusterComputeResource
/DATACENTER-PE/datastore/datastorecluster01/dsc1_1
/DATACENTER-PE/datastore/datastorecluster01/dsc1_2
/DATACENTER-PE/datastore/datastorecluster01/dsc1_3
/DATACENTER-PE/datastore/datastorecluster02/dsc2_1
/DATACENTER-PE/datastore/datastorecluster02/dsc2_2
/DATACENTER-PE/datastore/datastorecluster02/dsc2_3
/DATACENTER-PE/datastore/datastorecluster03/dsc3_1
/DATACENTER-PE/datastore/datastorecluster03/dsc3_2
/DATACENTER-PE/datastore/datastorecluster03/dsc3_3

# # mas eu só quero os datastoreclusters!
# govc find -dc DATACENTER-PE -type ClusterComputeResource | rev | cut -f2 -d'/' | rev | sort | uniq -c
  3 datastorecluster01
  3 datastorecluster02
  3 datastorecluster03

# # uma alternativa é usar o comando govc ls
# govc ls -dc DATACENTER-PE'
/DATACENTER-PE/vm
/DATACENTER-PE/network
/DATACENTER-PE/host
/DATACENTER-PE/datastore

# govc ls /ESTALEIRO-SPO/datastore
/DATACENTER-PE/datastore/datastorecluster01
/DATACENTER-PE/datastore/datastorecluster02
/DATACENTER-PE/datastore/datastorecluster03
```

É bem provável que não haja uma maneira de usar o *find* para esta finalidade, pois o comando '*govc datacenter.info*' mostra apenas o número de *Datastores*, enquanto para os clusters de computação, ele mostra adequadamente que tenho 3 clusters e 9 máquinas.

Mas não tome minha palavra como final. Eu estou **muito longe** de ser um especialista em VMWare.

# Tenho tudo que preciso para o Terraform?

O meu principal uso de Terraform, no momento, é o de criar 'cascas' vazias de máquinas virtuais que irão ser instaladas usando PXE. 

Se você achou a ideia idiota, eu tenho minhas razões! Mas tolere essa particularidade e acompanhe abaixo o que é necessário para criar um *Terraform resource* do tipo *vsphere_virtual_machine* dessa forma:

**Nota**: Não entendeu nada dos Terraform listados abaixo? Entre em contato por qualquer canal - Twitter, Linkedin, Facebook, Instagram... E me deixe saber que você quer saber mais sobre Terraform!


## Recuperação do objeto datacenter. 
Preciso:
* do nome do datacenter;

```
data "vsphere_datacenter" "cluster_datacenter" {
  name          = lookup(local.cluster_vsphere, "vsphere_datacenter")
}
```

## Recuperação do objeto datacenter. 
Preciso:
* do nome do datastore cluster;
* do id do datacenter (recuperado com a estrutura *data.vsphere_datacenter*);

```
data "vsphere_datastore_cluster" "cluster_datastore" {
  name          = lookup(local.cluster_vsphere, "vsphere_datastore_cluster")
  datacenter_id = data.vsphere_datacenter.cluster_datacenter.id
}
```
## Recuperação do cluster de computação.
Preciso:
* do nome do cluster de computação;
* * do id do datacenter (*data.vsphere_datacenter.nome.id*);
```
data "vsphere_compute_cluster" "compute_cluster" {
  name          = lookup(local.cluster_vsphere, "vsphere_compute_cluster")
  datacenter_id = data.vsphere_datacenter.cluster_datacenter.id
}
```
## Recuperação das redes.
Preciso:
* do nome da rede;
* do id do datacenter.

```
data "vsphere_network" "networks" {
  for_each = local.cluster_network_profiles

  name = each.value
  datacenter_id = data.vsphere_datacenter.cluster_datacenter.id
}
```

## Resource VM

Aqui está o resource que uso para criar as VMs. 

Eu posso trocar metade dos lookups por referências diretas. Mas, enquanto fazia, por incrível que pareça, ficava mais fácil para poder acertar a profundidade da estrutura de dados que eu optei usar como 'input' no *terraform.tfvars*.

> **Cicatrizes de batalha**: não faça coisas complexas no Terraform, ainda que você consiga apenas porque você pode; você não trabalha sozinho, e alguém vai precisar entender algum dia.

De qualquer forma, de novo, esqueça o objeto Terraform ilegível por causa da minha tendência natural de fazer coisas absurdas. Observe que criei as máquinas com os três recursos de *data* listados acima.

```
resource "vsphere_virtual_machine" "vm" {
  for_each  = local.vms
  name      = each.value.hostname

  # Pode pular tudo até o (1).  
  cpu_hot_add_enabled = true
  cpu_hot_remove_enabled = true
  memory_hot_add_enabled = true
  guest_id = "coreos64Guest"

  num_cpus = lookup(lookup(local.cluster_profiles,each.value.profile), "num_cpus")
  memory   = lookup(lookup(local.cluster_profiles,each.value.profile), "memory")
  disk {
    label = "${each.value.hostname}-disk"
    size  = lookup(lookup(local.cluster_profiles,each.value.profile), "disk_size")
    unit_number = 0
    eagerly_scrub = false
    thin_provisioned = true
  }

  # Truque sujos necessários para falar para o Terraform não esperar as máquinas.  
  wait_for_guest_net_timeout = 0

  # (1): cluster de computação
  resource_pool_id = data.vsphere_compute_cluster.compute_cluster.resource_pool_id

  # (2): datastore cluster
  datastore_cluster_id = data.vsphere_datastore_cluster.cluster_datastore.id
  folder = vsphere_folder.folder.path
  
  # Nós gostamos de máquinas virtuais com muitas interfaces, e cada uma com uma quantidade variável delas.
  dynamic "network_interface" {

    for_each = each.value.network

    content {
      # (3): redes.
      network_id = lookup(lookup(data.vsphere_network.networks, network_interface.key), "id")
    }

  }

  # O backup adiciona uma tag; precisamos ignorá-la ou o Terraform irá tentar apagá-la.
  lifecycle {
    ignore_changes= [
      annotation,
      # "vapp.0.properties",
    ]
  }
}
```

## Resumindo

Para a minha necessidade inicial (criação de cascas para instalação via PXE), a resposta é **sim**, os comandos *govc* listados acima são suficientes para que eu recupere todas as informações por meio de objetos *data* do Terraform.

Por mais 'tosco' que possa parecer, sim, o Terraform só precisa dos nomes dos objetos, e não de ids obscuros e esquisitos.

> **Cicatrizes de batalha**: Se você prefere deixar para lá esse papo de linha de comando e quer usar a interface gráfica para ver todos os nomes e transcrever na mão, se prepare para surpresas desagradáveis se os seus administradores de VMWare forem tão trolls quanto os meus.

O uso de nomes demanda um **cuidado especial** por causa da tolerância do VMware com caracteres especiais. O VMWare deixa você criar a rede da seguinte maneira na interface gráfica:

* **rede_cluster_kubernetes_1_192.168.69.0/24**

Legal, né? Mas olha como é o nome dela de verdade:

```
# govc find -dc DATACENTER-PE -type  Network
...
./network/kubernetes_cluster_producao_192.168.69.0%2f24
...
```

O nome que precisa ir no Terraform é o listado pelo Govc.

# Isso é tudo?

Infelizmente, não.

Meu problema atual é que preciso 'particionar' um host VMWare em um determinado número de máquinas virtuais, usando o disco local daquela máquina como datastore.

Isso quer dizer que precisarei fazer uma parte 2 tentando levantar todas essas informações adicionais!

Alguém tem interesse?

(Na verdade a demanda era para criar máquinas virtuais e entregar o disco como por meio de [**Raw Device Mapping**](https://docs.vmware.com/en/VMware-vSphere/6.0/com.vmware.vsphere.storage.doc/GUID-9E206B41-4B2D-48F0-85A3-B8715D78E846.html), mas aparentemente isso **não é possível com o Terraform** e [**talvez nunca seja**](https://github.com/hashicorp/terraform-provider-vsphere/issues/397),)



