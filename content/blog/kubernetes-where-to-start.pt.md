+++
title = "Kubernetes: por onde começar"
description = "Como dar os primeiros passos nesta tecnologia"
author = "Marcelo Andrade"
date = "2020-09-01"
tags = ["kubernete", "iniciante"]
categories = ["Kubernetes", "Iniciando"]
[[images]]
  src = "img/reusable/kubernetes-helm-reallife.jpg"
  alt = "Helm on boat in real life"
  stretch = ""
+++

No último ano, aqui em Recife, tive a oportunidade de participar (e palestrar!) em alguns eventos. Em comum, havia o fato de muito de seus conteúdos - se não integralmente - estarem relacionados ao Kubernetes.

Nas duas oportunidade em que fiz apresentações, entrei em detalhes sobre: 

* o desafio de usar o objeto Ingress - que, até a recém-lançada versão 1.19, ainda era beta, embora exista desde a versão 1.1;
* uma nova abordagem de como fazer a gerência de um cluster e a entrega contínua de aplicações: GitOps;

Embora eu sempre procure abordar o assunto de uma forma que não seja absolutamente incompreensível para quem não tenha qualquer tipo de contato com Kubernetes, em todos os eventos, me deparo com uma realidade: a de que existe muita gente interessada sobre Kubernetes, mas com bastante dificuldade de se introduzir a esta tecnologia.

Sempre me é feita uma pergunta: por onde começar? Como dou os primeiros passos? E eu nunca tenho uma resposta de qualidade a oferecer:  eu já "aprendi tudo" lá atrás, em um passado remoto no qual inexistia bons tutorials ou cursos, e a documentação descrevia o Kube Controller como "the program that runs the control loops that controls the state of objects" (ou algo assim).

Felizmente, os tempos mudaram, e é ampla a documentação de cada mínimo aspecto relacionado ao Kubernetes, com bastante oferta de cursos (pagos e gratuitos) sobre o assunto.

Sem mais delongas, **por onde começar** quando quer se aprender sobre Kubernetes?

# KubeAcademy
* https://kube.academy/

A VMware disponibilizou um excelente material (e, vale ressaltar **completamente gratuito**) sobre containers e Kubernetes. Os conteúdos são apresentados de maneira fácil de digerir, e alguns dos cursos inclusive contam com laboratórios práticos interativos com **Katacoda**, o que, na minha opinião, torna este conteúdo possivelmente o melhor disponível para uma apresentação inicial.

A desvantagem para quem não domina o inglês é o uso de player próprio e a inexistência de legendas, embora seja possível regular a velocidade do player para melhorar a compreensão para aqueles não tão fluentes.

# Katacoda / instruqt

* https://www.katacoda.com
* https://play.instruqt.com/public


O Katacoda, adquirido recentemente pela tradicional editora de livros técnicos O'Reilly, por muito tempo, serviu como laboratório prático para quem não tem recursos para subir um cluster Kubernetes para prática pessoal. 

Os cursos dno Katacoda são interativos e executados diretamente no navegador, o que torna o processo simples e dinâmico. Antes de produção restrita, agora o Katacoda permite que qualquer um disposto a aprender a usar a plataforma possa criar seus próprios cursos com ela - o que 

O ponto negativo é que a maneira de abordar o conteúdo teórico fica em segundo plano. E, acredite, muito do trabalhar bem com Kubernetes adequadamente vem de entender a maneira e o porquê optou-se por resolver os problemas desta forma.

Instruqt é bastante semelhante em abordagem ao Katacoda; mas, no geral, a interface apresenta mais problemas e os cursos de Kubernetes em qualidade inferior.

# Kubernetes.io

A documentação do projeto vem crescendo em volume e qualidade a olhos vistos a cada versão que passa. A versão 1.18 introduziu a oferta em outros idiomas que não o inglês, o que nem imaginava que fosse acontecer algum dia.

O destaque da documentação fica por parte dos [**conceitos**](https://kubernetes.io/docs/concepts/overview/).  Muitos dos textos crípticos do começo foram reescritos de maneira compreensível, e agora as ideias são, sim, plenamente acessíveis ao "grande público".

Vale a pena mencionar que a documentação conta com duas seções extremamente valiosas para quem está aprendendo e pensa em tirar a certificação CKA/CKAD: a seção de [**Tasks**](https://kubernetes.io/docs/tasks/) e a de [**Tutoriais**](https://kubernetes.io/docs/tutorials/).

Embora o português esteja lá, a maior parte das páginas que acessei ainda está disponível apenas em inglês.

# EDX - Introduction to Kubernetes

https://www.edx.org/course/introduction-to-kubernetes

Lembro que, à medida que novos membros chegavam à equipe, a recomendação padrão era a fazer este curso do EDX elaborado pela Linux Foundation - basicamente porque era a **única opção**.

É possível ler grande parte dos conteúdos disponíveis sem custo adicional, mas vários recursos são oferecidos apenas aos pagantes. 

Deixo aqui esta entrada pelo valor histórico, mas acredito que a relevância deste curso hoje é limitada se comparado ao que já é oferecido pela documentação oficial do Kubernetes.

# Mas e em português?

Aqui, eu peço desculpas, mas vou ficar devendo.

A qualidade dos materiais que tive a oportunidade de ler (inclusive em sites pagos) está entre o **ruim** e o **sofrível**.  Então, não tenho uma boa recomendação a oferecer.

---

*Se você conhecer algum bom site com conteúdo de Kubernetes em português e quiser recomendar, entre em contato por qualquer uma das minhas redes sociais que eu terei um prazer enorme em incluí-los nesta lista!*