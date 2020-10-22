+++
title = "Por que Docker?"
description = "Docker e contêiners - razões diretas e pragmáticas perdidas em um texto bem menos direto e pragmático"
author = "Marcelo Andrade"
date = "2020-09-21"
tags = ["containers", "docker", "iniciando"]
categories = ["containers", "docker", "Iniciando"]
[[images]]
  src = "img/reusable/boxes.jpg"
  alt = "Boxes are containers"
  stretch = ""
+++

Em conversas - principalmente com pessoas dos dois extremos do espectro de idade - sobre nosso glorioso mercado de TI vez por outra surge este questionamento: por que usar Docker? 

Eu tendo a reclamação, em especial de estudantes e do pessoal novo: em casa, sem um arsenal de ferramentas de CI/CD implantadas por terceiros, parece uma incrível bobagem e perda de tempo. Dockerfile, rede virtual, putz, pra que complicar, se eu posso rodar tudo na minha máquina sem problemas?

(Já o pessoal antigo, é preguiça e má vontade: vá aprender a usar contêineres e saia da zona de conforto imediatamente!)

## Por que Docker?

Essa resposta é trivial: 

**Docker é simples; Docker é fácil de entender; Docker "faz tudo"**. 

Docker abstrai a infinidade de detalhes necessários para a execução de contêineres no Linux com um cobertor quente, bonito e agradável que permite ao seu usuário ser produtivo com um mínimo de overhead no aprendizado de uma tecnologia "colateral".

E mais: o "engine Docker" vai muito além: o software veio cuidar de praticamente todas as etapas do ciclo de vida de um contêiner, desde sua criação até viabilizar que as imagens cheguem onde desejamos - seja no host, seja no Docker Registry, mantido pelo Docker! -, passando pela execução e manutenção dos containers e imagens no servidor.

Existe uma pergunta muito mais difícil, que inclusive muitos administradores de sistemas que usam contêineres todos os dias nos últimos anos não sabe responder: 

**Se você não usar Docker, vai usar o quê?**

Até mesmo pesquisar essa resposta é [difícil<sup>1</sup>](#sup1); muitos dos conteúdos disponíveis sobre o assunto estão ou extremamente defasados ou simplesmente errados em suas comparações. A resposta a essa pergunta **não atende às necessidades da maioria dos profissionais de TI**, logo é irrelevante (você tem interesse na resposta? Se sim, [me](https://twitter.com/marcelo_devsres) [deixe](https://www.instagram.com/marcelo_devsres/) [saber](https://www.facebook.com/devsresnetwork)!).

Enquanto a "concorrência" não trabalhar isso, Docker será o líder incontestável em uso.

## Por que containers?

Falando a verdade, a pergunta correta não é "por que usar Docker", e sim, por que usar contêineres. Esta sim, diferentemente da primeira, é uma pergunta sem resposta direta e imediata.

Eu poderia copiar definições, justificativas e citar mil livros ou páginas importantes explicando o porquê da relevância da tecnologia de conteinerização sob as diferentes [óticas<sup>2</sup>](#sup2). Mas, em verdade, o que falta para muitos é entender que problemas essa tecnologia veio resolver, e que pode ser resumido em uma sentença:

**Contêineres resolvem o problema de "na minha máquina funciona!".**

Historicamente, sempre coube aos profissionais de infraestrutura o processo de implantação, manutenção e resolução de problemas em ambientes que executam programas elaborados por terceiros. E um desafio tradicional dessa época era a divergência entre os ambientes em que os softwares eram desenvolvidos, homologados e, por fim, postos em produção.

Não era incomum ajustes nos ambientes serem feitos "em tempo de homologação" para corrigir problemas e se perderem na "não documentação" do sistema, que era passada para as áreas de operações implantarem (e, consequentemente, falharem). Também não era incomum o uso de plataformas radicalmente diferentes e incompatíveis entre si, gerando conflitos homéricos sobre o uso de versão de um sistema operacional não internalizado pela empresa, ou a necessidade de atualização de um serviço para uma tecnologia de pouco domínio por parte dos administradores de serviços. Mesmo para tecnologias que naturalmente são pensadas para "rodar em todo lugar", como Java, encontra problemas com versões e parametrizações específicas que podem se perder na transição entre ambientes.

Contêineres, por definição, resolvem o problema de "reproducibilidade de resultados", ou "portabilidade" das aplicações: uma imagem Docker é gerada para a sua aplicação, com todos os arquivos (como elementos do sistema operacional, bibliotecas e utilitários) e configurações necessários para sua execução. Uma consequência comum da adoção deste modelo é a obrigatoriedade de adoção de um mínimo de boas práticas, como por exemplo a parametrização para que a aplicação receba configurações que variam de ambiente para ambiente.

Este problema já foi amplamente abordado no passado como virtualização de servidores e tecnologias de orquestração e gestão de configuração; nenhuma delas, entretanto, resolveu com a elegância e simplicidade de simplesmente "socar" tudo que é necessário em uma unidade parametrizável e orquestrável.

A partir desta entidade [**Contêiner**<sup>3</sup>](#sup3) que trivializa a implantação ("*deploy*") de um software, está aberto agora o caminho para a construção de novas soluções que resolvam outros problemas do processo de desenvolvimento de software e otimizem ainda mais o processo de entrega de resultados por parte das equipes.

---

<sup id=sup1>1</sup> Os melhores links sequer apareciam na página principal do Google:
* [A Comprehensive Container Runtime Comparison](https://www.capitalone.com/tech/cloud/container-runtime/)
* [Aquasec: Docker Alternatives](https://wiki.aquasec.com/display/containers/Docker+Alternatives+-+Rkt%2C+LXD%2C+OpenVZ%2C+Linux+VServer%2C+Windows+Containers)

 <sup id=sup2>2</sup> ["Ópticas" é muito feio.](https://duvidas.dicio.com.br/optica-optico-otica-ou-otico/)
 
 <sup id=sup2>3</sup> Observe que, no Linux, "Contêiner" **não** é uma primitiva do sistema operacional, e sim um conjunto destas. Do ponto de vista prático, acaba sendo uma.

