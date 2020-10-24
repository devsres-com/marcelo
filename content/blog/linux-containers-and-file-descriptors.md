+++
title = "Linux, contêineres e descritores de arquivos"
description = "Ou: por que você PRECISA de um Linux guru na sua equipe de 'software engineers'"
author = "Marcelo Andrade"
date = "2020-10-23"
tags = ["docker", "linux"]
categories = ["docker", "linux"]
[[images]]
  src = "img/reusable/penguins-docker.jpg"
  alt = "Imagem de Siggy Nowak por Pixabay"
  stretch = ""
+++

O mundo mudou, e tudo hoje é "*software engineer*" e containers em todas as direções. X as a Code, "abordagem dev para tudo", "o que um dev faria?", e assim vai. E como fica o profissional de **infraestrutura**? Aquele cara que hoje é conhecido como **Ops** puro sangue, e que foi basicamente substituído por desenvolvedores que entendem um pouco mais de operação que os outros? 

Eu sou um desses caras. Um **Guru Linux** das antigas, dos tempos do Slackware, *dependency chain download* e das configurações de X que queimavam monitores. 

Talvez este post de hoje mostre que ainda vale a pena ter "um de nós" no seu time!

---

Uma nova imagem docker para uma solução interna foi gerada com atualizações e melhorias. A imagem está 100% operacional; todos os testes funcionais estão ok. 

Mas ela está com um pequeno detalhe: não temos **logs**.

E basicamente o único propósito desta imagem é gerar esses logs, que servem para auditoria. Sem logs, ela não serve para absolutamente nada.

O que mudou no ambiente entre as duas versões? Basicamente **tudo**:

* Versão do SO;
* Versão do Docker;
* Versão do Kubernetes;
* Versão da imagem base;
* Versão do próprio software;
* Scripts de inicialização do container;

E aí, por onde começar?

---

Aqui, um pouco de contexto:

A aplicação é legada e **não sabe jogar logs na saída padrão**.

Para remediar este problema, a solução foi:

