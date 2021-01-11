+++
title = "Kubernetes on-premises - parte 4"
description = "Como criar clusters Kubernetes"
author = "Marcelo Andrade"
date = "2021-01-11"
tags = ["kubernetes", "on-prem", "passado"]
categories = ["kubernetes", "on-prem", "passado"]
[[images]]
  src = "img/reusable/minecraft.png"
  alt = "Minecraft"
  stretch = ""
+++

Outros capítulos da série **Kubernetes on-premises**:

* [Kubernetes on-prem - parte 1]({{< ref "blog/kubernetes-on-prem-1.md" >}}): Introdução.
* [Kubernetes on-prem - parte 2]({{< ref "blog/kubernetes-on-prem-2.md" >}}): Redes em Kubernetes - CNI e Calico.
* [Kubernetes on-prem - parte 3]({{< ref "blog/kubernetes-on-prem-3.md" >}}): Tráfego de entrada em clusters Kubernetes


# Como criar clusters Kubernetes

**Disclaimer**: provavelmente este é o capítulo mais dispensável da série porque é o que eu tenho menos coisas relevantes para compartilhar. De qualquer forma, ele precisava ser escrito!

Até agora, abordei diversas considerações relevantes que devem ser feitas **antes** de se instalar um cluster Kubernetes - e, em particular, pontos que particularmente são mais dolorosos para quem não está operando em Cloud Providers.

Mas finalmente, vem a pergunta: **como instalar clusters Kubernetes**? Qual é o **jeito certo**?

Quando recebi essa missão pela primeira vez, eu não estava sequer remotamente preparado para isso. Combine o fato de que a instalação deveria ser realizada em um sistema ainda mais hostil que o normal como o CoreOS Linux (eventualmente renomeado para Container Linux e posteriormente absorvido pela Red Hat) tornou tudo incrivelmente mais difícil.

A primeira coisa que vem em mente quando pergunto isso para as pessoas é, claro, **kubeadm**.

