+++
title = "Calicoctl, TLS e gestão de certificados"
description = "O que fazer quando nem o modo DEBUG das ferramentas te dá uma dica?"
author = "Marcelo Andrade"
date = "2020-10-06"
tags = ["kubernetes", "intermediário", "calico"]
categories = ["kubernetes", "intermediário", "calico"]
[[images]]
  src = "img/reusable/square-into-circle.jpg"
  alt = "Square into circle and hammer"
  stretch = ""
+++

Estava precisando configurar algumas [GlobalNetworkPolicies](https://docs.projectcalico.org/reference/resources/globalnetworkpolicy) em um cluster Kubernetes; para isso, é necessário intervir diretamente no Calico, pois o Kubernetes **ainda** não conta com objetos de Networkpolicies que não sejam '*namespaced*'.

Obviamente, usamos '*backend*' ETCD: por que simplificaríamos tudo usando CRDs no próprio Kubernetes, não é? Deixamos isso para os **amadores**! 

(PS: fazemos assim porque somos tão pioneiros em usar tecnologias '*bleeding edge*' que, à época da implantação, usar Kubernetes como backend sequer era uma opção - sim, nós somos '*that old school*'! Ah, e naturalmente, somos preguiçosos demais para migrar agora.)

Mas voltando ao assunto... 

"Calico, dê-me informações!"

```
$ calicoctl -l debug  get globalnetworkpolicies
_
```

NADA!

Putz, caíram minhas regras de firewall de novo?

Chamei um teste de conectividade nas portas, tudo ok:

```
$ for i in 192.168.0.11 192.168.0.12 192.168.0.13 ; do { timeout 1 curl -svz1 telnet://$i:2379 2>&1 ; } | grep Connected; done

* Connected to 192.168.0.11 (192.168.0.11) port 2379 (#0)
* Connected to 192.168.0.12 (192.168.0.12) port 2379 (#0)
* Connected to 192.168.0.13 (192.168.0.13) port 2379 (#0)
```

Nah, tudo ok.

**DEBUG, ATIVAR**:

```
$ calicoctl -l debug  get globalnetworkpolicies

INFO[0000] Log level set to debug                       
INFO[0000] Executing config command                     
DEBU[0000] Resource: projectcalico.org/v3, Kind=Node    
DEBU[0000] Data: - apiVersion: projectcalico.org/v3
  kind: Node
  metadata:
    creationTimestamp: null
  spec: {}
  status: {} 
DEBU[0000] Loading config from JSON or YAML data        
DEBU[0000] Datastore type: etcdv3                       
INFO[0000] Loaded client config: apiconfig.CalicoAPIConfigSpec{DatastoreType:"etcdv3", EtcdConfig:apiconfig.EtcdConfig{EtcdEndpoints:"https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379", EtcdDiscoverySrv:"", EtcdUsername:"", EtcdPassword:"", EtcdKeyFile:"/etc/calico/tls/acof/tls.key", EtcdCertFile:"/etc/calico/tls/acof/tls.crt", EtcdCACertFile:"/etc/calico/tls/acof/tls.ca", EtcdKey:"", EtcdCert:"", EtcdCACert:""}, KubeConfig:apiconfig.KubeConfig{Kubeconfig:"", K8sAPIEndpoint:"", K8sKeyFile:"", K8sCertFile:"", K8sCAFile:"", K8sAPIToken:"", K8sInsecureSkipTLSVerify:false, K8sDisableNodePoll:false, K8sUsePodCIDR:false, KubeconfigInline:"", K8sClientQPS:0}} 
DEBU[0000] Using datastore type 'etcdv3'                
INFO[0000] Client: {{{CalicoAPIConfig projectcalico.org/v3} {      0 {{0 0 <nil>}} <nil> <nil> map[] map[] [] []  []} {etcdv3 {https://192.168.0.11:2379,https://192.168.0.12:2379,https://192.168.0.13:2379    /etc/calico/tls/acof/tls.key /etc/calico/tls/acof/tls.crt /etc/calico/tls/acof/tls.ca   } {      false false false  0}}} 0xc00000ec18 0xc0001fed70} 
DEBU[0000] Processing List request                       list-interface=Node rev=
DEBU[0000] Get Global Resource key from /calico/resources/v3/projectcalico.org/nodes 
DEBU[0000] Didn't match regex                           
DEBU[0000] List options is a parent prefix, ensure path ends in /  list-interface=Node rev=
DEBU[0000] Adding / to path                              list-interface=Node rev=
DEBU[0000] Calling Get on etcdv3 client                  etcdv3-etcdKey=/calico/resources/v3/projectcalico.org/nodes/ list-interface=Node rev=

```

Putz, imprime um monte de lixo inútil para a resolução do problema, e trava na requisição ao servidor ETCD do mesmo jeito.

Qual o próximo passo lógico de diagnóstico agora? Tarô? Leitura de mão?

___

Ainda que não traga nenhuma novidade para a depuração do problema, as mensagens de DEBUG trazem algumas informações relevantes para aqueles pouco familiarizados com ETCD e como ocorre a conexão com esse negócio imprescindível para clusters Kubernetes, mas que pouquíssimas pessoas efetivamente conhecem.

A conectividade é