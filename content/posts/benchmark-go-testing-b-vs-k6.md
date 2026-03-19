---
title: "Benchmark em Go: testing.B vs k6 - quando usar cada um"
date: 2026-03-18T10:00:00-03:00
draft: false
---

Estávamos no meio de um refinamento quando a dúvida apareceu: a implementação que estávamos discutindo poderia degradar a performance do endpoint. O time ficou naquele silêncio de quem reconhece o risco, mas não sabe exatamente como medir. Sugeri fazermos um teste de benchmark para tirar a dúvida antes de seguir em frente. Algumas pessoas concordaram, comentaram sobre "validar a performance", e seguimos com o refinamento. Na hora, confesso que não dei muita importância para os termos que cada um usou.

O tempo passou, o código foi implementado, mas a tarefa de benchmark ficou lá, parada no board, sem ninguém pegar. Até que um colega perguntou: "Já fizemos teste no k6?"

Foi aí que eu percebi a confusão. k6 é uma ferramenta de teste de performance e carga, não de benchmark. São coisas diferentes. Expliquei a diferença ali mesmo, e ficou claro para todo mundo, mas a verdade é que a maioria do time não sabia distinguir um do outro. E, pensando bem, eu já tinha visto essa mesma confusão acontecer em outros times e outras empresas.

Decidi então escrever este artigo. A ideia é simples: explicar com exemplos do dia a dia o que é benchmark no Go, como usar o `testing.B` na prática e quando a ferramenta certa é o k6, não o `go test -bench`.

---

## O que é um Benchmark?

Um benchmark, no contexto do pacote `testing` do Go, é uma função que mede o custo de uma operação específica em isolamento. Custo aqui significa tempo de CPU e alocações de memória para executar uma função ou trecho de código determinado número de vezes.

