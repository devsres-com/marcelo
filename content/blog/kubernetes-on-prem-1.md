+++
title = "Kubernetes on-premisses - parte 1"
description = "Relato histórico das decisões tomadas na implantação de um cluster Kubernetes on-premises."
author = "Marcelo Andrade"
date = "2020-12-09"
tags = ["kubernetes", "on-prem", "passado"]
categories = ["kubernetes", "on-prem", "passado"]
[[images]]
  src = "img/reusable/jenga.png"
  alt = "jenga"
  stretch = ""
+++

Recentemente, participei do [Kubicast](https://blog.getupcloud.com/kubicast-51-1df6d5278623), em que pude desabafar um pouco com o [João Brito](https://twitter.com/juniorjbn) da [Getup](https://getup.io/) sobre como é criar e um cluster Kubernetes em um ambiente tecnologicamente inóspito e oposto ao universo super estável, escalável e redundante dos cloud providers.

Recebi algumas mensagens de alguns poucos valentes que precisam trilhar esse caminho. Mas, antes, vou ressaltar uma coisa importante: neste primeiro momento, eu vou abordar a **história** do que foi feito e compartilhar a **experiência**. Provavelmente existem alternativas melhores para um começo menos pedregoso; eu vou **tentar** listá-las à medida que for abordando os assuntos, mas não é um requisito no momento. 

# Ponto de partida: infraestrutura

Não queria perder tempo discutindo infraestrutura, mas, querendo ou não, entender o tamanho das suas limitações é importante para planejar o que se quer fazer.

O que estava disponível para conduzir os meus trabalhos eram:

* Máquinas físicas ("bare metal") para atuar como nós para as cargas ("workers");
* VMWare VSphere com um datacenter dedicado e cluster datastores pré configurados para a criação de máquinas virtuais para todo o resto.

Não posso compartilhar algo que não fiz; portanto, não vou indicar soluções de virtualização ou orquestração de máquinas virtuais que você pode implantar em máquinas físicas. Mas isso é apenas por pura **falta de experiência**.

Além do que você "tem", você precisa avaliar o que você não tem:

Eu, a princípio, **não tinha disponível**:

* Balanceadores de carga;
* Acesso ao firewall;
* Acesso ao roteador;
* Acesso aos switches para configuração dinâmica das portas (para coisas tipo VLAN);

Dependendo do tamanho da empresa e da hostilidade das equipes diante das tecnologias que você está implantando, você pode ter diversos problemas de integração do seu cluster Kubernetes ao resto da empresa.

O que eu julgo mais importante nesta etapa é usar o máximo da automação pré-existente. Infelizmente, para mim, não havia nenhuma, e quantidades extenuantes de procedimentos manuais foram executados para garantir atendimento de prazos, o que cobra a conta mais tarde.

Hoje usamos uma combinação de [Terraform](https://www.terraform.io/) com instalação via PXE usando [Matchbox](https://github.com/poseidon/matchbox) de maneira muito semelhante ao disposto pelo projeto [Typhoon](https://github.com/poseidon/typhoon)

# Definições do cluster

Existem algumas perguntas que você deve se fazer antes de construir o cluster. Podemos englobar todas elas em uma discussão sobre "topologia", embora seja muito mais que isso.

Vou elencar uma lista (não exaustiva) de algumas das decisões mais importantes que pretendo trabalhar nesta série de posts:

* Topologia do Controlplane;
* Como fazer o deploy do cluster;
* Como fazer o deploy dos 'penduricalhos' (outros utilitários a serem usados no cluster).
* Sistema operacional (sim, por incrível que pareça!);
* Modelo de redes do cluster (i.e., qual o CNI e que recursos avançados deste usar);
* Modelo de tráfego de entrada - um dos maiores desafios para os clusters on-premises;

# Topologia do cluster - Controlplane:

Esta definição prévia é importante porque o 'salto' de um modelo mono-master para outro multi-master acaba sendo mais trabalhoso que simplesmente implantar um multi-master de primeira.

Para fazer testes breves e criar familiaridade com a tecnologia, vale a pena testar o modelo mais simples. Mas muito cuidado ao implantar 'pilotos' e 'provas de conceito' simplórias demais, especialmente se sua organização tem a mania de transformar estes ambientes em produção por decreto.

O que se encontra de mais comum em topologia de Controlplane é uma organização 'stacked' em que são criados 3 máquinas que combinam Master com ETCDs e uma solução de loadbalancer externa.

Optamos por:
* 2 VMs para loadbalancers dedicados para os Kubernetes Masters;
* 2 VMs para Kubernetes masters;
* 5 VMs de ETCDs dedicados para atender ao cluster Kubernetes;
* 3 VMs de ETCDs adicionais dedicados para o Calico;

Imagino que não seja necessário mencionar, mas um 'Kubernetes Master' é a máquina que irá executar (no mínimo) os três principais componentes do cluster Kubernetes: o **Apiserver** (em que múltiplos podem estar ativos no mesmo momento) e os par **Scheduler**/**Controller**, os quais apenas um por cluster atua como ativo, funcionando a base de *leases* para identificar os líderes e assim garantir o *failover*.

## Master loadbalancers

Em um ambiente multi-master, você **precisa** de uma maneira de distribuir a carga entre eles que **não seja DNS round robin**. Algumas empresas contam com hardware dedicado para esta finalidade, que faz healthcheck e coisas elegantes do tipo. 

Por estarem fora do meu 'alcance', optamos por subir duas VMs com [**Nginx**](https://nginx.org/) atuando como proxy para os Apiservers, usando [**Keepalived**](https://www.keepalived.org/) para implementar VRRP para o IP virtual associado ao nome do host.

**Nota**: Interessante notar que projetos como o [**Kubespray**](https://kubespray.io) fazem o deploy automático de um [*Daemonset* com Nginx](https://kubespray.io/#/docs/ha-mode) que atua como um proxy interno da comunicação entre os nós e o Apiserver. Se o acesso aos Apiservers por produtos **externos ao cluster** não for um requisito, de repente nem seria necessário subir essas máquinas. Não é o nosso caso, mas pode ser o seu! Ainda que não use Kubespray, pode usar a ideia como inspiração.

## Masters não 'stacked' nos ETCDs

A documentação do Kubernetes descreve, em sua sessão de *High Availability*, duas topologias básicas:

* [Stacked control plane + ETCD nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#stacked-control-plane-and-etcd-nodes);
* [External ETCD nodes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#external-etcd-nodes)

A esmagadora maioria das pessoas opta por uma topologia 'stacked' em que tanto o Kubernetes Master quanto o ETCD são instalados na mesma máquina. É uma topologia óbvia e conveniente. Optamos por não o fazer, pois preferimos usar um cluster ETCD de 5 membros em vez de 3.

Por que não subir 5 masters, então? 

Porque as máquinas ETCD praticamente não demandam recursos. Um nó ETCD pode rodar, por exemplo, com 4GB de RAM e 2 processadores. Já um Kubernetes Master demanda uma quantidade substancialmente maior de recursos. Como os recursos virtuais eram relativamente exíguos à época, optamos por apenas duas máquinas como Master Kubernetes enquanto criávamos mais máquinas ETCD com "o que sobrou".

Obviamente, você deve dimensionar as máquinas 'de acordo com a necessidade', mas qual um bom ponto de partida? Existia uma tabela interessante na documentação do Kubernetes que fazia uma correlação entre os tipos de instância dos Cloud Providers que você deveria usar de acordo com o tamanho do seu cluster. Ela foi removida na versão 1.19, mas [ainda está disponível para as versões anteriores](https://v1-17.docs.kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components) (ou se você selecionar o idioma japonês!), e eu acho legal para ter uma ideia:

> On GCE/Google Kubernetes Engine, and AWS, kube-up automatically configures the proper VM size for your master depending on the number of nodes in your cluster. On other providers, you will need to configure it manually. For reference, the sizes we use on GCE are

* 1-5 nodes: n1-standard-1
* 6-10 nodes: n1-standard-2
* 11-100 nodes: n1-standard-4
* 101-250 nodes: n1-standard-8
* 251-500 nodes: n1-standard-16
* more than 500 nodes: n1-standard-32

> And the sizes we use on AWS are

* 1-5 nodes: m3.medium
* 6-10 nodes: m3.large
* 11-100 nodes: m3.xlarge
* 101-250 nodes: m3.2xlarge
* 251-500 nodes: c4.4xlarge
* more than 500 nodes: c4.8xlarge

É importante levar em consideração o que ele chama de um nó, que seria uma máquina com não mais que 100 pods - sua realidade pode ser drasticamente diferente do esperado.

## ETCDs do Kubernetes

O tamanho mínimo aceitável para executar ETCD que ofereça alguma tolerância a falhas corresponde a **três** nós. Porque **cinco** então?

Um cluster de cinco membros permitirá a falha concorrente de **dois** membros. Em Cloud Providers como a AWS, você provavelmente pode sobreviver com um cluster de três máquinas espalhadas em três *Availability Zones* diferentes; dificilmente uma Region irá experimentar uma falha em duas AZs ao mesmo tempo.

Em nosso ambiente, devido a problemas de performance ao usar discos virtuais com o cluster datastore oferecido via SAN, optamos por usar discos NVME locais do host VMWare em que a máquina ETCD está hospedada. Esta dependência do host faz com que valha a pena gastar recursos subindo duas máquinas a mais para essa finalidade, pois não há a flexibilidade de live migration ou outros recursos do tipo.

Eu pretendo abordar ETCD em mais detalhes em outro post, mas aqui vale a ressalva: o cluster ETCD é **mais importante que qualquer outra coisa* no ambiente (talvez, exceto, do que você usa para armazenar os dados das aplicações). 

Executar um cluster on-premise significa arcar com plena administração desta solução, que não é particularmente trivial: é necessário cuidado extremo com os backups (e seus testes), automação afiadíssima para reconstrução do cluster com velocidade em caso de tragédias e monitoração bastante exagerada para tentar identificar a proximidade dos possíveis gargalos de performance antes que sintomas se abatam sobre o cluster.

## ETCDs do Calico
O uso de ETCDs para o Calico hoje é opcional, já que o uso do [próprio Kubernetes](https://docs.projectcalico.org/getting-started/kubernetes/hardway/the-calico-datastore#using-kubernetes-as-the-datastore) como datastore já é considerado maduro e estável o suficiente. Optar por Kubernetes Datastore normalmente demanda implantar um componente adicional do Calico ([Typha](https://docs.projectcalico.org/getting-started/kubernetes/hardway/install-typha)) para desonerar os apiserver. Embora a recomendação seja para 'clusters grandes', melhor já ir se acostumando com ele do que ter que implementar emergencialmente porque a performance do cluster está degradada.

Uma vantagem de ter um cluster dedicado para o Calico é que, se você eventualmente quiser fazer um 'offload' da carga do ETCD do Kubernetes removendo, por exemplo, os Event objects como recomendado [aqui](https://kubernetes.io/docs/setup/best-practices/cluster-large/#etcd-storage), as máquinas já estão prontas!

Uma desvantagem de usar ETCD como backend para o Calico é o fato de que todos os nós precisarão de comunicação estável com o cluster ETCD. Em algumas literaturas, uma recomendação comum de segurança é a de sugerir que as máquinas de ETCD estejam em uma rede isolada das demais máquinas, oferecendo acesso apenas ao Kubernetes Apiserver. Dependendo do tipo de rede que você precisa lidar e a quantidade de intermediários atuando entre os barramentos, isso dificilmente é uma opção neste caso.



