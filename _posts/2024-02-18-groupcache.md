---
title: "Implementando uma aplicação em Go com Groupcache no Kubernetes"
date: 2024-02-18
---

![Groupcache](/blog/docs/assets/groupcache.png)

* [Resumo](#resumo)
* [O que é groupcache?](#o-que-é-groupcache)
  * [1 \- Resolve o problema do "estouro da manada" (thundering herd)](#1---resolve-o-problema-do-estouro-da-manada-thundering-herd)
  * [2 \- Não requer manutenção de um conjunto extra de servidores (cache centralizado)](#2---não-requer-manutenção-de-um-conjunto-extra-de-servidores-cache-centralizado)
* [Mas qual groupcache?](#mas-qual-groupcache)
* [Como utilizar](#como-utilizar)
  * [1/3 \- Declare os peers](#13---declare-os-peers)
  * [2/3 \- Crie um grupo](#23---crie-um-grupo)
  * [3/3 \- Consulte o cache](#33---consulte-o-cache)
* [Exemplo concreto: Kubernetes](#exemplo-concreto-kubernetes)
  * [Proxy: kubecache](#proxy-kubecache)
    * [Descoberta de peers](#descoberta-de-peers)
    * [Métricas do groupcache](#métricas-do-groupcache)
    * [Testando o kubecache](#testando-o-kubecache)
* [Outro exemplo: oauth2 client\-credentials](#outro-exemplo-oauth2-client-credentials)
* [Outras Métricas, Logs e Traces](#outras-métricas-logs-e-traces)
* [Referências](#referências)

# Resumo

Receita sobre como implementar uma aplicação em Go para Kubernetes usando a biblioteca Groupcache.

# O que é groupcache?

Groupcache é uma biblioteca de cache distribuído para Go, criada pelo Google para melhorar o desempenho do site **dl.google.com** e publicada como open source.

Diferente de caches centralizados, como o Redis, o groupcache é uma biblioteca que transforma cada instância da sua aplicação em um nó do cache distribuído. Cada instância da aplicação armazena uma parte do cache. Mas todas as instâncias têm a mesma visão unificada do cache: qualquer instância da aplicação que solicitar uma chave X obterá o mesmo valor.

O groupcache traz dois grandes benefícios:

## 1 - Resolve o problema do "estouro da manada" (thundering herd)

Quando múltiplos clientes buscam concorrentemente uma chave que não está disponível no cache, o **groupcache** coordena o preenchimento do cache em uma única instância que obterá a informação necessária e distribuirá para todas as demais instâncias.

Esse problema é especialmente importante para aplicações extremamente carregadas que dependem de uma chave que acabou de expirar no cache centralizado: se não houver coordenação, todas as instâncias da aplicação recorrerão ao banco de dados (ou outro serviço) para obter a informação atualizada.

## 2 - Não requer manutenção de um conjunto extra de servidores (cache centralizado)

O groupcache é parte da aplicação e escala junto com ela. Ao adicionar novas instâncias à aplicação, a capacidade do cache aumenta proporcionalmente. Não existe a necessidade de implantar e administrar um conjunto separado de servidores/serviços centralizados (como o Redis).

# Mas qual groupcache?

Surgiram alguns forks do **groupcache** para contornar limitações do projeto original.

O **mailgun** adicionou várias melhorias, incluindo suporte a TTL, mas deixou de fora a possibilidade de alocação explícita do estado.

O **galaxycache** fez o contrário: adicionou estado explícito mas não possui TTL.

O **modernprogram** é um fork do **mailgun** que adiciona exclusivamente o suporte a estado explícito.

| Groupcache | Estado explícito | Expiração de chaves |
| --- | --- | --- |
| [google](https://github.com/golang/groupcache) | Não | Não |
| [mailgun](https://github.com/mailgun/groupcache) | Não (*1) | Sim |
| [galaxycache](https://github.com/vimeo/galaxycache) | Sim | Não (*2) |
| [modernprogram](https://github.com/modernprogram/groupcache) | Sim | Sim |

- (*1) [https://github.com/mailgun/groupcache/issues/66](https://github.com/mailgun/groupcache/issues/66)
- (*2) [https://github.com/vimeo/galaxycache/pull/25](https://github.com/vimeo/galaxycache/pull/25)

# Como utilizar

Olhando superficialmente, a utilização do groupcache parece muito simples, em apenas 3 passos.

## 1/3 - Declare os peers

A função `groupcache.NewHTTPPool()` registra as rotas da instância do **groupcache** no http mux padrão do Go, para que todas as instâncias possam se comunicar entre si. O valor `peers` retornado deve ser usado para registrar os URLs de todas as instâncias da aplicação.

**NOTA**: Apesar de não estar explícito nesse guia de 3 passos, o **groupcache** espera que a aplicação lance um servidor HTTP usando o mux padrão.

```
me := "http://10.0.0.1"
peers := groupcache.NewHTTPPool(me)

// Quando os peers mudarem:
peers.Set("http://10.0.0.1", "http://10.0.0.2", "http://10.0.0.3")
```

## 2/3 - Crie um grupo

Observe que a criação do grupo requer uma função de preenchimento. Quando a chave (key) não for encontrada no cache, o **groupcache** automaticamente utilizará a função de preenchimento para obter o conteúdo, que será guardado no cache e devolvido para o invocador.

```
var thumbNails = groupcache.NewGroup("thumbnail", 64<<20, groupcache.GetterFunc(
    func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
        fileName := key
        dest.SetBytes(generateThumbnail(fileName))
        return nil
    }))
```

## 3/3 - Consulte o cache

Quando a aplicação precisar da informação, ela consultará o cache usando a função `cache.Get(...)`. `cache` é a variável que armazena o cache, como `thumbNails` no exemplo abaixo. É responsabilidade do groupcache recuperar ou gerar a informação que ainda não estiver cacheada.

```
var data []byte
err := thumbNails.Get(ctx, "big-file.jpg",
    groupcache.AllocatingByteSliceSink(&data))
```

# Exemplo concreto: Kubernetes

O exemplo anterior, da seção "Como utilizar", foi retirado da documentação do **groupcache**. Ele está correto, porém é insuficiente para implementar uma aplicação concreta.

Para ilustrar a utilização do **groupcache** em um cenário mais realista, consideremos o caso abaixo.

## Proxy: kubecache

Objetivo: Usar o **groupcache** para criar uma aplicação proxy, chamada **kubecache**, que faz cache de requisições HTTP GET para um serviço de backend. A aplicação **kubecache** encaminhará todas as requisições HTTP GET que receber para um servidor HTTP de backend. Cada requisição respondida ficará retida no cache por 5 minutos. O ambiente de implantação será Kubernetes: rodam no cluster K8S as aplicações consumidoras, bem como o próprio **kubecache**, e também o serviço de backend a que o **kubecache** encaminhará as requisições não encontradas no cache.

De posse desse objetivo, é possível examinar os detalhes de implementação do **kubecache**, a seguir.

### Descoberta de peers

O **groupcache** oferece a API abaixo para definir todas as instâncias que compõem o cache distribuído.

```
peers.Set("http://10.0.0.1", "http://10.0.0.2", "http://10.0.0.3")
```

No exemplo dado, há 3 instâncias e cada uma delas deve executar exatamente a mesma declaração de peers.

O **groupcache** não entra no mérito de como as instâncias peers são descobertas, porque esse detalhe depende do ambiente de implantação: a descoberta de servidores físicos é diferente da de máquinas virtuais, que difere dos servidores de nuvem, que difere dos PODs do Kubernetes, e assim por diante.

Para o caso concreto do **kubecache**, o ambiente de implantação foi definido como Kubernetes, então usamos o projeto [kubegroup](https://github.com/udhos/kubegroup) que faz a descoberta automática de peers.

Abaixo, um breve exemplo de como acionar o **kubegroup**. Para um exemplo funcional do uso do **kubegroup**, vide o código-fonte: [ativando o kubegroup no kubecache](https://github.com/udhos/kubecache/blob/main/cmd/kubecache/groupcache.go#L52).

```
options := kubegroup.Options{
    Pool:              peers, // criado pelo groupcache.NewHTTPPool(...)
    GroupCachePort:    ":5000",
    Debug:             true,
}

kg, errKg := kubegroup.UpdatePeers(options)
if errKg != nil {
    log.Fatal().Msgf("kubegroup error: %v", errKg)
}
```

Quando ativado, o **kubegroup** automaticamente expõe no contexto do Prometheus da aplicação duas métricas sobre a descoberta de peers:

```
kubegroup_peers: quantidade de peers (PODs) descobertos
kubegroup_changes: alteração de peers recebidas
```

### Métricas do groupcache

Por ser independente do ambiente de implantação, o **groupcache** não expõe métricas em nenhum formato específico. Apenas cria contadores internos que podem ser consultados pela aplicação para exposição de métricas no formato desejado.

Para expor as métricas do **groupcache** no formato do Prometheus, usamos a biblioteca [groupcache_exporter](https://github.com/udhos/groupcache_exporter).

A utilização do **groupcache_exporter** é ilustrada abaixo. Para ver como o **groupcache_exporter** é ativado no **kubecache**, consulte: [groupcache_exporter no kubecache](https://github.com/udhos/kubecache/blob/main/cmd/kubecache/groupcache.go#L152).

```
g := modernprogram.New(app.cache)
labels := map[string]string{}
namespace := ""
collector := groupcache_exporter.NewExporter(namespace, labels, g)
prometheus.MustRegister(collector)
```

Uma vez ativado, o **groupcache_exporter** exporá as seguintes métricas no formato do Prometheus:

```
groupcache_cache_bytes
groupcache_cache_bytes
groupcache_cache_evictions_total
groupcache_cache_evictions_total
groupcache_cache_gets_total
groupcache_cache_gets_total
groupcache_cache_hits_total
groupcache_cache_hits_total
groupcache_cache_items
groupcache_cache_items
groupcache_gets_total
groupcache_hits_total
groupcache_loads_deduped_total
groupcache_loads_total
groupcache_local_load_errs_total
groupcache_local_load_total
groupcache_peer_errors_total
groupcache_peer_loads_total
groupcache_server_requests_total
```

A descrição de todas as métricas expostas pelo **groupcache_exporter** está disponível neste arquivo: [exporter.go](https://github.com/udhos/groupcache_exporter/blob/main/exporter.go)

### Testando o kubecache

O **kubecache** é uma aplicação real criada somente para demonstrar a utilização do **groupcache** no Kubernetes. Está preparada para ser facilmente instalada em um cluster Kubernetes para experimentações.

Por exemplo, o helm pode ser usado para instalar:

```
helm repo add kubecache https://udhos.github.io/kubecache

helm upgrade kubecache kubecache/kubecache --install --values values.yaml
```

**NOTA**: Para brincar com o **kubecache**, provavelmente será necessário customizar a variável de ambiente `BACKEND_URL` usando o `values.yaml`, para apontar para o serviço de backend dentro do cluster Kubernetes.

```
#
# Trecho do values.yaml com as variáveis de ambiente.
#
configMapProperties:
  #SECRET_ROLE_ARN: ""
  #DEBUG_LOG: "true"
  #LISTEN_ADDR: ":9000"
  #BACKEND_URL: "http://config-server:9000"
  #BACKEND_TIMEOUT: 300s
  #CACHE_TTL: 300s
```

Assim que o **kubecache** estiver rodando no Kubernetes com a variável `BACKEND_URL` apontando para o serviço de backend, será possível testar a chamada a partir de um POD com o `curl`:

```
curl kubecache:9000/rota/do/backend
```

Para mais informações, visite o projeto: [https://github.com/udhos/kubecache](https://github.com/udhos/kubecache)

# Outro exemplo: oauth2 client-credentials

Uma vez que uma aplicação estiver preparada com o groupcache, ela pode ser facilmente extendida para criar vários grupos para armazenar diferentes tipos de informações.

Por exemplo, o projeto [groupcache_oauth2](https://github.com/udhos/groupcache_oauth2) oferece um plugin para fazer requisições para endpoints protegidos pelo fluxo client-credentials do oauth2. Automaticamente, os tokens são recuperados, armazenados no **groupcache** e renovados conforme necessário.

# Outras Métricas, Logs e Traces

A aplicação **kubecache** também expõe métricas de requisições HTTP no formato do Prometheus, registra logs utilizando o pacote [zerolog](https://github.com/rs/zerolog) e está instrumentada para envio de traces com o [OpenTelemetry](https://opentelemetry.io/docs/languages/go/instrumentation/).

O **groupcache** não causa nenhuma interação especial com esses outros itens de observability.

# Conclusão

O **groupcache** oferece uma alternativa interessante de cache distribuído para aplicações Go, com o benefício de tratar adequadamente o problema de Thundering Herd.

O roteiro apresentado nesse documento ilustra em detalhes como implementar o **groupcache** para implantação em ambiente Kubernetes, endereçando os temas de descoberta automática de peers (PODs) e de exposição de métricas do cache distribuído no formato do Prometheus.

# Referências

As duas referências abaixo são ótimas documentações para aprofundamento no **groupcache**.

[1] [dl.google.com: Powered by Go](https://go.dev/talks/2013/oscon-dl.slide#1)

[2] [GroupCache: The superior Golang cache](https://www.mailgun.com/blog/it-and-engineering/golangs-superior-cache-solution-memcached-redis/)