O pacote `testing` da biblioteca padrão do Go fornece o tipo `testing.B`, que controla o loop de iteração, aquece o runtime e coleta métricas automaticamente. A documentação oficial está em [pkg.go.dev/testing](https://pkg.go.dev/testing).

**O que um benchmark mede:**
- Nanossegundos por operação (`ns/op`)
- Bytes alocados por operação (`B/op`)
- Número de alocações por operação (`allocs/op`)

**O que um benchmark não mede:**
- Comportamento do sistema com múltiplos usuários simultâneos
- Latência de rede
- Saturação de banco de dados
- Degradação sob carga real

Resumindo: benchmark mede o custo de uma função em isolamento. Teste de carga mede o comportamento de um sistema sob pressão.

---

## Como usar testing.B na prática

### O loop de benchmark: b.Loop() vs b.N

A partir do Go 1.24, a forma recomendada de escrever o loop de benchmark é com `b.Loop()`. A documentação oficial afirma: *"New benchmarks should prefer using B.Loop, which is more robust and more efficient."*

```go
// Forma moderna (Go 1.24+) — recomendada
func BenchmarkExemplo(b *testing.B) {
    for b.Loop() {
        // código a medir
    }
}

// Forma legada (compatível com todas as versões)
func BenchmarkExemploLegado(b *testing.B) {
    for i := 0; i < b.N; i++ {
        // código a medir
    }
}
```

`b.Loop()` traz três vantagens sobre `b.N`:
1. Gerencia o timer automaticamente na primeira chamada e ao retornar `false`, tornando `b.ResetTimer()` desnecessário na maioria dos casos
2. Previne dead-code elimination nativamente, sem precisar atribuir resultados a variáveis de pacote
3. É mais eficiente internamente, evitando overhead do loop `b.N`

**Limitação importante:** `b.Loop()` não deve ser combinado com `b.StopTimer()` e `b.StartTimer()` dentro do loop. `b.Loop()` já gerencia o timer automaticamente, e usar os dois juntos produz comportamento não documentado — não há erro em tempo de execução, mas os resultados podem ser inconsistentes. Se você precisa pausar o timer dentro de cada iteração (por exemplo, para fazer um setup por iteração que não deve ser medido), use o estilo `b.N` com controle manual do timer.

Use `b.N` apenas se precisar manter compatibilidade com Go anterior a 1.24 ou se precisar de `b.StopTimer()`/`b.StartTimer()` dentro do loop.

---

### Exemplo básico: medindo uma função de concatenação

```go
package stringutil_test

import (
	"strings"
	"testing"
)

// BenchmarkConcatBuilder mede o custo de concatenar strings com strings.Builder
func BenchmarkConcatBuilder(b *testing.B) {
	for b.Loop() {
		var sb strings.Builder
		sb.WriteString("Olá, ")
		sb.WriteString("mundo!")
		_ = sb.String()
	}
}

// BenchmarkConcatPlus mede o custo de concatenar strings com o operador +
func BenchmarkConcatPlus(b *testing.B) {
	for b.Loop() {
		s := "Olá, "
		s += "mundo!"
		_ = s
	}
}
```

Execute com:

```bash
go test -bench=. -benchmem ./...
```

> **Nota sobre o exemplo:** com apenas duas concatenações, o operador `+` pode ser mais rápido que `strings.Builder`. A vantagem do Builder aparece quando há muitas concatenações em sequência, pois ele evita alocações intermediárias. Use esse benchmark para aprender a mecânica, não para tirar conclusões sobre qual é mais rápido em geral.

---

### Exemplo intermediário: setup fora da medição

Quando você precisa preparar dados antes da medição, use `b.ResetTimer()` (com o estilo `b.N`) para evitar contaminação dos resultados. Com `b.Loop()`, o timer é controlado automaticamente:

```go
package codec_test

import (
	"encoding/json"
	"testing"
)

type Produto struct {
	ID    int     `json:"id"`
	Nome  string  `json:"nome"`
	Preco float64 `json:"preco"`
}

// BenchmarkJSONMarshal mede apenas o custo da serialização, excluindo a preparação
func BenchmarkJSONMarshal(b *testing.B) {
	// Preparação fora do loop: não entra na medição com b.Loop()
	produto := Produto{ID: 1, Nome: "Teclado Mecânico", Preco: 349.90}

	for b.Loop() {
		_, err := json.Marshal(produto)
		if err != nil {
			b.Fatal(err)
		}
	}
}

// BenchmarkJSONUnmarshal mede o custo da desserialização
func BenchmarkJSONUnmarshal(b *testing.B) {
	produto := Produto{ID: 1, Nome: "Teclado Mecânico", Preco: 349.90}
	dados, err := json.Marshal(produto)
	if err != nil {
		b.Fatal(err)
	}

	for b.Loop() {
		var p Produto
		if err := json.Unmarshal(dados, &p); err != nil {
			b.Fatal(err)
		}
	}
}
```

---

### Exemplo avançado: sub-benchmarks com tabela

Sub-benchmarks permitem comparar variantes de uma mesma operação em uma única execução, seguindo o mesmo padrão de table-driven tests que você já usa com `testing.T`:

```go
package busca_test

import (
	"sort"
	"testing"
)

// gerarSlice cria uma fatia ordenada de inteiros para os benchmarks
func gerarSlice(n int) []int {
	s := make([]int, n)
	for i := range s {
		s[i] = i * 2
	}
	return s
}

// BenchmarkBuscaBinaria compara buscas em fatias de tamanhos diferentes
func BenchmarkBuscaBinaria(b *testing.B) {
	tamanhos := []struct {
		nome string
		n    int
	}{
		{"1k", 1_000},
		{"10k", 10_000},
		{"100k", 100_000},
	}

	for _, tc := range tamanhos {
		// Nota: a partir de Go 1.22 (com 'go 1.22' no go.mod), a captura
		// de variável de loop é automática. Em módulos com go.mod declarando
		// versão anterior a 1.22, o idioma 'tc := tc' ainda é necessário.
		b.Run(tc.nome, func(b *testing.B) {
			dados := gerarSlice(tc.n)
			alvo := tc.n
			for b.Loop() {
				_ = sort.SearchInts(dados, alvo)
			}
		})
	}
}
```

Execute somente os sub-benchmarks de 10k com:

```bash
go test -bench=BenchmarkBuscaBinaria/10k -benchmem ./...
```

---

## Interpretando os resultados do go test -bench

Uma saída típica se parece com isto:

```
goos: linux
goarch: amd64
pkg: github.com/usuario/projeto/stringutil
cpu: Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
BenchmarkConcatBuilder-8     12485632     95.42 ns/op    48 B/op    2 allocs/op
BenchmarkConcatPlus-8        14203041     84.51 ns/op    16 B/op    1 allocs/op
PASS
ok   github.com/usuario/projeto/stringutil  3.214s
```

Coluna por coluna:

| Coluna | Significado |
|---|---|
| `BenchmarkConcatBuilder-8` | Nome da função e número de CPUs usadas (GOMAXPROCS) |
| `12485632` | Número de iterações executadas |
| `95.42 ns/op` | Tempo médio por operação em nanossegundos |
| `48 B/op` | Bytes alocados no heap por operação (requer `-benchmem`) |
| `2 allocs/op` | Número de alocações no heap por operação (requer `-benchmem`) |

**Como comparar resultados entre versões do código:**

Use a ferramenta `benchstat`, disponível em [pkg.go.dev/golang.org/x/perf/cmd/benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat):

```bash
# Instale benchstat
go install golang.org/x/perf/cmd/benchstat@latest

# Salve os resultados da versão antiga (10 execuções para confiabilidade estatística)
go test -bench=. -benchmem -count=10 ./... > antes.txt

# Após sua mudança, salve os novos resultados
go test -bench=. -benchmem -count=10 ./... > depois.txt

# Compare estatisticamente
benchstat antes.txt depois.txt
```

A documentação do `benchstat` recomenda pelo menos 10 execuções (`-count=10`) para resultados estatisticamente confiáveis. Uma única execução, ou mesmo três, pode ser ruidosa o suficiente para mascarar diferenças reais.

---

## Flags úteis do go test -bench

Além de `-bench`, o `go test` oferece flags que ampliam o controle sobre a execução e a saída dos benchmarks:

| Flag | O que faz | Exemplo |
|---|---|---|
| `-benchmem` | Inclui métricas de alocação de memória na saída | `go test -bench=. -benchmem` |
| `-benchtime` | Define a duração mínima ou o número exato de iterações | `-benchtime=5s` ou `-benchtime=1000x` |
| `-cpu` | Executa o benchmark com diferentes valores de GOMAXPROCS | `-cpu=1,2,4,8` |
| `-count` | Repete o benchmark N vezes (essencial para `benchstat`) | `-count=10` |
| `-cpuprofile` | Gera um perfil de CPU para análise com `pprof` | `-cpuprofile=cpu.prof` |
| `-memprofile` | Gera um perfil de memória para análise com `pprof` | `-memprofile=mem.prof` |
| `-run` | Define quais testes de unidade rodar (use `^$` para pular todos e rodar só benchmarks) | `-run=^$ -bench=.` |

A flag `-benchtime=1000x` (com o sufixo `x`) é especialmente útil quando você quer um número fixo de iterações em vez de um tempo mínimo, por exemplo para benchmarks que envolvem I/O e demoram mais por iteração.

A combinação `-run=^$ -bench=.` é um padrão comum: o `-run=^$` não casa com nenhum teste de unidade, garantindo que apenas os benchmarks sejam executados.

---

## Boas práticas e armadilhas comuns

### Boas práticas

**1. Use `b.Loop()` em projetos Go 1.24+**

É mais simples, mais seguro e é o padrão recomendado oficialmente. Elimina a necessidade de `b.ResetTimer()` e de variáveis de pacote para prevenir dead-code elimination.

**2. Sempre use `-benchmem`**

Otimizações de tempo frequentemente escondem regressões de memória. Meça os dois.

**3. Use `-count=10` com benchstat**

Uma execução pode sofrer interferência de outros processos. Use pelo menos 10 execuções e `benchstat` para análise estatística confiável.

**4. Fixe o ambiente**

Resultados variam entre máquinas. Se possível, rode benchmarks em CI com hardware dedicado e compare apenas resultados da mesma máquina.

**5. Com `b.N`: evite dead-code elimination**

Se você ainda usa o estilo `b.N`, o compilador Go pode eliminar operações cujos resultados não são usados. Com `b.Loop()` isso é tratado automaticamente; com `b.N`, atribua resultados a uma variável de pacote:

```go
// Com b.N: atribuição a variável de pacote previne eliminação pelo compilador
var resultado string

func BenchmarkHashCorreto(b *testing.B) {
	var r string
	for i := 0; i < b.N; i++ {
		r = calcularHash("entrada")
	}
	resultado = r
}

// Com b.Loop(): não precisa dessa técnica
func BenchmarkHashModerno(b *testing.B) {
	for b.Loop() {
		calcularHash("entrada") // b.Loop() previne eliminação nativamente
	}
}
```

### Armadilhas comuns

**Setup dentro do loop:** preparação de dados dentro do loop distorce a medição. Com `b.Loop()`, coloque o setup antes do loop. Com `b.N`, use `b.ResetTimer()` ou `b.StopTimer()` / `b.StartTimer()`. Se o setup precisa acontecer a cada iteração (e não apenas uma vez antes do loop), você precisará do estilo `b.N` com `b.StopTimer()` / `b.StartTimer()`, já que `b.Loop()` não suporta pausar o timer dentro do loop.

**Confiar em um único número:** `ns/op` sozinho não conta a história completa. Uma função 10% mais rápida que aloca o dobro de memória pode ser um passo atrás em sistemas com pressão de GC.

**Poucas execuções:** use `-count=10` e `benchstat`. Uma única execução não é estatisticamente confiável.

---

## Quando usar k6 em vez de testing.B

São ferramentas para perguntas diferentes. A tabela abaixo resume a distinção:

| Critério | `testing.B` | k6 |
|---|---|---|
| O que mede | Custo de uma função isolada | Comportamento do sistema sob carga |
| Granularidade | Nanossegundos por operação | Requisições por segundo, latência de percentil |
| Unidade testada | Função ou pacote Go | Endpoint HTTP, serviço completo |
| Dependências externas | Nenhuma (testes em isolamento) | Banco de dados, rede, infraestrutura real |
| Usuários simulados | 1 (execução sequencial) | Dezenas a milhares de usuários virtuais |
| Integração com CI | `go test` nativo | Script JS separado, requer runtime k6 |
| Responde à pergunta | "Quantos ns esta função consome?" | "Minha API aguenta 500 req/s?" |

**Use `testing.B` quando:**
- Você quer comparar dois algoritmos ou estruturas de dados
- Você precisa validar que uma refatoração não introduziu regressão de performance
- Você quer medir o impacto de uma mudança no uso de memória
- Você está desenvolvendo uma biblioteca e quer publicar benchmarks reproduzíveis

**Use k6 quando:**
- Você quer saber como sua API se comporta com 200 usuários simultâneos
- Você quer medir latência de p95 e p99 em condições reais
- Você quer fazer testes de stress, soak ou spike em um sistema distribuído
- Você quer validar SLAs antes de ir para produção

As duas ferramentas são complementares. Benchmarks garantem que o código Go é eficiente; testes de carga garantem que o sistema como um todo se comporta bem sob pressão.

---

## Próximo passo: profiling com pprof

Benchmarks mostram *quanto* uma operação custa, mas não mostram *onde* o tempo é gasto. Quando um benchmark revela que uma função está mais lenta do que o esperado, o próximo passo natural é usar `pprof` para identificar os gargalos.

O `go test` já integra com `pprof` nativamente. Para gerar um perfil de CPU durante um benchmark:

```bash
go test -bench=BenchmarkJSONMarshal -cpuprofile=cpu.prof ./...
go tool pprof cpu.prof
```

Para um perfil de memória:

```bash
go test -bench=BenchmarkJSONMarshal -memprofile=mem.prof ./...
go tool pprof mem.prof
```

Dentro do `pprof`, o comando `top` mostra as funções que mais consomem recursos, e `web` gera um grafo visual (requer Graphviz instalado). Para uma interface web interativa, use:

```bash
go tool pprof -http=:8080 cpu.prof
```

A documentação oficial de profiling está em [go.dev/doc/diagnostics](https://go.dev/doc/diagnostics).

---

## Conclusão

A confusão entre benchmark e teste de carga tem uma raiz simples: as duas respondem à palavra "performance." Mas performance de uma função e performance de um sistema são grandezas diferentes.

**Leve estes pontos para o próximo projeto:**

1. Use `b.Loop()` em projetos Go 1.24+: é mais simples e é o padrão recomendado.
2. Sempre use `-benchmem` para capturar métricas de alocação.
3. Use `-count=10` e `benchstat` para comparações estatisticamente válidas.
4. Com `b.N`: impeça dead-code elimination atribuindo resultados a variáveis de pacote.
5. k6 entra depois que o código já está otimizado, para validar o sistema sob carga real.

Um time maduro usa as duas ferramentas. Benchmarks antes do merge, testes de carga antes do deploy em produção.

---

## Para saber mais

- [pkg.go.dev/testing](https://pkg.go.dev/testing) — documentação oficial do pacote `testing`, incluindo `testing.B` e `b.Loop()`
- [pkg.go.dev/testing#hdr-Benchmarks](https://pkg.go.dev/testing#hdr-Benchmarks) — referência direta à seção de benchmarks
- [pkg.go.dev/golang.org/x/perf/cmd/benchstat](https://pkg.go.dev/golang.org/x/perf/cmd/benchstat) — ferramenta oficial de comparação estatística de benchmarks
- [go.dev/doc/diagnostics](https://go.dev/doc/diagnostics) — guia oficial de diagnóstico e profiling em Go
- [go.dev/blog/all](https://go.dev/blog/all) — todos os artigos do blog oficial do Go
