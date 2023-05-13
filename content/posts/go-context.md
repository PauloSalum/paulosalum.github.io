---
title: "Context em Go: Um Guia Aprofundado"
date: 2023-05-13T02:47:17-03:00
draft: false
---
Olá, pessoal! Hoje vamos falar sobre um tópico muito importante no universo Go: o pacote `context`. Ele é uma ferramenta poderosa que nos permite gerenciar e controlar o tempo de vida de processos e operações. Vamos mergulhar nesse assunto!

## Entendendo o Context

Em Go, `context` é um pacote que nos permite passar valores de escopo de solicitação, prazos de cancelamento e sinalização em toda a pilha de chamadas. Ele é projetado para uso em solicitações recebidas em um servidor e é passado para funções que precisam acessar esses valores.

## Criando e Passando Context

Para começar a usar o `context`, precisamos criar um. Temos duas funções principais para isso: `context.Background()` e `context.TODO()`. A primeira é usada para criar um novo contexto, que é o contexto pai de todos os outros. A segunda é usada quando não está claro qual contexto deve ser usado ou se o contexto ainda não está disponível.

Depois de criar um contexto, podemos passá-lo para outras funções que precisam dele. O contexto é geralmente o primeiro parâmetro passado para uma função.

```go
func DoSomething(ctx context.Context, arg Arg) error {
    // ...
}
```

## Cancelando Operações com Context

Um dos usos mais comuns do `context` é para cancelar operações de longa duração. Podemos criar um contexto que pode ser cancelado usando a função `context.WithCancel(parentContext)`.

```go
ctx, cancel := context.WithCancel(context.Background())
```

A função `WithCancel` retorna um novo contexto e uma função `cancel`. Quando chamamos a função `cancel`, ela envia um sinal para o contexto para parar todas as operações associadas a ele.

## Definindo Prazos e Timeouts com Context

Além do cancelamento, `context` também pode ser usado para definir prazos e timeouts para operações. `context.WithDeadline(parentContext, deadline)` cria um novo contexto que será cancelado automaticamente no tempo de `deadline` fornecido. `context.WithTimeout(parentContext, timeout)` é semelhante, mas cancela o contexto após o `timeout` fornecido.

## Context em Ação

Vamos ver um exemplo de como o `context` pode ser usado em uma aplicação real. Suponha que temos uma função `DoWork` que leva algum tempo para ser concluída. Queremos ser capazes de cancelar `DoWork` se demorar muito tempo.

```go
func DoWork(ctx context.Context) {
    for {
        select {
        case <-time.After(1 * time.Second):
            fmt.Println("Doing some work")
        case <-ctx.Done():
            fmt.Println("Canceling work")
            return
        }
    }
}
```

Neste exemplo, `DoWork` verifica continuamente se o `ctx` foi cancelado. Se foi, ele retorna e para de trabalhar. Caso contrário, ele faz algum trabalho e depois espera um segundo antes de verificar novamente.

## Context e Goroutines

Em Go, é comum iniciar várias goroutines para realizar tarefas em paralelo. No entanto, isso pode levar a situações em que uma goroutine está bloqueada em uma operação que nunca será concluída porque outra goroutine encontrou um erro ou porque o processo inteiro está

## Context e Goroutines

Em Go, é comum iniciar várias goroutines para realizar tarefas em paralelo. No entanto, isso pode levar a situações em que uma goroutine está bloqueada em uma operação que nunca será concluída porque outra goroutine encontrou um erro ou porque o processo inteiro está sendo encerrado. Aqui é onde o `context` realmente brilha.

Ao criar uma nova goroutine, você pode passar um `context` para ela. Essa goroutine pode então passar o mesmo `context` para outras funções e goroutines. Isso cria uma árvore de goroutines que podem ser canceladas ao mesmo tempo.

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    go DoWork(ctx)
    time.Sleep(5 * time.Second) // cancel after 5 seconds
    cancel()
}
```

Neste exemplo, a função `main` cria um novo `context` com cancelamento e inicia `DoWork` em uma nova goroutine, passando o `context` para ela. Depois de dormir por 5 segundos, `main` chama a função `cancel`, que cancela o `context`. Isso faz com que `DoWork` pare de trabalhar.

## Conclusão

O pacote `context` em Go é uma ferramenta essencial para gerenciar e controlar operações de longa duração e simultâneas. Com ele, você pode facilmente passar valores de escopo de solicitação, definir prazos e cancelar operações. Embora possa parecer um pouco complicado no início, com prática e entendimento, ele se tornará uma parte valiosa de sua caixa de ferramentas de desenvolvimento Go.

Lembre-se, o `context` deve ser o primeiro parâmetro de uma função e deve ser passado de função para função. Além disso, nunca deve ser armazenado ou colocado em estruturas globais. Com essas práticas recomendadas em mente, você estará no caminho certo para dominar o `context` em Go.

Espero que este artigo tenha sido útil para você. Fique à vontade para deixar seus comentários e perguntas, e vamos continuar aprendendo juntos! Até a próxima!