```
...
    ln -sf /dev/stdout /var/log/app/aplication.log && \    
...
```
> **Nota**: nem sempre isso resolve o problema. O [Tinyproxy](http://tinyproxy.github.io/) e o [pureftpd],(https://www.pureftpd.org/) em algum momento na história, não funcionavam, embora atualmente salvo engano estão ok).

---

Esse problema foi trazido para a equipe.

Como **tudo** foi mudado (o terror de quem adora culpar "a mudança" pela causa do problema), é particularmente doloroso descascar essa cebola sem chorar; em especial, se te falta conhecimento em **sistemas operacionais**. 

E é por isso que é importante que as empresas mantenham por perto seus **Gurus em Ops**!

A solução imediata proposta por um dos outros **Ops** foi:

* introduzir um syslog no container via sidecar;
* reconfigurar a aplicação para enviar seus logs para o syslog;
* fazer o syslog exibir as mensagens na saída padrão.

Possivelmente esta é a solução adequada, mas eu **me recuso a aceitar a derrota**. Introduzir mais um container (ou processo) sem entender o que aconteceu, para mim, é inaceitável. Pode até ser implementado dessa forma, depois que o problema for devidamente entendido.

---

## Analisando o Dockerfile

Como listado acima, foi criado um link simbólico associando o arquivo de logs a **/dev/stdout**:

```
...
    ln -sf /dev/stdout /var/log/app/aplication.log && \    
...
```
E o que isso representa para o container em execução?

```
lrwxrwxrwx    1 root     root            11 Mar 27  2018 /var/log/app/application.log -> /dev/stdout
lrwxrwxrwx    1 user     user            15 Oct 24 00:31 /dev/stdout -> /proc/self/fd/1
lrwx------    1 user    user             64 Oct 24 02:34 /proc/self/fd/1 -> /dev/pts/0
```

Basicamente, que estamos linkando o arquivo para **/proc/self/fd/1**. 

Sempre lembrando que, no Linux, cada processo normalmente tem os [três descritores de arquivo padrão POSIX](https://en.wikipedia.org/wiki/File_descriptor):
* stdin: fd/0
* stdout: fd/1
* stderr: fd/2

Então estamos associado o arquivo de logs da aplicação ao stdout (fd/1) do processo (self).

Até aqui, tudo bem.
___ 

Para quem nunca parou para pensar, essa é a explicação de uma outra famosa 'frase shell':

```shell
# Por que isso...
$ command  > /dev/null 2>&1

# É diferente disso?
$ command 2>&1 > /dev/null
```

Neste caso, você está redirecionando o descritor de arquivos **stdout** (ou 1) que será aberto para seu comando para **/dev/null** (>, também podendo ser escrito 1>). E redirecionando o descritor de arquivos **stderr** (ou 2) para o mesmo descritor de arquivos de 1, (i.e., /dev/null também). Isso faz com que toda a saída do comando 'desapareça' - muito comum em scripts nos quais os devs querem sumir com todo tipo de mensagem de warning, e que torna depurar a causa do problema um terror mais tarde!

Se você fizer na ordem inversa, você redireciona **stderr** para **stdout**, e depois, **stdout** para /dev/null. Essa operação não impacta a anterior; logo, a saída regular do comando é suprimida, mas a saída de erro ainda será exibida (no lugar "errado", mas será). É útil se o programa é excessivamente 'verbose' e você só tem interesse nas possíveis mensagens de erro.

Não entendeu? Alguém desenhou [aqui](https://wiki.bash-hackers.org/howto/redirection_tutorial#order_of_redirection_ie_file_2_1_vs_2_1_file).

## O que o docker faz para exibir logs?

O*Docker* possui um parâmetro *log-driver* que habilita a conhecida função *docker logs* - apenas os logdrivers 'json-file' e 'journald' viabilizam seu uso. O log-driver padrão é 'json-file'.

A descrição deste log-driver está na [documentação oficial](https://docs.docker.com/config/containers/logging/json-file/):

```
By default, Docker captures the standard output (and standard error) of all your containers, and writes them in files using the JSON format. The JSON format annotates each line with its origin (stdout or stderr) and its timestamp. Each log file contains information about only one container.
```

Portanto, o Docker joga o conteúdo gerado pelo **fd/1** e **fd/2** do container em um arquivo. 

> **Nota**: Um **Pod** Kubernetes composto por 2 containers irá gerar dois arquivos diferentes. 

Não tem exemplos na documentação, então aqui segue um:

```
{"log":"I1024 03:08:36.962547      12 instance.go:317] updating 0 host(s): []\n","stream":"stderr","time":"2020-10-24T03:08:36.962684216Z"}
```

Ele inclusive tem a bondade de ilustrar para você de que tipo é ("stream": "**stderr**").

Os arquivo normalmente são jogados em /var/lib/docker/containers, mas isso é irrelevante.

O que **realmente** é relevante é como o Docker faz a ponte da saída dos comandos para este arquivo.

## Como o linux associa os descritores de arquivos de processos?

Antes de chegar no Docker, como ver os descritores de arquivos de um processo?

Primeiro, descobrimos o pid do nosso processo, por exemplo, o shell da minha sessão atual:

```
$ ps
   PID TTY          TIME CMD
  9896 pts/0    00:00:00 bash
195216 pts/0    00:00:00 ps
```

PID 9896, ok. Vamos conferir:

```
ls -l /proc/9896/fd 
total 0
lrwx------. 1 core core 64 Oct 24 00:43 0 -> /dev/pts/0
lrwx------. 1 core core 64 Oct 24 00:43 1 -> /dev/pts/0
lrwx------. 1 core core 64 Oct 24 00:43 2 -> /dev/pts/0
lrwx------. 1 core core 64 Oct 24 01:20 255 -> /dev/pts/0
```

(255? Sim, porque **bash**. Deixo essa história para outro dia.)

E o que é /dev/pts/0? São os "pseudo terminais" criados para podermos interagir com o SO. Se você abre múltiplas janelas de terminais, vai ver isso aqui:

```
$ ls /dev/pts
0  1  10  11  12  2  3  4  5  6  7  8  9  ptmx
```

Na máquina acima, só tenho um:
```
$ ls /dev/pts/
0  ptmx
```

(E o ptmx? shhhh, ele é o 'master'! Mas deixa esse assunto pra lá.)

Tá, o **bash** abriu um terminal. E outros programas que normalmente rodam em backgroun, por exemplo, o kubelet?

```
sudo ls -l /proc/$( pgrep kubelet )/fd/
total 0
lr-x------. 1 root root 64 Sep 19 19:17 0 -> /dev/null
lrwx------. 1 root root 64 Sep 19 19:17 1 -> 'socket:[14836]'
lrwx------. 1 root root 64 Sep 19 19:17 10 -> 'socket:[49723]'
lrwx------. 1 root root 64 Sep 19 19:17 11 -> 'socket:[25802]'
lrwx------. 1 root root 64 Sep 19 19:17 12 -> 'socket:[33709]'
lr-x------. 1 root root 64 Sep 19 19:17 13 -> anon_inode:inotify
lrwx------. 1 root root 64 Sep 19 19:17 15 -> 'socket:[34950]'
lr-x------. 1 root root 64 Sep 19 19:17 16 -> 'pipe:[14848]'
l-wx------. 1 root root 64 Sep 19 19:17 17 -> 'pipe:[14848]'
lrwx------. 1 root root 64 Sep 19 19:17 18 -> 'socket:[39097]'
lrwx------. 1 root root 64 Sep 19 19:17 19 -> 'socket:[34949]'
lrwx------. 1 root root 64 Sep 19 19:17 2 -> 'socket:[14836]'
lrwx------. 1 root root 64 Sep 19 19:17 20 -> 'socket:[3073082]'
lr-x------. 1 root root 64 Sep 19 19:17 21 -> /dev/kmsg
lrwx------. 1 root root 64 Sep 19 19:17 22 -> 'socket:[274349500]'
lrwx------. 1 root root 64 Sep 19 19:17 23 -> 'socket:[802786832]'
lrwx------. 1 root root 64 Sep 19 19:17 26 -> 'socket:[44326]'
lrwx------. 1 root root 64 Sep 19 19:17 27 -> 'socket:[44328]'
lr-x------. 1 root root 64 Sep 19 19:17 28 -> anon_inode:inotify
lrwx------. 1 root root 64 Oct 13 22:33 29 -> 'socket:[2566051059]'
lrwx------. 1 root root 64 Sep 19 19:17 3 -> 'socket:[14840]'
lr-x------. 1 root root 64 Sep 19 19:17 30 -> anon_inode:inotify
lrwx------. 1 root root 64 Sep 19 19:17 32 -> 'socket:[36435]'
lr-x------. 1 root root 64 Sep 19 19:17 33 -> anon_inode:inotify
lrwx------. 1 root root 64 Sep 19 19:17 35 -> 'socket:[14849]'
lr-x------. 1 root root 64 Sep 19 19:17 36 -> /dev/kmsg
lrwx------. 1 root root 64 Sep 19 19:17 37 -> 'anon_inode:[eventpoll]'
lr-x------. 1 root root 64 Sep 19 19:17 38 -> 'pipe:[25803]'
l-wx------. 1 root root 64 Sep 19 19:17 39 -> 'pipe:[25803]'
lrwx------. 1 root root 64 Sep 19 19:17 4 -> 'anon_inode:[eventpoll]'
lrwx------. 1 root root 64 Sep 19 19:17 5 -> 'socket:[2621822183]'
lrwx------. 1 root root 64 Sep 19 21:17 52 -> 'socket:[1275760594]'
lrwx------. 1 root root 64 Sep 19 19:17 6 -> 'socket:[2623166919]'
lrwx------. 1 root root 64 Sep 19 19:17 7 -> 'anon_inode:[eventpoll]'
lrwx------. 1 root root 64 Sep 19 19:17 8 -> 'socket:[30969]'
lrwx------. 1 root root 64 Sep 19 19:17 9 -> 'socket:[30971]'
```

Putz, que horror, hein? 

Bem, o que importa é que descritores de arquivos são associados a coisas. Pode ser um terminal, pode ser qualquer uma das tralhas acima. 

## Como o docker faz a ponte entre container e log file

Para ver facilmente informações sobre o seu container no systemd, você pode executar o comando **machinectl** para listar os containeres existentes:

```
# machinectl
No machines.
```

Hum. Esqueci que o Docker odeia o systemd e eles não se conversam direito.

Vamos do jeito mais difícil:

```
# systemd-cgls
Control group /:
-.slice
├─kubepods
│ ├─burstable
...
│ │ ├─pod6540eb7c-34d1-42c4-b6c7-cf83bd05e472
│ │ │ ├─df5869b859ea271c8836708c055b32387d26584c2e2824b0898e8bcbf7f6d6e5
│ │ │ │ └─164408 /pause
│ │ │ └─8169619db23377b7df0ba05babcb8e3c17cfa662ff024b8cd136a4c6e2594070
│ │ │   ├─165234 /sbin/tini /usr/local/bin/start.sh
│ │ │   ├─165311 /bin/bash /usr/local/bin/start.sh
│ │ │   ├─/ bash /usr/local/bin/notify.sh
│ │ │   ├─166143 inotifywait -qrm --exclude=.*\.conf -e MOVED_TO --format %e:...
│ │ │   ├─166155 /usr/bin/app

```

Aqui vemos um típico **Pod** Kubernetes, que é sempre composto por pelo menos **dois** containers:
* O famoso container 'pause' (df5869b859ea...)
* O container da aplicação (8169619db233...).

O container conta com um init simplório (tini) que executa um shell script, que executa um shell script(!), que executa a aplicação. 

A partir do nome do container, é possível investigar o PID do processo encarregado pelo containerd de cuidar do nosso container:

```
# ps auxw | fgrep b68d2844a70ea6 | head -n1
root     144768  0.0  0.0   8564  4604 ?        Sl   02:34   0:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/b68d2844a70ea62317392d31f16590ef05f1afb2431519c47df7bfaa3ecd81f9 -address /run/docker/libcontainerd/docker-containerd.sock -containerd-binary /run/torcx/unpack/docker/bin/containerd -runtime-root /var/run/docker/runtime-runc -debug
```
E, com isso, podemos ver seus file descriptors:

```
# # Suprimi alguns da saída., daí os números ausentes.
# ls -l /proc/144768/fd
total 0
lr-x------. 1 root root 64 Oct 24 02:36 0 -> /dev/null
lrwx------. 1 root root 64 Oct 24 02:36 1 -> 'socket:[25008]'
lrwx------. 1 root root 64 Oct 24 02:36 2 -> 'socket:[25008]'
...
lrwx------. 1 root root 64 Oct 24 02:36 4 -> 'anon_inode:[eventpoll]'
lrwx------. 1 root root 64 Oct 24 02:36 5 -> 'anon_inode:[eventpoll]'
lrwx------. 1 root root 64 Oct 24 02:36 6 -> 'socket:[3134831467]'
lrwx------. 1 root root 64 Oct 24 02:36 7 -> 'socket:[3134853345]'
lr-x------. 1 root root 64 Oct 24 02:36 8 -> 'pipe:[3134884039]'
...
lr-x------. 1 root root 64 Oct 24 02:36 10 -> 'pipe:[3134884040]'
lr-x------. 1 root root 64 Oct 24 02:36 12 -> 'pipe:[3134884041]'
lrwx------. 1 root root 64 Oct 24 02:34 21 -> /dev/pts/ptmx
```
Interessante.

Vamos olhar os descritores de arquivos desses processos:

```
# ls -l /proc/{165234,165311,166142,166143,166155}/fd/
/proc/165234/fd/:
total 0
dr-x------. 2 31 31  0 Oct 24 01:42 .
dr-xr-xr-x. 9 31 31  0 Oct 24 01:42 ..
lr-x------. 1 31 31 64 Oct 24 01:42 0 -> ''pipe:[3134884039]''
l-wx------. 1 31 31 64 Oct 24 01:42 1 -> 'pipe:[3134884040]'
l-wx------. 1 31 31 64 Oct 24 01:42 2 -> 'pipe:[3134884041]'

/proc/165311/fd:
total 0
dr-x------. 2 31 31  0 Oct 24 01:42 .
dr-xr-xr-x. 9 31 31  0 Oct 24 01:42 ..
lr-x------. 1 31 31 64 Oct 24 01:42 0 -> ''pipe:[3134884039]''
l-wx------. 1 31 31 64 Oct 24 01:42 1 -> 'pipe:[3134884040]'
l-wx------. 1 31 31 64 Oct 24 01:42 2 -> 'pipe:[3134884041]'
lr-x------. 1 31 31 64 Oct 24 01:42 255 -> /usr/local/bin/start.sh

/proc/166142/fd:
total 0
dr-x------. 2 31 31  0 Oct 24 01:42 .
dr-xr-xr-x. 9 31 31  0 Oct 24 01:42 ..
lr-x------. 1 31 31 64 Oct 24 01:42 0 -> ''pipe:[3134884039]''
l-wx------. 1 31 31 64 Oct 24 01:42 1 -> 'pipe:[3134884040]'
l-wx------. 1 31 31 64 Oct 24 01:42 2 -> 'pipe:[3134884041]'
lr-x------. 1 31 31 64 Oct 24 01:42 255 -> /usr/local/bin/notify.sh

/proc/166143/fd:
total 0
dr-x------. 2 31 31  0 Oct 24 01:42 .
dr-xr-xr-x. 9 31 31  0 Oct 24 01:42 ..
lr-x------. 1 31 31 64 Oct 24 01:42 0 -> ''pipe:[3134884039]''
l-wx------. 1 31 31 64 Oct 24 01:42 1 -> 'pipe:[2185069392]'
l-wx------. 1 31 31 64 Oct 24 01:42 2 -> /dev/null
lr-x------. 1 31 31 64 Oct 24 01:42 3 -> anon_inode:inotify

/proc/166155/fd/:
total 0
dr-x------. 2 31 31  0 Oct 24 01:42 .
dr-xr-xr-x. 9 31 31  0 Oct 24 01:42 ..
lrwx------. 1 31 31 64 Oct 24 01:42 0 -> /dev/null
lrwx------. 1 31 31 64 Oct 24 01:42 1 -> /dev/null
lrwx------. 1 31 31 64 Oct 24 01:42 2 -> /dev/null
```

Ahá!

Basicamente, todos os **stdout** dos PIDs dos shell scripts foram direcionados para'pipe:[3134884040]'! Este pipe está associado a um outro *fd* do containerd-shim que fará as mensagens chegarem no arquivo de logs apropriado em /var/lib/docker/containers.

Mas e o processo da aplicação no container? Tanto stdin, quando stdout quanto stderr apontam para **/dev/null**!

Por que isso aconteceu?

Bem, porque a maior parte dos *daemons* no Linux, ao subir, fazem a atribuição de pelo menos os fds 0 e 1 a /dev/null. Alguns ainda deixam stderr livre para imprimir mensagens de erro, mas não foi o caso dessa aplicação.

Veja este container aqui com nginx:

```
# ps auxw
PID   USER     TIME  COMMAND
    1 nobody    0:00 /usr/bin/dumb-init -- /usr/local/bin/start-inotify.sh
    8 nobody    0:00 {start-inotify.s} /bin/bash /usr/local/bin/start-inotify.sh
   10 nobody    0:00 nginx: master process nginx -c /etc/nginx/nginx.conf
   11 nobody   13:17 nginx: worker process

# ls -l /proc/10/fd
total 0
lrwx------    1 nobody   nobody          64 Oct 24 05:19 0 -> /dev/null
lrwx------    1 nobody   nobody          64 Oct 24 05:19 1 -> /dev/null
l-wx------    1 nobody   nobody          64 Oct 24 05:19 2 -> pipe:[3001744697]
lrwx------    1 nobody   nobody          64 Oct 24 05:19 3 -> socket:[3001748727]
l-wx------    1 nobody   nobody          64 Oct 24 05:19 4 -> pipe:[3001744697]
l-wx------    1 nobody   nobody          64 Oct 24 05:19 5 -> pipe:[3001744696]
lrwx------    1 nobody   nobody          64 Oct 24 05:19 6 -> socket:[3001742291]
lrwx------    1 nobody   nobody          64 Oct 24 05:19 7 -> socket:[3001748728]

```

Então, relembrando o nosso Dockerfile:

```
...
    ln -sf /dev/stdout /var/log/app/aplication.log && \    
...
```

Não é nenhuma surpresa que não esteja funcionando. Você mandou apontar o log da aplicação para /dev/stdout, que é /dev/null.

## Como resolver?

Este é o famoso caso em que se demora mais para explicar a solução do problema que para resolver o problema.

A explicação já foi acima.

A troca abaixo "resolve" o problema:

```
    ln -sf /dev/stdout /var/log/app/aplication.log && \    
```

por 

```
ln -sf /proc/1/fd/1 /var/log/app/aplication.log && \
```

Um truque sujo, simples, porém altamente eficaz e com o mínimo de trauma no deploy da aplicação. O **stdout** do pid 1 **não é** /dev/null, e agora o logs da aplicação aparecem normalmente.

> **Nota**:  solução pode não funcionar em ambientes com --pid=host, SELinux ou Apparmor ativo sem modificações das policies padrão!