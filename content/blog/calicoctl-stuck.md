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

A chave para a solução do problema passa sutilmente despercebida na diretiva EtcdEndpoints:

```
...https://...
```

Esse tipo de erro obscuro, fantasma, de conexões que não começam (ou, no caso, que nunca terminal) são típicos de problemas de validação TLS. Mas que tipo de validação TLS? 

Um programa "bem feito" normalmente ajuda a depuração. Vamos analisar o comportamento do comando **curl** quando exposto a diversos problemas de validação TLS:

* Autoridade certificadora inválida:
```
# curl https://etcd1.local:2379
curl: (60) Peer's Certificate issuer is not recognized.
```
* Nome do servidor não consta no certificado:
```
# curl --cacert tls/sp2/ca.pem --resolve hostname.invalido:2379:10.99.17.11 https://hostname.invalido:2379 
curl: (51) Unable to communicate securely with peer: requested domain name does not match the server's certificate.
```
* O servidor requer autenticação com certificado válido pelo cliente, e vocẽ não enviou nenhum:
```
# curl --cacert tls/sp2/ca.pem https://etcd1.local:2379 
curl: (58) NSS: client certificate not found (nickname not specified)
```

Bem, já é o suficiente.

O **calicoctl**, entretanto, deixa **bastante a desejar** nesse cenário (A Tigera possivelmente acredita que você não terá problemas para reconhecer algo tão bobo?).

Praticamente **todos os erros que envolvem certificados e TLS** terminam com o *calicoctl* travando por tempo indefinido.

* Autoridade certificadora inválida:
```
# ETCD_ENDPOINTS=https://etcd1.local:2379 calicoctl -l debug get nodes
...
INFO[0000] Loaded client config: apiconfig.CalicoAPIConfigSpec{DatastoreType:"etcdv3", EtcdConfig:apiconfig.EtcdConfig{EtcdEndpoints:"https://etcd1.local:2379", EtcdDiscoverySrv:"", EtcdUsername:"", EtcdPassword:"", EtcdKeyFile:"", EtcdCertFile:"", EtcdCACertFile:"", EtcdKey:"", EtcdCert:"", EtcdCACert:""}, KubeConfig:apiconfig.KubeConfig{Kubeconfig:"", K8sAPIEndpoint:"", K8sKeyFile:"", K8sCertFile:"", K8sCAFile:"", K8sAPIToken:"", K8sInsecureSkipTLSVerify:false, K8sDisableNodePoll:false, K8sUsePodCIDR:false, KubeconfigInline:"", K8sClientQPS:0}} 
... 
^C
```

* Nome do servidor não consta no certificado:
```
# # Criei uma entrada no /etc/hosts para etcd.devsres.com, o nome não resolve.
# ETCD_ENDPOINTS=https://etcd.devsres.com:2379 ETCD_CA_CERT_FILE=tls/tls.ca calicoctl -l debug get nodes
...
INFO[0000] Loaded client config: apiconfig.CalicoAPIConfigSpec{DatastoreType:"etcdv3", EtcdConfig:apiconfig.EtcdConfig{EtcdEndpoints:"https://etcd.devsres.com:2379", EtcdDiscoverySrv:"", EtcdUsername:"", EtcdPassword:"", EtcdKeyFile:"", EtcdCertFile:"", EtcdCACertFile:"tls/tls.ca", EtcdKey:"", EtcdCert:"", EtcdCACert:"tls/tls.ca"}, KubeConfig:apiconfig.KubeConfig{Kubeconfig:"", K8sAPIEndpoint:"", K8sKeyFile:"", K8sCertFile:"", K8sCAFile:"", K8sAPIToken:"", K8sInsecureSkipTLSVerify:false, K8sDisableNodePoll:false, K8sUsePodCIDR:false, KubeconfigInline:"", K8sClientQPS:0}} 
... 
^C
```
* O servidor requer autenticação com certificado válido pelo cliente, e vocẽ não enviou nenhum: 
```
# ETCD_ENDPOINTS=https://sp2srvvpkv00001:2379 ETCD_CA_CERT_FILE=tls/tls.ca calicoctl -l debug get nodes 
...
INFO[0000] Loaded client config: apiconfig.CalicoAPIConfigSpec{DatastoreType:"etcdv3", EtcdConfig:apiconfig.EtcdConfig{EtcdEndpoints:"https://sp2srvvpkv00001:2379", EtcdDiscoverySrv:"", EtcdUsername:"", EtcdPassword:"", EtcdKeyFile:"", EtcdCertFile:"", EtcdCACertFile:"tls/tls.ca", EtcdKey:"", EtcdCert:"", EtcdCACert:""}, KubeConfig:apiconfig.KubeConfig{Kubeconfig:"", K8sAPIEndpoint:"", K8sKeyFile:"", K8sCertFile:"", K8sCAFile:"", K8sAPIToken:"", K8sInsecureSkipTLSVerify:false, K8sDisableNodePoll:false, K8sUsePodCIDR:false, KubeconfigInline:"", K8sClientQPS:0}} 
...
^C
```

Enfim, você não pode confiar que o *calicoctl* irá te contar se houver qualquer tipo de falha na validação TLS, seja ela do lado do servidor ou do cliente.

A mensagem que me permitiu diagnosticar adequadamente o problema foi a consulta dos logs dos servidores ETCD diretamente:

```
Oct 15 12:18:29 etcd1.local docker[832]: 2020-10-15 15:18:29.066104 I | embed: rejected connection from "192.168.255.42:54608" (error "tls: failed to verify client's certificate: x509: certificate has expired or is not yet valid", ServerName " etcd1.local")

Oct 15 12:18:59  etcd1.local docker[832]: 2020-10-15 15:18:59.136121 I | embed: rejected connection from "192.168.255.42:56290" (error "tls: failed to verify client's certificate: x509: certificate has expired or is not yet valid", ServerName " etcd1.local")

```

Aqui, diagnóstico trivial: os **pseudo administradores** deste ambiente deixaram os certificados do cliente calicoctl **expirarem**.
___

Conclusão:

* Conectividade TLS pode ser um horror para depurar;
* Se você gera certificados para sua Intranet, implemente um controle adequado para saber quando estes estiverem próximos de expirar;

