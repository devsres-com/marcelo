+++
title = "Prometheus node exporter com TLS"
description = "Restringindo acesso ao endpoint de métricas da sua máquina"
author = "Marcelo Andrade"
date = "2020-10-30"
tags = ["prometheus", "node-exporter", "tls"]
categories = ["prometheus", "node-exporter", "tls"]
[[images]]
  src = "img/reusable/prometheus.png"
  alt = "Prometheus having its liver devoured by the eagle"
  stretch = ""
+++

Ah, Prometheus. O mundo "*cloud-native*" simplesmente **adora**. 

Desde sua recepção pela CNCF em 2016 como primeiro projeto - após o Kubernetes - a ser incubado, a adesão e admiração tem sido crescente, a ponto de você encontrar issues como esta em [projetos aleatórios na Internet](https://github.com/goharbor/harbor/issues/4557#issuecomment-377801119):

> jnovack commented on Apr 1, 2018
> I'm pretty sure Prometheus-compatible metrics exposure is almost a requirement in 2018 for project adoption.

(Ps: o projeto em questão ainda não exporta métricas)

E isso antes mesmo de ser considerado um projeto "graduado"

Logo, como somos modistas, vamos substituir **todas as monitorações internas** por Prometheus porque a gente **pode**.

(Algum dia faço uma análise se isso é estúpido ou genial!)

O primeiro passo é implantar o [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) para exportar métricas gerais do nó (i.e., substituir o tradicional cpu/memória/disco). 

---

## O mínimo que você precisa saber sobre Prometheus para não passar vergonha

* Você executa o programa na máquina; 
* Ele abre um endpoint HTTP em uma porta (para o Node Exporter, o default é 9100);
* Os softwares que você instala e geram métricas são chamados de "*exporters*" (dã). 
* Você precisa configurar o servidor para 'ir buscar' as métricas; o nome jurídico disso em prometês é "*scrape*".

O que torna o Prometheus uma ideia genial como solução de monitoração é que o foco **não são os exporters**, mas sim a instrumentação do código das aplicações usando as bibliotecas do projeto. Assim, sua aplicação pode oferecer nativamente um endpoint para métricas **sem precisar de um exporter**.

Prometheus é uma solução de monitoração que vem com sotaque "Dev", não "Ops".

## Prometheus e TLS

O acesso aos endpoints de métricas dos exporters corriqueiramente é feito usando protocolo HTTP simples nas portas expostas, o que não incomoda a maioria das pessoas.

Obviamente isso não é o mais recomendado; você não quer suas métricas expostas por aí, ou mesmo uma saraivada de portas abertas no seu host exposto à Internet, por exemplo.

Ainda há um agravante: determinados exporters (o próprio Node Exporter é um desses), dependendo do tipo de carga a que a máquina está submetida, podem, por si só, onerar excessivamente a máquina em caso de muitas consultas repetidas.

Nós já conseguimos simular um DoS em um host Kubernetes acessando continuamente o endpoint de métricas daquela máquina exposto via HTTP. Logo, deixar estes endpoints expostos não é uma alternativa.

O problema aqui é: a maioria dos exporters não implementa a opção de controle de acesso ou mesmo HTTPS/TLS. Você precisa se virar.

Existem várias alternativas para resolver este problema:

* [A documentação oficial](https://prometheus.io/docs/guides/tls-encryption/) sugere você instalar e configurar um [Nginx](https://www.nginx.com/);
* Uma [issue](https://github.com/prometheus/node_exporter/issues/1286#issuecomment-474732671) no projeto Prometheus Node Exporter sugere o projeto [Ghostunnel](https://github.com/ghostunnel/ghostunnel);
* Para aplicações que executam em clusters Kubernetes, existe a alternativa do [Kube RBAC Proxy](https://github.com/brancz/kube-rbac-proxy).

O que usamos hoje é uma solução interna baseada em Nginx que lê um configmap e expõe diversos exporters em uma única porta, usando o truque de "*path redirect*" dos "*scrapers*"; os diversos exporters são configurados para bind em *localhost* apenas.

## Prometheus Node Exporter com suporte nativo TLS

Porém, se você prestou atenção na issue em que é sugerido o Ghostunell, vai observar que ela faz menção à **implementação nativa** de um [endpoint TLS HTTPS](https://github.com/prometheus/node_exporter#tls-endpoint) para o Prometheus Node Exporter!

Calma, esta funcionalidade ainda está descrita como experimental, e a recomendação ainda é você manter seu proxy nos ambientes de produção, ok?

**De jeito nenhum**! Aqui é bleeding edge, p@r#!&!

Vamos implantar isso **já**!

## Configurando o Prometheus Node Exporter TLS

A documentação é tão extensa que eu vou copiar integralmente aqui:

```
TLS endpoint
** EXPERIMENTAL **

The exporter supports TLS via a new web configuration file.

./node_exporter --web.config=web-config.yml

See the https package for more details.
```

Ok, eles criaram um README.md no diretório com o código. 

As orientações aqui acompanham um modelo de arquivo de configuração (o tal web-config.yml) que é indicado no exemplo acima, bem como um arquivo com a configuração mínima necessária:

```
# web-config.yml
# Minimal TLS configuration example. Additionally, a certificate and a key file
# are needed.
tls_server_config:
  cert_file: server.crt
  key_file: server.key
```

A configuração acima configura o endpoint TLS. Mas, obviamente, não faz qualquer tipo de restrição ao acesso - apenas protege as informações usando criptografia. 

O que, no nosso caso, potencializa o ataque de DoS com o consumo extra de CPU do TLS! Só isso não nos serve! Precisamos **limitar** o acesso a um conjunto de certificados digitais.

Observando o arquivo de configuração de exemplo, temos o seguinte trecho que interessa:

```
  # Server policy for client authentication. Maps to ClientAuth Policies.
  # For more detail on clientAuth options: [ClientAuthType](https://golang.org/pkg/crypto/tls/#ClientAuthType)
  [ client_auth_type: <string> | default = "NoClientCert" ]
```

Ok, é isso que estamos procurando: **ClientAuth**. A opção padrão, naturalmente, é *NoClientCert*, ou seja, não fazer qualquer tipo de exigência, como observamos no exemplo acima.

Quais são os valores aceitos por este parâmetro?

Bem, era querer demais que estivesse nessa documentação, não é mesmo? Siga o link e olhe direto na biblioteca, seu preguiçoso!

Ao clicar no [link](https://golang.org/pkg/crypto/tls/#ClientAuthType) descrito no modelo de arquivo de configuração, somos recebidos por uma página com a seguinte informação:

```
// ClientAuthType declares the policy the server will follow for TLS Client Authentication.

type ClientAuthType int
const (
    NoClientCert ClientAuthType = iota
    RequestClientCert
    RequireAnyClientCert
    VerifyClientCertIfGiven
    RequireAndVerifyClientCert
)
```

Opa, estes devem ser os valores válidos para a configuração, certo?

**Errado**!

```
$ cat /etc/prometheus/web-config.yml 

tls_server_config:
  cert_file: /etc/ssl/private/tls.crt
  key_file: /etc/ssl/private/tls.key
  client_auth_type: "RequireAnyClientCert"
  client_ca_file: /etc/ssl/private/tls.ca

$ systemctl restart prometheus-node-exporter

$ journalctl -fu prometheus-node-exporter
...
Oct 29 23:37:21host.intranet docker[59791]: level=error ts=2020-10-30T02:37:21.312Z caller=node_exporter.go:194 err="Invalid ClientAuth: RequireAnyClientCert"
Oct 29 23:37:21 host.intranet systemd[1]: prometheus-node-exporter.service: Main process exited, code=exited, status=1/FAILURE
```

Aproveitando que estamos aqui, **nem tente** remover o o parâmetro *client_auth_type* e deixar o *client_ca_file* no arquivo de configuração, você vai receber este erro aqui:

```
Oct 29 23:40:08 host.intranet docker[61689]: level=error ts=2020-10-30T02:40:08.981Z caller=node_exporter.go:194 err="Client CA's have been configured without a Client Auth Policy"
```

O que nem é condenável. Por que você especificaria uma CA sem especificar um Client Auth? Seu idiota!

Enfim, quais são os parâmetros?

A melhor maneira de descobrir isso, obviamente, é **olhando no código**:

* https://github.com/prometheus/node_exporter/blob/master/https/tls_config.go#L145

```
	switch c.ClientAuth {
	case "RequestClientCert":
		cfg.ClientAuth = tls.RequestClientCert
	case "RequireClientCert":
		cfg.ClientAuth = tls.RequireAnyClientCert
	case "VerifyClientCertIfGiven":
		cfg.ClientAuth = tls.VerifyClientCertIfGiven
	case "RequireAndVerifyClientCert":
		cfg.ClientAuth = tls.RequireAndVerifyClientCert
	case "", "NoClientCert":
		cfg.ClientAuth = tls.NoClientCert
	default:
		return nil, errors.New("Invalid ClientAuth: " + c.ClientAuth)
	}

	if c.ClientCAs != "" && cfg.ClientAuth == tls.NoClientCert {
		return nil, errors.New("Client CA's have been configured without a Client Auth Policy")
	}
```

Agora sim, sabemos **exatamente** quais são os parâmetros aceitos! E descobrimos que os caras do Prometheus Node Exporter queriam apenas zoar com a nossa cara:

* Para usar tls.RequestClientCert, use "RequestClientCert";
* Para usar  tls.VerifyClientCertIfGiven, configure "VerifyClientCertIfGiven";
* Para usar tls.RequireAndVerifyClientCert, configure "RequireAndVerifyClientCert";

Agora:

* Para usar tls.*Require**Any**ClientCert*, use "RequireClientCert" - sacaram a fuleragem aqui?

Pffff! 

Passada esta etapa, a segunda pergunta: qual a diferença entre eles?

É possível cavocar isso em lugares obscuros, mas eu vou sugerir lemos a incrível documentação escrita pelo [projeto Traefik](https://doc.traefik.io/traefik/https/tls/) a respeito:

```
The clientAuth.clientAuthType option governs the behaviour as follows:

* NoClientCert: disregards any client certificate.

* RequestClientCert: asks for a certificate but proceeds anyway if none is provided.

* RequireAnyClientCert: requires a certificate but does not verify if it is signed by a CA listed in clientAuth.caFiles.

* VerifyClientCertIfGiven: if a certificate is provided, verifies if it is signed by a CA listed in clientAuth.caFiles. Otherwise proceeds without any certificate.

* RequireAndVerifyClientCert: requires a certificate, which must be signed by a CA listed in clientAuth.caFiles.
```

Basicamente:

* **RequestClientCert**: "exige" o certificado do cliente, mas se o cliente não enviar, tudo bem (?!);
* **RequireAnyClientCert**: "exige" o certificado, dá erro se nenhum for enviado, mas **não verifica** se ele é assinado pela CA especificada em client_ca_file! 
* **VerifyClientCertIfGiven**: se o usuário enviar um cerfificado, ele é verificado contra a CA especificada. Se não, prossegue sem. (?!?!?!)
* **RequireAndVerifyClientCert**: exige E valida o certificado. Ufa!

Então, se o objetivo é limitar o acesso a usuários com certificados válidos assinados pela CA, apenas o último parâmetro é efetivamente útil.

## Restrição via subject (dn) do certificado

A minha solução baseada em Nginx tornava possível a autorização do acesso aos endpoints das métricas **apenas para o certificado com um determinado subject** (no caso, o certificado configurado no servidor Prometheus responsável pelo scraping).

Para aqueles curiosos de como fazer isso no Nginx, segue a dica:

```
# trechos de um arquivo de configuração de nginx:

http {
    
  map $ssl_client_s_dn $is_allowed {
      default no;
      "CN=prometheus-scraper" yes;
  }

...

    location / {
 ...
      if ($is_allowed = no) {
        add_header X-SSL-Client-S-DN $ssl_client_s_dn always;
        return 403;
      }
...      
```

Infelizmente isso **não é possível** de ser feito no Prometheus Node Exporter.

Se alguém acha que a validação por certificado não é forte o suficiente, existe a alternativa de configurar um par usuário/senha para complementar a autenticação como descrito na própria documentação:

```
basic_auth_users:
  [ <string>: <secret> ... ]
```

Brincadeira divertida!




