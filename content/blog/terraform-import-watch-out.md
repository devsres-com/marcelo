+++
title = "Cuidado com terraform import"
description = "Importando recursos que usam count e for_each"
author = "Marcelo Andrade"
date = "2020-10-06"
tags = ["terraform", "intermediário", "vmware"]
categories = ["terraform", "intermediário", "vmware"]
[[images]]
  src = "img/reusable/square-into-circle.jpg"
  alt = "Square into circle and hammer"
  stretch = ""
+++

(Nota: eu tenho tentado produzir conteúdo de qualidade seguindo uma linha de raciocínio com começo, meio e fim, com valor histórico e altamente apreciável. Com isso, meu último post foi mês passado e eu tenho mais de 30 drafts a concluir. Portanto, vou tornar isso aqui um braindump de conteúdos aleatórios que eu julgar relevante. Foi mal!)

___


Quem **não adora Terraform**?

Basta escrever meia dúzia de arquivos que algum programa magicamente interpreta tudo e cria coisas mágicas para você. Fantástico! O único problema é aprender a usar direito.

A Hashicorp, até alguns anos atrás, fazia documentações tão crípticas e ilegíveis que eventualmente perceberam que era necessário um esforço maior para "mentes menores" compreenderem seus softwares. Com isso, investiram somas substanciais de dinheiro fazendo sites como o [Learn Hashicorp](https://learn.hashicorp.com/terraform) ou mesmo workshops gratuitos interativos com [Instruqt](https://play.instruqt.com/hashicorp). Eles realmente têm feito um bom trabalho nessa seara.

Em muitos aspectos, o Terraform ainda é deficiente. 

Às vezes é **pura frescura**: vide o **terraform-provider-kubernetes**, que simplesmente se recusava a implementar APIs betas por anos, tornando-o praticamente inútil. E mesmo quando **deployments** chegaram à GA,  demorou [quase um ano](https://github.com/hashicorp/terraform-provider-kubernetes/issues/3) para implementar este elemento que é considerado como pedra fundamental para qualquer aplicação Kubernetes.

Mas muitas vezes nem é culpa da Hashicorp: a integração com **VMware**, mesmo com as últimas funcionalidades implementadas pela última versão, é, literalmente, **um horror**. E o problema aqui são as severas limitações funcionais das APIs públicas disponibilizadas pela própria VMWare. 

Mas chega de 'rant': vamos a conteúdo!

## Terraform import

 Todo mundo sabe que o Terraform precisa de total controle sobre o que cria; o que existe **tem que ser criado por ele, e isso é inegociável**.

 Quer dizer, nem tanto; eles dão uma colher de chá para você tentar adaptar uma infraestrutura já existente à sua automação: o comando **terraform import**.

 Suponha que, agora, você usa Terraform e conseguiu criar 2 máquinas usando seu novíssimo programa! Como você faz para 'incorporar' as 2000 máqunas que já existem na sua infraestrutura? Executando 2000 vezes (no mínimo) o comando '*import*' especificando os '*terraform resources*' que as máquinas representam, bem como um identificador alienígena que varia completamente de maneira praticamente imprevisível de acordo com o tipo de '*provider*'! (Tudo bem, é melhor que toda infraestrutura tem que ser criado por ele e isso ser inegociável).

 **Nosso problema**: cluster Kubernetes com 10 máquinas virtuais usando **VMWare Vsphere**. Precisamos adicionar mais 4. Obviamente não usamos Terraform no passado. Como fazer?

 Comando da [documentação oficial](https://registry.terraform.io/providers/hashicorp/vsphere/latest/docs/resources/virtual_machine#importing):

 ```
 terraform import vsphere_virtual_machine.vm /dc1/vm/srv1
 ```

O colega com pouca experiência que está trilhando os tortuosos caminhos dos programas Terraform criados por mim (que consigo ser ainda mais críptico que a própria Hashicorp) falou:

 > "Ufa! Ainda bem que é só isso, certo?"

 Claro que não!

 * Se o seu programa (tipo, literalmente, no diretório corrente) criar '**resources**' do tipo '*vsphere_virtual_machine*'  que sejam escalares (i.e., não usem '*for_each*' ou '*count*'), o primeiro parãmetro do comando está correto. Caso contrário, está **errado**!
 * /dc1/vm/srv1 é obviamente um placeholder; você precisa descobrir o 'caminho VMware' das suas máquinas virtuais, e, se você não entende de VMWare, pode ter alguma dificuldade com a nomenclatura.

 A segunda parte é fácil: basta compor o nome do datacenter com a string arbitrária vm com as 'pastas' criadas no Vcenter e, por fim, o nome dos servidores:

 * "/bsa/vm/infra/10069/kubernetes df1/hosts/master1"
 * "/bsa/vm/infra/10069/kubernetes df1/hosts/master2"
 * "/bsa/vm/infra/10069/kubernetes df1/hosts/worker1"
 * "/bsa/vm/infra/10069/kubernetes df1/hosts/worker2"

 Tranquilo. (Espero que tenha acesso à API do VMWare para descobrir isso!)

 A primeira parte, por outro lado, irá variar **radicalmente** de acordo com o Terraform que você está usando.

 Em nosso caso, por exemplo, eu criei um módulo terraform que encapsula a criação dos recursos VMWare VSphere necessários. Esse encapsulamento 'vaza' por uma série de razões, mas me permite passar as máquinas como uma variável map para o módulo mais ou menos da seguinte forma:

 ```
 vms = {
    "master1" = {
      "profile" = "k8s_worker"
      "hostname" = "master1"
      "network" = {
        "rede1" = "192.168.0.1",
        "rede2" = "10.0.0.1"
      }
    }
    "worker1" = {
      "profile" = "k8s_worker"
      "hostname" = "worker1"
      "network" = {
        "rede1" = "192.168.0.101",
        "rede2" = "10.0.0.101",        
        "rede3" = "172.16.0.1",
      }
    }
    "ingress1" = {
      "profile" = "k8s_ingress"
      "hostname" = "ingress1"
      "network" = {
        "rede1" = "192.168.0.201",
        "rede2" = "10.0.0.201",        
        "rede4" = "200.160.2.1",
      }
    }
 ```
___ 

(E aqui, um conselho: **evite ao máximo** criar maps de maps de maps ou maps de arrays de maps como eu gosto de fazer. É uma ideia idiota, que deixa seu programa Terraform praticamente incompreensível para seres humanos. Eu adoro fazer assim por razões, mas definitivamente não recomendo).
___

Enfim, a criação dos meus resources estão encapsulados no módulo. 

Então, o comando '*terraform import*' deve, necessariamente, fazer referência ao resource **dentro do módulo**:

* module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm

E aqui está a pegadinha: aqui está a definição do meu resource vm:

```
resource "vsphere_virtual_machine" "vm" {
  for_each  = local.vms
  name      = each.value.hostname
  ...
```

Portanto, o meu 'resource' vm **não** é um tipo simples (ou escalar, como eu chamo), e sim um tipo complexo (por causa do uso do iterador for_each) do tipo **map**.

O comando **correto** para importar máquinas já existentes, portanto, é assim:

```shell
# terraform import 'module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm["master2"]' '/bsa/vm/infra/10069/kubernetes df1/hosts/master2'

# terraform import 'module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm["worker2"]' '/bsa/vm/infra/10069/kubernetes df1/hosts/worker2'
...
```
 
## Tá, mas e daí?

E daí que: o que pode acontecer se eu executar o **comando errado**?

```shell
# terraform import module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm '/bsa/vm/infra/10069/kubernetes df1/hosts/worker2'
```

Acima, erramos o comando, e importamos 'worker2', que deveria ser membro de um map, para vsphere_virtual_machine.vm do módulo diretamente. 

O que se espera normalmente? Um erro de execução, certo?

Vai dar erro, mas não no import. O import irá executar de maneira bem sucedida. Mas, depois disso, provavelmente tudo relacionado a este programa falhará.

Após a execução do comando errado, fui agraciado com a seguinte mensagem:

```
# terraform apply 
...
module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm["ingress3"]: Refreshing state... [id=421ab407-1bc3-dceb-6906-72fe3fad4c0e]

Error: Unsupported attribute

  on .terraform/modules/vmware-cluster-k8s-raw/locals.tf line 104, in locals:
 104:   vms_result = { for key, value in vsphere_virtual_machine.vm : key => { for network in value.network_interface : network.network_id => network.mac_address } } 
```

O que esse erro quer dizer? Quer dizer que aquela sequência mágica de 'terraform maps comprehension' falhou. Por quê? Putz, vai adivinhar.

Meu colega aplicou um ["Senhor, eu desisto, Senhor!"](https://www.youtube.com/watch?v=oTWW4GXTXS0):

{{< youtube oTWW4GXTXS0 >}}

E isso, obviamente, faz qualquer sênior SRE muito feliz!
___

Como fui eu que pari a besta, voltei para entender.

Observei o seguinte descrevendo o arquivo de estados:

```
# terraform state list
module.vmware-cluster-k8s-raw.data.vsphere_compute_cluster.compute_cluster
module.vmware-cluster-k8s-raw.data.vsphere_datacenter.cluster_datacenter
module.vmware-cluster-k8s-raw.data.vsphere_datastore_cluster.cluster_datastore
module.vmware-cluster-k8s-raw.data.vsphere_network.networks["cluster"]
module.vmware-cluster-k8s-raw.data.vsphere_network.networks["ingress"]
module.vmware-cluster-k8s-raw.data.vsphere_network.networks["management"]
module.vmware-cluster-k8s-raw.data.vsphere_network.networks["storage"]
module.vmware-cluster-k8s-raw.vsphere_folder.folder
module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm
module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm["worker1"]
module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm["worker2"]
```

Algo estranho aqui: temos dois tipos de ocorrências para o objeto que representa o map vm:

* module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm: um 'escalar'!
* module.vmware-cluster-k8s-raw.vsphere_virtual_machine.vm[]: este sim com todas as ocorrências adequadas!

O comando errado de import criou uma entrada errada no '*state file*' que bagunçou o funcionamento da lógica do módulo. 

Nossa situação foi um pouco pior, porque o comando gerou a máquina em um lugar no VMWare que ela não deveria ser criada por razões desconhecidas (novamente, em vez de dar erro!), e que não tínhamos permissão para remover também por razões desconhecidas.

A **solução** para este caso foi a remoção da máquina problemática do 'map' de objetos criados; após um apply bem sucedido com a remoção da máquina problemática, o Terraform foi capaz de perceber o desvio no arquivo de estado e corrigiu sozinho, removendo a entrada inválida e me poupando (desta vez!) a experiência aterrorizante de manipular o estado com '*terraform state*', então, se era para isos que você veio aqui, desculpe frustrar sua expectativa!

---

Para aprender com esta experiência:

* Criar módulos Terraform para tudo parece uma ideia genial, mas quanto mais você encapsula achando que está reusando código, mais estará duplicando entradas de variáveis e se distanciando das validações básicas do programa;
* Evite usar de maneira leviana maps, arrays, counts e for_eachs a menos que você esteja disposto a ir até o fim por suas escolhas - ou trabalhe com alguém insano que use em todo canto; aí, meu amigo, meus pêsames;
* Não execute 'imports' até ter certeza absoluta do que está fazendo.







