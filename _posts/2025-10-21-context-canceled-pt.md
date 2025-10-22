---
title: "Quem Cancelou Meu Contexto HTTP no Golang?"
date: 2025-10-21
---

[Version in English](/blog/2025/10/21/context-canceled-en.html)

![Stop](/blog/docs/assets/gopher_stop_sign.png)

# Pacote context

A [documentação do pacote context](https://pkg.go.dev/context) introduz o conceito de contexto da seguinte forma:

> O pacote context define o tipo Context, que carrega prazos (deadlines), sinais de cancelamento e outros valores com escopo de requisição através de limites de API e entre processos.
> 
> Requisições recebidas por um servidor devem criar um Context, e chamadas de saída para servidores devem aceitar um Context.

Portanto, em handlers HTTP, recuperamos o contexto das requisições HTTP e o propagamos para outros componentes.

Como verificamos corretamente os erros e também registramos erros inesperados nos logs, é provável que cedo encontremos mensagens pouco claras de `context canceled` aparecendo em nossos logs de produção.

```
A consulta ao banco de dados retornou este erro: context canceled
```

Isso sequer é um erro?

# Quem Cancelou Meu Contexto HTTP no Golang?

Ao revisar o código da aplicação, temos certeza de que ela não cancelou o contexto.

Quem cancelou?

Um suspeito comum é a biblioteca que criou o contexto para nós, se houver.

Por exemplo, o pacote de servidor HTTP da biblioteca padrão é conhecido por cancelar requisições sempre que detecta que a conexão do cliente foi quebrada. Como o cliente HTTP já se foi, não faz sentido terminar a requisição.

Mas agora a condição `context canceled` será relatada como erro por qualquer serviço para o qual propagamos o contexto. Não é exatamente um erro real, mas é difícil distingui-lo de um erro verdadeiro porque não fornece a causa.

# Como Evitar Essa Ambiguidade?

A partir do Go 1.20, o pacote `context` fornece uma nova função para criar contextos canceláveis com causas explícitas:

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("a conexão do cliente foi encerrada: %w", err))
```

Essa melhoria nos dá a oportunidade de fornecer educadamente uma causa explícita ao cancelar um contexto. A causa do cancelamento provavelmente será registrada em logs pelo nosso código de verificação de erros, tornando claro o motivo do cancelamento.

E quanto às bibliotecas existentes que cancelam contextos sem fornecer uma causa?

Sim, isso é doloroso. Enquanto esperamos que sejam atualizadas, precisamos estar atentos que mensagens pouco claras de `context canceled` podem aparecer em nossos logs e possivelmente não indicam erros reais!

# Principais Pontos

1\. Sempre que precisar cancelar um contexto, forneça uma causa clara e explícita.

```go
ctx, cancelWithCause := context.WithCancelCause(context.Background())
// ...
cancelWithCause(fmt.Errorf("a conexão do cliente foi encerrada: %w", err))
```

2\. `context.WithCancelCause` foi adicionado no Go 1.20. Bibliotecas anteriores a essa versão provavelmente não fornecem uma causa ao cancelar seus contextos. Portanto, esteja preparado para lidar com erros inesperados de `context canceled` vindos dessas bibliotecas.

3\. O servidor HTTP da biblioteca padrão é conhecido por cancelar contextos sem fornecer a causa. Mas há esperança para o futuro (veja o issue nas referências abaixo).

# Referências

[proposal: net/http: add context cancellation reason for server handlers](https://github.com/golang/go/issues/64465)
