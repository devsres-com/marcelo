+++
title = "Kubernetes on-premises - parte 2"
description = "Redes em Kubernetes - CNI e Calico"
author = "Marcelo Andrade"
date = "2020-12-19"
tags = ["kubernetes", "on-prem", "passado","network"]
categories = ["kubernetes", "on-prem", "passado", "network"]
[[images]]
  src = "img/reusable/blueprint.jpg"
  alt = "Building from ground up"
  stretch = ""
+++

Um pouco atrasado pelos diversos "detours", mas seguimos com a continuação da Parte 1!

* [Kubernetes on-prem - parte 1]({{< ref "blog/kubernetes-on-prem-1.md" >}}): se você perdeu o initial rambling!

# Redes no Kubernetes: componentes plugáveis

A primeira regra sobre redes no Kubernetes é que você não fala sobre redes no Kubernetes.

Agora sério: o Kubernetes, conceitualmente, [descreve vários aspectos](https://kubernetes.io/docs/concepts/cluster-administration/networking/) de como deve ocorrer a comunicação entre seus componentes. Mas ele não descreve o **como implementar**; ele "terceiriza" essas linhas gerais para outros projetos.

Não é apenas com a rede que o Kubernetes faz isso; recentemente, comentei sobre [como usar volumes CEPH no Kubernetes]({{< ref "blog/ceph-csi-on-kubernetes.md" >}}) usando plugins que implementam a [Container Storage Interface](https://github.com/container-storage-interface/spec). E acredito que praticamente todas as pessoas interessads em Kubernetes ouviram a notícia de que o [suporte a Docker pelo kubelet é considerado "deprecated" com o lançamento da versão 1.20](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation). Isso aconteceu basicamente porque Kubernetes implementa a [Container Runtime Interface](https://github.com/kubernetes/cri-api), que desacopla do Kubernetes a função de gerenciamento de containers no sistema operacional e outros detalhes.

Como o Docker não implementa essa interface, ele atua como um intermediário cujo papel é inserir overhead entre a integração do Kubernetes com o [containerd](https://github.com/containerd/containerd), que é quem efetivamente executa as operações de containers. E se o containerd implementa CRI, por que não falar diretamente com ele, cortando o atravessador?

Então, assim como usa **CRI** e **CSI**, para redes, o Kubernetes aplica a [**Container Network Interface** (**CNI**)](https://github.com/containernetworking/cni).

Não é nenhuma surpresa que o projeto faça isso. Não faz sentido implementar código de operação de containers, ou de storages, ou mesmo de redes dentro do próprio Kubernetes (embora ainda haja código desse tipo!) se existem outros projetos que executam estas tarefas de maneira eficiente. Basta apenas que haja interesse, por parte dos demais projetos, de implementar esse conjunto.

E há. **Bastante interesse**, diga-se de passagem - são mais de 20 projetos e empresas diferentes oferecendo todo tipo de produto (com todo tipo de preço também).

> **Espaço para ironia**: quando as opções estão limitadas a duas alternativas, normalmente é bem mais fácil escolher. Por outro lado, quanto tempo você já perdeu tentando escolher um filme para assistir no Netflix? 

Se eu não sei absolutamente **nada** sobre Kubernetes, e sendo o CNI crítico para o funcionamento - não é uma decisão delegável para mais tarde - como faço para escolher? 

Vamos olhar como é o processo de "escolha" dos administradores Kubernetes.

## CNI em Cloud Providers

Usuários dos grandes Cloud Providers também podem escolher usar entre os diversos CNIs existentes. Mas normalmente não há **qualquer razão** para fazer isso, pois as suas ofertas de clusters 'managed' vêm com CNIs próprios.

Se estou usando meu cluster EKS e a AWS oferece um [plugin CNI](https://github.com/aws/amazon-vpc-cni-k8s) que é nativamente compatível com seus VPCs, permitindo que Pods assumam o mesmo endereçamento IP das subnets criadas nas várias Availability Zones, por que você iria querer usar outra coisa? Mesmo se você quiser que o endereçamento dos Pods use outra rede, [basta configurar o CNI AWS](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html) para isso. Até [existe a possibilidade](https://docs.aws.amazon.com/eks/latest/userguide/alternate-cni-plugins.html), ainda que a AWS se exima do suporte. Mas normalmente não há uma boa justificativa para isso.

> **Nota**: Administradores avançados, que preferem, por exemplo, manter controle sobre seus Control Planes, podem optar por levantar clusters Kubernetes em instâncias EC2. Mas aí são administradores **avançados**, que provavelmente não estão lendo este texto e não tem qualquer dificuldade em escolher seu CNI.

**Resumindo**: em Cloud Providers, a escolha do CNI é uma "*non-issue*".

## CNIs "Enterprise"

Diversas grandes empresas do mercado já perceberam que o interesse em Kubernetes é crescente e desenvolveram ofertas próprias de CNIs proprietários compatíveis com seus produtos, como o [Contiv para CISCO ACI](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/aci/apic/sw/kb/b_Kubernetes_Integration_with_ACI.html) ou o [VMWare NSX-T CNI](https://docs.vmware.com/en/VMware-NSX-T-Data-Center/2.2/com.vmware.nsxt.ncp_kubernetes.doc/GUID-6AFA724E-BB62-4693-B95C-321E8DDEA7E1.html), normalmente com seu uso exclusivamente atrelado a existência de equipamentos ou licenças de software no seu ambiente. 

Grandes empresas com contratos milionários têm suporte "enterprise" e a possibilidade de fazer uso de Kubernetes on-prem usando componentes específicos que se integram com alguns de seus sistemas pagos. Em situação de pouca experiência, talvez valha a pena investir nisso. **Não é um caminho que eu seguiria**, mas eu sou um nerd #resistenciaOPs que trabalha com Linux há 22 anos.

Eu não posso oferecer qualquer opinião sobre eles, nem mesmo ruim sobre CNIs proprietários. Embora seja um grande defensor do uso de software livre em todos os lugares, entendo que, por exemplo, um ponto positivo para seu uso é que pode ser uma forma de angariar "aliados" (na forma de administradores CISCO ou VMWare, por exemplo) para participar de projetos loucos implantando Kubernetes on-prem, dividindo trabalho que, de outra forma, será feito somente por você.

**Resumindo**: mesmo se a sua 'nuvem privada Kubernetes' não vier pronta do fornecedor, grandes empresas podem optar por integrar Kubernetes on-prem com softwares proprietários fornecidos por alguns desses.

## CNIs para Kubernetes on premises

Infelizmente, não temos aqui um meio termo de dificuldade. Pulamos diretamente de "*nem precisei pensar sobre isso*" para "*leia TUDO sobre TODOS para escolher o que melhor se aplica para você*".

A oferta é enorme e, a cada ano, alguma equipe de heróis geniais de empresas obscuras saem com uma alternativa que seja mais interessante para seus objetivos. E aí está a beleza do universo software livre: se você é um herói genial, é bem provável que ache, em meio aos bilhões de habitantes da terra, outros heróis com finalidades afins para contribuir para o seu projeto.

Vou fazer um leve 'gossiping' dos projetos a que fui exposto ao longo dos últimos quatro anos.

### Os 'big 4'

Basicamente 4 CNis são as opções mais recorrentemente faladas no universo Kubernetes. São eles:

#### Calico
O projeto [Calico](https://www.projectcalico.org/) de longe é a opção mais comum em instalações, principalmente por ser um dos 'earlier adopters' do recurso de *NetworkPolicies* no Kubernetes.

Ainda hoje a recomendação padrão para aplicação de *Networkpolicies* em clusters EKS na AWS é feita por meio do deploy de componente específico do Calico, citado na [documentação oficial](https://docs.aws.amazon.com/eks/latest/userguide/calico.html).

O Azure AKS oferece suporte a Calico nativamente, e mantém uma [tabela comparativa de features](https://docs.microsoft.com/pt-br/azure/aks/use-network-policies#differences-between-azure-and-calico-policies-and-their-capabilities).

Há a oferta de uma versão Enterprise do Calico pela [Tigera](https://www.tigera.io/), que é uma empresa praticamente dedicada integralmente a conectividade e segurança de rede para microsserviços. A versão enterprise inclui algumas funcionalidades que tendem a fazer brilhar os olhos dos profissionais de segurança de rede, mas que não tornam a versão "community" incompleta.

Uma das principais barreiras para o uso do Calico era seu site que remetia à decada de 90, documentação confusa e pouquíssimo material de treinamento; todos esses aspectos tem sido "atacados" pela Tigera e estão muito melhores, contando até com um [Academy Tigera Certified Calico Operator: Level 1](https://academy.tigera.io/course/certified-calico-operator-level-1/) gratuito!

#### Weave

O [Weave net](https://www.weave.works/docs/net/latest/overview/) também foi um projeto existente à época da minha implantação original de Kubernetes. Este CNI foi criado pela Weave É uma empresa muito popular entre usuários Kubernetes, principalmente por ter desenvolvido o hoje oficialmente adotado pela AWS [eksctl](https://www.weave.works/oss/eksctl/).

A Weave não tem um foco exclusivo em networking, muito pelo contrário. Atua em diversas frentes de desenvolvimento com produtos direcionados ao deploy e observabilidade de aplicações, além de ter criado a ideia de 'GitOps' e disponibilizado programas como o Flux, Flagger  e Cortex. O 'appeal' de manter seu CNI Weave Net se dá pela integração com seu Weave Kubernetes Platform.

### Cilium

O CNI [Cilium](https://cilium.io/) sempre foi a principal (e por muito tempo, talvez a única) opção para aqueles interessados pelo uso de eBPF em especial por quem quer se livrar dos fardos que são o iptables e o kube-proxy. 

Embora a substituição da confusão que é iptables em clusters Kubernetes não necessariamente se traduza em melhora de performance, a evolução natural da tecnologia eventualmente vai "chegar lá.

(O Calico suporte eBPF desde a versão 3.13, salvo engano).

Existe suporte enterprise para Cilium por meio da empresa [**Isovalent**](https://isovalent.com/product).

### Flannel / Canal

O quarto, [Flannel](https://github.com/coreos/flannel/), pela sua simplicidade, acabou se tornando o "CNI padrão" para deploy de clusters para testes. É usado, ainda hoje, por praticamente todos os clusters nos exames CKA/CKAD. 

**Nota**: [Canal](https://github.com/projectcalico/canal) é o nome normalmente usado para o uso combinado do Flannel com os recursos para Networkpolicy do Calico; não é um CNI diferente.

Muitos usuários mais experientes recomendam usar o Flannel para iniciar estudos com Kubernetes por não haver tantas "partes móveis", mas é possível que a escassez de documentação e o estado atual do projeto não encorajem seu uso sequer para isso (embora parece que esteja havendo uma retomada).

Ainda que seja amplamente recomendado para aprendizado e para instalações minimalistas para laboratório e angariar conhecimento em Kubernetes, acredito que absolutamente ninguém recomende fazer qualquer uso sério de Flannel em ambientes que tenham a mínima chance de serem 'graduados' à produção.

### Outros

Aqui vou listar outros três que já tive algum contato, apenas para referência:

* [Kube router](https://www.kube-router.io/): um projeto que tivemos a oportunidade de usar um de seus 'pedaço' quando precisamos de uma solução que substituisse o kube-proxy;
* [Contiv](https://contivpp.io/): desenvolvido pela Cisco;
* [Antrea](https://github.com/vmware-tanzu/antrea): desenvolvido pela VMWare;

# Que CNI escolho e por que Calico?

Essa é a primeira grande questão de quem quer instalar Kubernetes em seu datacenter, e praticamente impossível de responder caso você se encontre completamente desassistido de consultoria mais experiente.

Agora deveria ser a hora em que eu puxo tabelas incríveis e demonstro qual o melhor CNI. Infelizmente eu não posso fazer isso, por uma simples razão: 99% da minha experiência com Kubernetes aconteceu única e exclusivamente com o uso do **Calico** como CNI.

Não há como fazer comparações sensatas sem conhecer como as demais tecnologias se comportam "no campo de batalha".

Você pode até consultar bons artigos de Internet, como a fantástica (e bastante recente) comparação de performance entre alguns dos CNIs mais populares:

* https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-updated-august-2020-6e1b757b9e49

Recomendo a leitura integral desse artigo, até para entender 'por que Calico?'. Realmente é uma boa opção.

Você pode (deve!) fazer perguntas técnicas relevantes e averiguar como cada CNI se comporta com relação a:

* VXLAN/Geneve ou Layer 3 com BGP?
* eBPF ou iptables/ipset e ipvs?
* Suporte a IPv6?
* criptografia?
* data backplane dedicado ou integrado ao Kubernetes?
* Windows?

Ou sobre preço:

* Você é "liso" ou pode contratar suporte?
* Tenho um CNI proprietário disponível que integra com minha infra?
* Preciso ter contrato de suporte por questão de compliance?

Mas, por fim, normalmente os principais requisitos para a escolha não são tão técnicos, e sim:

* Qual a documentação mais simples (e mais clara);
* Qual o mais usado - para que eu tenha mais chance de ter ajuda quando encontrar problemas;

Que **hoje** provavelmente significa que você quer usar **Calico** no seu cluster.

# Escolhi. Mas e se me arrepender?

Não importa o que você escolher, entenda que o CNI não é necessariamente é um casamento, mas mudanças **serão dolorosas** e irão demandar parada integral do cluster para ajustes, possivelmente incluindo um '*kubectl delete pods -A*'. Provavelmente a alternativa mais viável será subir um novo cluster com o novo CNI e migrar os workloads para posterior desativação do anterior.

Você pode ler a experiência de um Kubernetes admin que precisou migrar de [Weave para Calico](https://medium.com/@andresperezl/weave-to-calico-7c69677222d0) por problemas irrecuperáveis com seu Weave CNI.

Alheio à sua escolha, tenha uma coisa em mente: você **precisa** dedicar uma quantidade substancial de tempo para entender adequadamente o projeto à sua escolha. Não é possível apenas se dedicar a estudar Kubernetes e achar que o plugin CNI 'just-works' como se fosse algo plug-and-play. Se possível, aloque gente com forte background de rede para entender detalhes da implantação para mapear possíveis bloqueios futuros e dificuldades que virão.

# Bônus: consultoria de implantação

Até agora, apresentei conteúdos e conselhos genéricos e filosóficos sobre Kubernetes e CNI. Estou extremamente insatisfeito com o texto, então hora de algo mais relevante. Uma "consultoria gratuita" para pessoas interessadas em colocar a faca nos dentes e nadar contra a maré implantando Kubernetes on-prem.

Parei de falar com gerentes agora; é a vez dos #Ops.

## Kubernetes realmente abstrai a rede?

Claro que não. Tem um limite até onde essa abstração pode chegar.

Para criar clusters com **kubeadm**, existem dois parâmetros que permitem configurar as duas redes relevantes para o Kubernetes:

* A rede de **Pods**: configurada pelo parâmetro --pod-network-cidr;
* A rede de **Services**: diretiva --service-cidr, valor default "10.96.0.0/12";

> Se você não sabe **absolutamente nada** sobre Kubernetes, um resumo absurdamente conciso sobre:
> * os containers da sua aplicação receberão endereços IP da rede de **Pods**;
> * você provavelmente vai rodar múltiplas instâncias de cada tipo de container (i.e., 2 pods para frontend, 5 para backend, 3 para filas) 
> * o **Kubernetes Service** é um IP virtual associado a um nome DNS que você usará para fazer um 'balanceamento interno' entre containers do mesmo tipo.
> 
> Logo, uma aplicação com 2 frontend, 5 backend e 3 filas usarão 10 endereços da rede de Pods e 3 endereços da rede de Services.

O próprio [proceso de instalação do Calico](https://docs.projectcalico.org/getting-started/kubernetes/quickstart) recomenda que você execute o kubeadm especificando a rede de pods:

> Initialize the master using the following command:
>
> ```sudo kubeadm init --pod-network-cidr=192.168.0.0/16```

**Muito importante** (e também meio óbvio): certifique-se que não '*overlapping*' entre a rede de pods e a rede serviços.

Para onde vão essas configurações? Diretamente para os vários elementos do cluster Kubernetes.

* api-server:
```
# cat /etc/kubernetes/manifests/kube-apiserver.yaml  | fgrep service-cluster
    - --service-cluster-ip-range=10.96.0.0/12
``` 

* controller:
```
# cat /etc/kubernetes/manifests/kube-controller-manager.yaml | egrep 'cluster-cidr|service-cluster'
    - --cluster-cidr=172.16.0.0/16
    - --service-cluster-ip-range=10.96.0.0/12
```
* kube-proxy:
```
# kubectl get configmap -n kube-system  kube-proxy  -o yaml | grep clusterCIDR
    clusterCIDR: 172.16.0.0/16
```

Isso quer dizer que modificar esses endereços após o cluster entrar em estado operacional **também não é trivial e sem impactos**. 

O que quer dizer que a segunda decisão mais importante para a qual você não está nem um pouco preparado a escolha das redes a serem usadas pelo Kubernetes.

## Que redes usar

Aqui cabem algumas decisões críticas sobre como você vai querer o seu cluster e que, provavelmente, precisam ser tomadas bem antes de você aplicar o seu primeiro kubectl.

### 1. Seu cluster será uma entidade **isolada** ou **integrada** ao seu ambiente corporativo?

A primeira decisão crítica. Se o uso de Kubernetes pretende ser auto contido, você pode (e deve!) usar alguma rede privada completamente diferente da usada pela sua empresa e assim não ter que se preocupar com o endereçamento dos IPs de pods: basta alocar uma rede grande o suficiente (como a 172.16.0.0/16 listada acima) como sua **rede de Pods** e uma outra rede também grande (10.96.0.0/12) para a sua **rede de Services**.

**Minha recomendação**, obviamente, é a de **isolar** os **Pods** do seu cluster do resto da sua empresa. Por uma razão bem simples: provavelmente o 'resto' não é Cloud Native, segue o modelo de trabalho tradicional, burocrático, pouco dinâmico e especialmente pouco inteligente. Se você conseguir implantar o cluster Kubernetes como o princípio da versão 2.0 da sua empresa, **todos** vão ganhar. Aplicações novas usam o modelo novo; aplicações 'legadas' ficam presas do lado oitentista.

Mas eu sei que isso é quase impossível.

### 2. Para um cluster integrado, o Pod precisará assumir um endereço IP válido na sua rede corporativa?

Esse é o pior cenário possível para a implantação de um cluster Kubernetes.

Normalmente redes corporativas tem sua alocação de endereçamento de Intranet pensada de uma forma, e clusters Kubernetes precisam de grandes pools de endereços para serem viáveis, em particular se você vai usar o CNI Calico.

Não há uma boa razão para alocar todo um range gigante para o cluster Kubernetes com o objetivo de tornar os IPs dos Pods acessíveis, pois Pods raramente são acessados diretamente. Se você precisa acessar um sistema no Kubernetes, provavelmente vai fazer uso de *Services*, *Nodeports* ou *Ingress Controllers* para serviços HTTP.

Inclusive, isso sequer é um requisito para que *Pods* no Kubernetes acessem serviços hospedados fora do cluster: o normal para um *IPPool Calico* é estar configurado com a diretiva *natOutgoing=true* para que toda comunicação direcionada para fora saia com o endereço IP do nó em que o Pod está hospedado.

Portanto, **minha recomendação** explícita é: **não**, não aloque IPs válidos da sua rede corporativa para os Pods do cluster Kubernetes.

Existe uma única situação em que isso é necessário: se for necessário para *Compliance* e *Auditoria*. 

Imagine o seguinte cenário de mais puro horror: seu cluster executa três sistemas, e cada um desses sistemas acessa um banco de dados Oracle próprio que é hospedado na parte 'tradicional' da sua empresa. Houve um acesso indevido, que foi rastreado para o cluster Kubernetes. Como saber **que Pod** realizou o acesso, já que os três sistemas rodam pods nos mesmos nós (e tudo que você tem é o endereço do nó Kubernetes)?

A única maneira de viabilizar isso é com endereçamento válido na sua Intranet e com sistemas de auditoria que registrem que Pod tinha que IP durante que tempo.

Ambas as coisas (tanto a parte da integração da rede do Kubernetes com a Intranet quanto a auditoria de Pods) são verdadeiros horrores para serem viabilizadas. Eu posso falar isso porque, infelizmente, precisei fazer as duas coisas.

Configurar o Calico para integrar com roteadores e, assim, divulgar rotas, por si só não é um problema, (embora tenha sido necessário [um PR](https://github.com/projectcalico/confd/pull/339) para viabilizar o que precisamos). Normalmente o problema é a integração com as equipes responsáveis por roteadores e firewalls que trabalham no modelo "tradicional". 

Já a auditoria de que IP estava com que Pod em que horário não existe uma solução adequada no universo do software livre sem subscrição.

### 3. Quantos Pods você pretende executar no seu cluster?

Um outro ponto positivo para isolar (ou pelo menos usar natOutgoing=True) é que você pode instanciar dezenas de clusters entregando os mesmos endereços IP das mesmas redes sem maiores problemas. Se você usar 1000 clusters e puder usar, em todos eles, a rede 172.16.0.0/16 , você não precisa se preocupar com esgotamento de endereços IP.

Se você pretende criar diversos clusters Kubernetes que conversem entre si em nível de comunicação Pod a Pod, isso pode vir a se tornar um problema para a sua alocação interna de endereços IP.

Agora, se você caiu no pior cenário possível que é precisar alocar endereços IP válidos na Intranet para seus Pods, vou começar com as más notícias.

Weave, Flannel e outros plugins CNI normalmente operam em Layer 2 usando técnicas de encapsulamento como VXLAN ou Geneve. O Calico, por padrão, opera em modo Layer 3 usando BGP. Cada nó de workload se torna um 'roteador BGP' executando um [Bird](https://bird.network.cz/) que é controlado pelo componente [Calico node](https://github.com/projectcalico/node). Isso inclusive é listado como 'vantagem', pois fica fácil entender e depurar problemas de comunicação, já que as rotas estão acessíveis.

Mas isso faz com que o Calico seja **extremamente perdulário** com as alocações de IP. O padrão dos IPPools Calico é segmentar a rede que você configurou usando a diretiva *blockSize=26* (para ipv4, para IPv6 é 122). Isto é, ele irá pegar a rede, fatiar em pedaços /26, e sair distribuindo entre hosts. 

Suponha que, para seu primeiro cluster, os controladores dos IPs válidos de rede tenham fornecido uma rede 192.168.69.0/24 para que você use como endereçamento de Pods. Se você não souber deste parâmetro, o Calico irá segmentar a rede em quatro redes /26 e distribuir para quatro nós. 

E depois? Bem, o Calico começará a criar rotas /32 para os endereços dos Pods e o número de rotas (que deveria ser apenas quatro) explodirá tendendo a chegar em uma rota para cada Pod. Na [documentação oficial](https://docs.projectcalico.org/reference/resources/ippool) há recomendação expressa para que você dimensione o *blockSize* para que cada nó receba ao menos uma subrede própria. Essa explosão na tabela de rotas pode gerar algo entre completo descontentamento por parte dos administradores dos roteadores que estão integrando com vocÊ até o **não funcionamento e travamento de ativos** por atingir algum limite do equipamento. Em uma rede /24, isso possivelmente não será um problema, mas eu consegui chegar nesse problema usando uma rede /19. 8000 rotas talvez seja o suficiente para travar alguns equipamentos.

Uma outra questão é a possível necessidade de limitar endereçamento de Pods para namespaces. Isso pode ser necessário, por exemplo, se o seu administrador Postgres estiver reticente quanto a liberar, no pg_hba, toda a rede 172.16.0.0/16. Eu honestamnete ignoraria tais administradores, mas, se isso não for opção, é possível dedicar *IPpools* a namespaces ou mesmo *Pods*. Isso é [descrito na documentação do Calico](https://docs.projectcalico.org/networking/legacy-firewalls). Se por acaso você precisar dessa funcionalidade, será necessário computar ainda mais volume para a fração da rede a ser alocada para o Kubernetes.

---

A necessidade de usar IPs válidos da sua Intranet para endereçamento de Pods causa uma explosão de complexidade para a qual a maioria das pessoas não está preprada ao começar a usar Kubernetes sem algum apoio técnico mais experiente. Por isso, se você quiser contratar uma consultoria avançada para resolver os seus problemas e quiser contratar um especialista você... **Não** pode me contratar, pois sou funcionário público federal e trabalho em uma empresa de TI do governo. Foi mal! 

Mas bater papo eu posso à vontade! Se ainda tem alguma dúvida, especialmente se for funcionário público de alguma empresa que não qualquer tipo de possibilidade de contratar verba de consultoria, manda uma mensagem no Linkedin que a gente troca uma idea! Estou precisando de mais sugestão de pautas para abordar aqui no [devsres.com](https://devsres.com/marcelo)!

Depois de bater papo comigo **e** ainda precisar de consultores especializados, é possível que eu te indique ir conversar com os caras da [Getup Cloud](https://getup.io/), mas provavelmente isso só vai acontecer se eles me chamarem para fazer mais uma edição do Kubicast.

