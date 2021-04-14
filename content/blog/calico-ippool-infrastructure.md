+++
title = "Calico IPPool dedicado para Infraestrutura"
description = "Como habilitar um IPPool apenas para os administradores"
author = "Marcelo Andrade"
date = "2021-04-13"
tags = ["calico", "kubernetes"]
categories = ["calico", "kubernetes"]
[[images]]
  src = "img/reusable/hammer-screw.jpg"
  alt = "Minecraft"
  stretch = ""
+++

**Setup**: Kubernetes com Calico CNI. 

Você tem o IPPool "oficial" para os pods de workload dos seus tenants, limitado porque conta com IPs válidos na rede para auditoria e que não sofrem NAT (*natOutgoing=false*) em suas comunicações com serviços dentro ou fora do cluster. 

Ok.

Mas temos alguns pods "de infraestrutura". Esses pods executam nos mesmos hosts de workload normal, mas precisam de acesso diferenciado e não queremos desperdiçar IPs válidos com eles.

**Solução 1**:

Você pode usar .hostNetwork=true (o equivalente K8s para o Docker --net=host).

Isso não é nem conveniente (colisão de portas?), nem seguro (https://research.nccgroup.com/2020/11/30/technical-advisory-containerd-containerd-shim-api-exposed-to-host-network-containers-cve-2020-15257/). 

**Solução 2**:

Criar um **IPPool**, com outro **CIDR** diferente do anterior, e inválido na Intranet. Nele você pode até mesmo usar um *natOutgoing=true* (SNAT para o IP do nó) para garantir os mesmos acessos do host ao Pod (que pode ser, por exemplo, um logshipper estilo fluentd/bit).

Mas como impedir que Pods de workload recebam IPs? O Calico oferece a seguinte alternativa:

1 - No **IPPool ** novo, é possível configurar o *nodeSelector=!all()*, que fará com que nenhum nó irá pegar blocos dessa rede automaticamente (ref: https://docs.projectcalico.org/reference/resources/ippool).

```yaml
$ cat ippool.yaml
apiVersion: crd.projectcalico.org/v1
kind: IPPool
metadata:
  name: infraestrutura
spec:
  blockSize: 25
  cidr: 172.16.0.0/16
  ipipMode: CrossSubnet
  natOutgoing: true
  nodeSelector: !all()
```

2 -  Nos elementos de infraestrutura desejados (namespace/deployment/daemonset), pode-se usar a *annotation* *cni.projectcalico.org/ipv4pools: "[\"pool-infra\"]"*. O Calico CNI se responsabilizará por dar o IP adequado. (ref: https://docs.projectcalico.org/networking/legacy-firewalls)

No caso, configurei os Runners do Gitlab para provisionarem os Pods com a annotation apropriada:

```bash
$ kubectl get pod runner-hfuy2bn3-project-19241-concurrent-0 -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/ipv4pools: '["infraestrutura"]'
  generateName: runner-hfuy2bn3-project-19241-concurrent-0
  labels:
    pod: runner-hfuy2bn3-project-19241-concurrent-0
  name: runner-hfuy2bn3-project-19241-concurrent-0629tc
  namespace: gitlab-runners
```
O resultado é o abaixo:

```bash
$ kubectl get pods -o jsonpath="{range .items[*]}{.status.podIP}:{.metadata.name}{'\n'}{end}"
10.10.100.1:gitlab-gitlab-runner-74b8bfb9c9-762hc
10.10.100.2:gitlab-gitlab-runner-74b8bfb9c9-7hn2l
10.10.100.3:gitlab-gitlab-runner-74b8bfb9c9-qqrpg
172.16.12.2:runner-hfuy2bn3-project-19241-concurrent-0629tc

```