E se eu te contar que, quando começamos, sequer existia **kubeadm**. [Este post](https://kubernetes.io/blog/2016/09/how-we-made-kubernetes-easy-to-install/) descreve o lançamento da primeira versão alpha do Kubeadm. Começamos a pensar na implantação em agosto de 2016.

Portanto, foi necessário para nós, desde o início, adotar algo muito parecido com o [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way), mas usando o Calico CNI.

## Nosso processo até hoje: userdatas

Devido à mais completa falta de oferta, foi necessário desenvolvermos nossa própria estrutura de criação de clusters. Ela foi modificada várias vezes ao longo dos anos, mas que segue a mesma ideia.

Nossos clusters são integralmente lançados a partir de arquivos tipo **userdata** para sistemas operacionais orientados a containers como Flatcar Linux até hoje, de maneira não interativa. 

Como o recurso de Certificate Signing do Control Plane do Kubernetes não estava disponível, fazemos uso de uma Autoridade Certificadora interna com API que gera certificados sob demanda. A comunicação com o API Server exige que a comunicação seja feita por meio do uso de certificados dessa CA, assim como a comunicação dos API Servers está restrita a clientes com certificados válidos.

Para falar a verdade, uma vez que o processo de saber o que e como deve ser instalado é mapeado, não há grandes mistérios no processo de instalação de clusters Kubernetes. O único grande porém introduzido é a necessidade de gestão e cuidado de certificados digitais para as máquinas. Se for este o caminho desejado para sua instalação on-prem, basicamente as únicas preocupações são:

* Como criar e manter CAs seguras;
* Como distribuir os certificados gerados para os nós Kubernetes de maneira segura;
* Garantir que os certificados nunca expirem, monitorando o vencimento e renovando automaticamente;

Faz parte do segundo item, mas devo enfatizar aqui o único problema real da nossa instalação, que até hoje eu não resolvi que é:
* Como fazer o certificado usado para validar os *ServiceAccount tokens* chegar nos demais API Servers;

### Processo de autenticação nos API Servers 

Para quem não é muito familizarizado com o conceito de **ServiceAccounts**, basta entender que são entidades Kubernetes que disparam a criação de *Tokens* que podem ser usados para acessar os API Servers. 

É possível habilitar vários métodos de autenticação em um cluster Kubernetes, mas basicamente os dois comumente em clusters fora da nuvem são:

* Certificados;
* Tokens de Service Accounts.

Os tokens de Service Accounts são criados para viabilizar uma maneira das aplicações em execução nos Clusters acessarem os API Servers sem grandes dificuldades. Eles são montados como volumes em todos os Pods, e se a aplicação for 'linkada' com as libs clientes de Kubernetes, sequer precisará de configuração para ser usada - elas 'se descobrem' usando o modo '*in-cluster*' de configuração.

Claro que, por segurança, os tokens de *Service Accounts* não contam com permissão para fazer absolutamente nada. Mas esse mecanismo trivializa toda a maneira de fazer chegar as credenciais nas aplicações. Basta que você faça as associações necessárias por meio de diretivas RBAC como clusterrolebindings ou rolebindings.

Por que estou compartilhando isto aqui? Porque, em algum lugar perdido na documentação, tem **uma** informação relevante que muita gente desconhece: **como** é validado um token de *Service Account*? Como estes são gerados?

O processo é bem simples: naturalmente, os tokens são gerados pelo [**Kubernetes Controller Manager**](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/). Já que ele tem todos os Controllers que fazem as coisas, nenhuma surpresa aqui. 

A parte importante é este parâmetro de configuração do Controller Manager:

```
--service-account-private-key-file string
Filename containing a PEM-encoded private RSA or ECDSA key used to sign service account tokens.
``` 

Ela indica a chave privada do certificado usado para assinar os tokens.

Kube Controller **gera**, mas quem é acessado é o **Kubernetes API Server**, certo? Sim, e, para fazer a verificação dos tokens, ele **também** conta com um parâmetro de confguração:

```
--service-account-key-file stringArray
File containing PEM-encoded x509 RSA or ECDSA private or public keys, used to verify ServiceAccount tokens. The specified file can contain multiple keys, and the flag can be specified multiple times with different files. If unspecified, --tls-private-key-file is used. Must be specified when --service-account-signing-key is provided
```

Aqui você pode usar tanto a chave privada quanto a chave pública, embora a lógica indique que você deve usar a chave pública.

Ou seja, você precisa de um par chave/certificado para lidar exclusivamente com a validação de tokens de service accounts.

Ok, mas e daí? Por que você está perdendo tanto tempo explicando isso?

Se você vai levantar um ambiente com um único Master Kubernetes, que executará os três componentes que fazem o Control Plane Kubernetes, isso aqui não é um problema. Mas acredito que essa topologia não é exatamente a desejada nem para um cluster simples de ambiente de desenvolvimento.

Idealmente, você instalará pelo menos dois Masters Kubernetes para executar os componentes de Control Plane. Como eles vão compartilhar a mesma configuração, você precisa fazer o mesmo par chave/certificado chegar em **todos os seus Masters**, ou você sentará em uma bomba relógio sem saber.

Em nossa primeira instalação, eu apenas reproduzi os procedimentos de instalação individual de cada máquina. Consequentemente, cada Master Kubernetes recebeu um par diferente de certificados e os usou para tudo. Como resultado, metade das requisições de Pods que faziam uso de ServiceAccounts falhavam, pois o 'outro' API Server estava configurado com uma chave diferente que reportava o token como inválido. Como metade das vezes funcionava, o problema passou batido por um tempo.

Além desse problema, como cada kube-controller-manager está configurado com uma chave privada diferente, os tokens eram gerados de maneira inconsistente, às vezes por uma chave, às vezes por outra.

Em situações em que isso acontece, é necessário deletar **TODOS** os tokens de ServiceAccounts e aguardar que sejam regerados. 

> **Cicatrizes de batalha**: se algum dia você for pego por um problema desse tipo por qualquer razão, você irá descobrir que o kube-controller-manager tme certa má vontade para agir na velocidade que você quer. 

> Na verdade, ele tem certa má vontade para agir em larga escala com todos os componentes. É importante saber disso ao definir escalas para o seu cluster. Observe os seguintes parâmetros de configuração:

```
--concurrent-deployment-syncs int32     Default: 5
The number of deployment objects that are allowed to sync concurrently. Larger number = more responsive deployments, but more CPU (and network) load

--concurrent-endpoint-syncs int32     Default: 5
The number of endpoint syncing operations that will be done concurrently. Larger number = faster endpoint updating, but more CPU (and network) load

--concurrent-namespace-syncs int32     Default: 10
The number of namespace objects that are allowed to sync concurrently. Larger number = more responsive namespace termination, but more CPU (and network) load

--concurrent-replicaset-syncs int32     Default: 5
The number of replica sets that are allowed to sync concurrently. Larger number = more responsive replica management, but more CPU (and network) load

--concurrent-resource-quota-syncs int32     Default: 5
The number of resource quotas that are allowed to sync concurrently. Larger number = more responsive quota management, but more CPU (and network) load

--concurrent-service-endpoint-syncs int32     Default: 5
The number of service endpoint syncing operations that will be done concurrently. Larger number = faster endpoint slice updating, but more CPU (and network) load. Defaults to 5.

--concurrent-service-syncs int32     Default: 1
The number of services that are allowed to sync concurrently. Larger number = more responsive service management, but more CPU (and network) load

--concurrent-serviceaccount-token-syncs int32     Default: 5
The number of service account token objects that are allowed to sync concurrently. Larger number = more responsive token generation, but more CPU (and network) load
```

Levando em consideração que a maioria dos usuários modernos faz uso de Cloud Providers e sequer conta com acesso a qualquer tipo de parametrização do Control Plane, este tipo de informação passa batido. Para ambientes on-prem ou instalações '*não-managed*', é necessário ter noção que existe um 'throttling' interno de operações que pode exigir um certo *tuning* para sua realidade.

## Alternativas old school ao processo de instalação

Como falei, infelizmente, na minha época, Kubernetes 'the Hard Way' na realidade era o 'the only way'. Então minha familiaridade com as técnicas de lançamento de clusters é extremamente limitada pela acomodação e falta de tempo de real P&D do que há no mercado.

Mas vamos tentar cobrir algumas coisas:

### [kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

A ferramenta 'de-facto' para apoio na instalação de clusters Kubernetes. Cobrada, inclusive, em provas de certificação. Ela faz uma série de semi automatizações de etapas óbvias e configura os elementos seguindo um padrão que, ainda que você opte por não usá-la, vale a pena conhecer para deixar o mais parecido possível.

Como o post listado acima na época de sua criação, kubeadm admite que as máquinas em que o cluster Kubernetes será configurado já foram criadas e provisionadas de uma maneira mínima.

A principal crítica é que é relativamente não amistosa para automação.

### [kops](https://kops.sigs.k8s.io/)

Uma das primeiras ferramentas que visava automatizar totalmente a criação de clusters que me lembro surgir. 

O kops não apenas configura Kubernetes em máquinas, ele **as provisiona** também.

Por essa característica, nunca tive a oportunidade de usá-lo. 

Kops opera e é considerado production-grade para AWS; suporte para GCE e Openstack estão em Beta, Azure e DigitalOcean Alpha. Embora ainda exista referências para que um suporte a VMWare Vsphere estava em estágio Alpha também, na verdade o pouco código que existia [já foi removido da codebase](https://github.com/kubernetes/kops/issues/9166).

### [kubespray](https://kubespray.io/#/)

O **kubespray** é um projeto que envolve uma quantidade absurdamente gigantesca de *Tasks Ansible* que visa prover automação total do processo de instalação, não apenas se preocupando com o lado 'kubernetes' da coisa: ele faz 'finalizações' específicas, dependendo se você está usando um cloud provider ou não, e automatiza inclusive ações relacionadas aos CNI, que o kubeadm não faz.

Aqui uma nota relevante: o kubespray **usa** o kubeadm; é uma centenas de etapas do seu processo de automação de clusters.

Já tive a oportunidade de subir alguns clusters usando Kubespray. A experiência gera um tipo de 'mixed feeling'. Como assim?

O kubespray **resolve** o problema de deploy de clusters Kubernetes. Ele **é** 'production ready'. Ele **funciona**.

Mas ele também tenta fazer uma quantidade gigantesca de coisas para atender a um amplo público alvo. O porte do projeto cresce num ritmo incrível, assim como sua complexidade. Já tentou administrar um playbook ansible com cerca de mil *tasks* das quais muitas são interligadas? Não? Pois é, esse é o desafio do kubespray.

Ele **demora**. Afinal, são mais de 1000 tasks ansible.

Às vezes, ele falha, porque bugs do Ansible, ou bugs do python, ou bugs em geral.

Às vezes, ele falha por alguma razão relevante, mas a depuração é particularmente desafiadora, em especial se você tiver pouca familiaridade com ansible (e não achar a mensagem de erro no Google!).

Às vezes, ele não faz o que você precisa, e aí você precisa criar vergonha na cara e tentar implementar a feature na árvore do Kubespray, como eu deveria ter feito porém não fiz. Acredite, é bem mais conveniente tentar fazer um pull request que manter patches para os playbooks, porque **eles mudam** de release para release.

O Kubespray vai demandar bem mais do que apenas aprendeer a executá-lo.

Assim como tudo no universo on-prem, vai demandar mais tempo do que você gostaria de dedicar para poder usá-lo de maneira produtiva no seu ambiente.

## Alternativas modernas ao processo de instalação

Eu já conheço pouco das soluções 'old school'. De qualquer forma, já ouvi falar de uma ou outra que acredito que vale a pena ser listada aqui para referência

### [Kubeone](https://docs.kubermatic.com/kubeone/v1.0/)

Kubeone, da Kubermatic, não é exatamente novíssimo, mas a versão 'GA' 1.0 do projeto foi lançada em agosto de 2020.

A ideia é implementar a mesma proposta do Kubespray: ser uma ferramenta para "lifecycle management" de clusters Kubernetes, seja na nuvem ou em ambientes on-prem.

**Tudo** nesta solução parece fantástico, extremamente promissor e promete substituir praticamente todo o meu trabalho de automação com custom Terraform/Kubespray. Obviamente, vai demantar testes extensivos para entender as limitações.

Qual a pegadinha? Não sei, não descobri. Fora o fato de que existe uma [oferta enterprise](https://docs.kubermatic.com/kubermatic/v2.13/concepts/kubeone-comparison/) por parte da Kubermatic, não consegui perceber de imediato algum problema que torne seu uso inviável.

Se você vai começar a jornada Kubernetes e não tem absolutamente nada, recomendo olhar este projeto antes. Se gostar, me conta!

### [Gardener]()

Na linha dos projetos **ultra modernos**, que tal usar Kubernetes para provisionar clusters Kubernetes? Esta é a proposta do projeto Gardener.

Aqui, o Control plane dos clusters a serem criados pelo Gardener ("shoot clusters") executará como *Pods* em um cluster Kubernetes ("seed cluster"). Genial, não? Qual o propósito de perder tempo fazendo o deploy de máquinas para Control Plane, load balancers para HA em caso de falhas se você pode executar tudo como *Pods*.

A parte curiosa é que este projeto foi desenvolvido pela SAP. Mais curioso ainda é saber que a SAP era um dos [top 10 commiters](https://k8s.devstats.cncf.io/d/9/companies-table?orgId=1&utm_source=thenewstack&utm_medium=website&utm_campaign=platform&var-period_name=Last%20year&var-metric=committers) para o projeto Kubernetes em 2019 (caiu um pouco em 2020).

Talvez a proposta arrojada desencoraje pessoas com menos experiência com Kubernetes. De qualquer forma, está listado aqui porque acho relevante!

### AWS EKS Distro

2020 foi um ano tão estranho que a AWS resolveu liberar o "Amazon EKS Distro", uma distribuição código aberto do Kubernetes.

Não acredita? Está [tudo aqui](https://github.com/aws/eks-distro).

AWS EKS Distro não tem absolutamente nada a ver com os dois exemplos anteriores, e não vai ajudar a provisionar ambientes, mas "empacota" tudo que é necessário para executar um cluster Kubernetes no padrão AWS. Por exemplo, o guia oficial de como usar o EKS-D em um cluster é usando **kops**, mencionado acima.

Serve mais como referência sobre como a AWS configura seus ambientes, para se inspirar se deve fazer algo parecido.

Este ano haverá o lançamento do AWS EKS Anywhere, que possivelmente fará o deploy de clusters automaticamente (?). Mas dificilmente será 'open source'.

---

Alguma sugestão de software para incluir na lista? Deixe-me saber! Compartilhe sua experiência comigo que eu apresento aqui com créditos!