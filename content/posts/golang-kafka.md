---
title: "Integração entre Golang e Kafka"
date: 2023-05-03T16:40:21-03:00
draft: false
---

Neste artigo, abordaremos a integração entre duas tecnologias populares no desenvolvimento de sistemas distribuídos: Golang e Kafka. Golang é uma linguagem de programação de alto desempenho e fácil aprendizado, ideal para desenvolvimento de aplicações concorrentes. Kafka é uma plataforma distribuída de streaming de eventos que permite a publicação e assinatura de eventos em tempo real.

A integração entre Golang e Kafka oferece diversas vantagens, como a capacidade de processar grandes volumes de dados em tempo real, escalabilidade, tolerância a falhas e facilidade de integração com outros sistemas.

## Começando com Golang e Kafka

### Instalando Golang e Kafka

Para começar a integrar Golang e Kafka, é necessário instalar ambos em seu ambiente de desenvolvimento. A instalação do Golang pode ser realizada seguindo as instruções disponíveis no site oficial da linguagem. Já a instalação do Kafka envolve o download do Apache Kafka e a instalação do Zookeeper, que gerencia o cluster Kafka.

Abaixo vemos um exemplo a ser executado no prompt de comando para instalação do Go e Kafka em maquina local.

```bash
# Instalar Golang
$ wget https://golang.org/dl/go1.17.1.linux-amd64.tar.gz
$ tar -C /usr/local -xzf go1.17.1.linux-amd64.tar.gz
$ export PATH=$PATH:/usr/local/go/bin

# Instalar Kafka
$ wget https://downloads.apache.org/kafka/2.8.2/kafka_2.13-2.8.2.tgz
$ tar -xzf kafka_2.13-2.8.2.tgz
$ cd kafka_2.13-2.8.2
```

### Configurando um cluster Kafka

Após a instalação, é necessário configurar um cluster Kafka. Isso pode ser feito seguindo os passos a seguir:

1. Iniciar o Zookeeper
2. Iniciar os servidores Kafka (brokers)
3. Criar tópicos para armazenar os eventos

```bash
# Iniciar o Zookeeper
$ bin/zookeeper-server-start.sh config/zookeeper.properties

# Iniciar o servidor Kafka
$ bin/kafka-server-start.sh config/server.properties
```
Para criar tópicos novos no Kafka usando a linha de comando, você pode usar a ferramenta `kafka-topics.sh` que acompanha a distribuição do Kafka. Siga os passos abaixo:

1. Abra o terminal (no caso de um cluster remoto, conecte-se via SSH).
2. Navegue até a pasta onde o Kafka está instalado (a pasta do Kafka deve conter a pasta `bin`).
3. Use o seguinte comando para criar um novo tópico:

```sh
./bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic <nome_do_topico>
```

Substitua `<nome_do_topico>` pelo nome desejado para o seu tópico. Além disso, você pode alterar os seguintes parâmetros:

- `--bootstrap-server`: Endereço e porta do servidor Kafka (ex: `localhost:9092` ou `kafka-broker-1:9092`).
- `--replication-factor`: Fator de replicação para o tópico (número de cópias do tópico mantidas no cluster).
- `--partitions`: Número de partições para o tópico (partições permitem o processamento paralelo de mensagens).

Depois de executar o comando, você receberá uma confirmação de que o tópico foi criado com sucesso.

Para listar os tópicos existentes, você pode usar o seguinte comando:

```sh
./bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

Isso listará todos os tópicos disponíveis no servidor Kafka especificado.

Com o cluster Kafka configurado, é possível prosseguir para a criação de produtores e consumidores em Golang.

## Produzindo mensagens com Golang

### Criando um produtor Kafka em Golang

Para criar um produtor Kafka em Golang, é necessário utilizar uma biblioteca cliente Kafka, como a `confluent-kafka-go`. Com a biblioteca instalada, é possível criar uma configuração de produtor e instanciar um objeto de produtor.

```go
import (
	"github.com/confluentinc/confluent-kafka-go/kafka"
)

func createProducer(broker string) (*kafka.Producer, error) {
	config := &kafka.ConfigMap{"bootstrap.servers": broker}
	producer, err := kafka.NewProducer(config)
	if err != nil {
		return nil, err
	}
	return producer, nil
}
```
A função `createProducer` é responsável por criar uma instância de um produtor Kafka. Ela aceita um único parâmetro, `broker`, que é a string contendo o endereço do broker Kafka que será utilizado.

A função começa criando uma instância de `kafka.ConfigMap` e preenchendo-a com a configuração necessária para conectar ao broker Kafka. Neste exemplo, apenas a configuração `"bootstrap.servers"` é definida, usando o valor do parâmetro `broker`.

Em seguida, a função `kafka.NewProducer` é chamada com a configuração criada (`config`). Essa função tenta criar uma nova instância de um produtor Kafka com a configuração fornecida e retorna dois valores: uma instância de `*kafka.Producer` e um valor de erro. Se a criação do produtor for bem-sucedida, o erro retornado será `nil`; caso contrário, o erro conterá informações sobre o problema encontrado.

A função `createProducer` retorna a instância do produtor criada (`producer`) e o erro (`err`). Isso permite que quem chame essa função verifique se ocorreu algum erro durante a criação do produtor e trate-o de acordo. Se a função for bem-sucedida, o produtor Kafka criado poderá ser utilizado para enviar mensagens a tópicos no cluster Kafka.

### Enviando mensagens para um tópico Kafka

Com o produtor criado, é possível enviar mensagens para um tópico Kafka. O produtor utiliza o método `Produce()` para enviar as mensagens, que podem ser strings, bytes ou objetos serializados.
```go
func sendMessage(producer *kafka.Producer, topic, message string) error {
	deliveryChan := make(chan kafka.Event)
	err := producer.Produce(&kafka.Message{
		TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
		Value:          []byte(message),
	}, deliveryChan)
	if err != nil {
		return err
	}
	e := <-deliveryChan
	msg := e.(*kafka.Message)
	return msg.TopicPartition.Error
}
```
A função `sendMessage` é responsável por enviar mensagens a um tópico Kafka usando um produtor Kafka. Ela aceita três parâmetros: `producer`, que é uma instância de `*kafka.Producer` previamente criada; `topic`, que é o tópico Kafka para o qual a mensagem será enviada; e `message`, que é a mensagem a ser enviada.

A função começa criando um canal chamado `deliveryChan`, que será usado para receber eventos relacionados à entrega da mensagem. Em seguida, ela utiliza a função `producer.Produce` para produzir uma nova mensagem Kafka, especificando a partição de destino como `kafka.PartitionAny` (o que significa que o produtor escolherá automaticamente a partição) e o valor da mensagem usando `[]byte(message)` para converter a string `message` em um slice de bytes.

O canal `deliveryChan` é passado como parâmetro para a função `producer.Produce`, que enviará eventos relacionados à entrega da mensagem para esse canal. Se ocorrer um erro ao tentar produzir a mensagem, a função `sendMessage` retorna esse erro.

Após tentar produzir a mensagem, a função aguarda um evento no canal `deliveryChan`. Quando um evento é recebido, a função converte o evento para uma mensagem Kafka usando `msg := e.(*kafka.Message)`.

Por fim, a função retorna o erro armazenado em `msg.TopicPartition.Error`. Se a mensagem foi entregue com sucesso, esse campo será `nil`; caso contrário, ele conterá detalhes sobre o erro que ocorreu durante a entrega da mensagem.

Para utilizar a função `sendMessage`, basta chamar essa função com um produtor Kafka previamente criado, o tópico para o qual deseja enviar a mensagem e a própria mensagem, e lidar com possíveis erros retornados.

## Consumindo mensagens com Golang

### Criando um consumidor Kafka em Golang

A criação de um consumidor Kafka em Golang segue um processo semelhante ao de criar um produtor. Utilizando a mesma biblioteca cliente Kafka, é preciso configurar e instanciar um objeto de consumidor.

```go
func createConsumer(broker, groupID, topic string) (*kafka.Consumer, error) {
	config := &kafka.ConfigMap{
		"bootstrap.servers": broker,
		"group.id":          groupID,
		"auto.offset.reset": "earliest",
	}
	consumer, err := kafka.NewConsumer(config)
	if err != nil {
		return nil, err
	}
	return consumer, nil
}
```
A função `createConsumer` tem como objetivo criar um consumidor Kafka utilizando a biblioteca confluent-kafka-go. Ela aceita três parâmetros: `broker`, que é a URL do servidor Kafka; `groupID`, que identifica o grupo de consumidores ao qual o consumidor pertence; e `topic`, que é o tópico Kafka do qual o consumidor receberá mensagens.

A função começa configurando um mapa de configuração `kafka.ConfigMap`. Esse mapa inclui as seguintes configurações:

1. `"bootstrap.servers"`: Define a URL do servidor Kafka que o consumidor se conectará.
2. `"group.id"`: Define o identificador do grupo de consumidores ao qual o consumidor pertencerá. Os consumidores que pertencem ao mesmo grupo trabalham juntos para processar mensagens de um tópico. Eles garantem que cada mensagem seja processada por apenas um dos consumidores do grupo.
3. `"auto.offset.reset"`: Define como o consumidor deve começar a ler as mensagens caso não haja um deslocamento comitido para um tópico específico. Neste caso, a configuração está definida como "earliest", o que significa que o consumidor começará a processar mensagens desde o início do tópico.

Após definir o mapa de configuração, a função tenta criar um novo consumidor Kafka com a função `kafka.NewConsumer(config)`. Se a criação for bem-sucedida, a função retorna o consumidor e `nil` para o erro. Caso contrário, ela retorna `nil` para o consumidor e o erro encontrado durante a criação.

Essa função é útil para encapsular a lógica de criação de um consumidor Kafka e pode ser facilmente reutilizada em diferentes partes do código. Para criar um consumidor, basta chamar a função `createConsumer` com os parâmetros apropriados e verificar se há erros antes de começar a consumir mensagens do tópico desejado.

### Recebendo mensagens de um tópico Kafka

Com o consumidor criado, é possível receber mensagens de um tópico Kafka. O consumidor utiliza o método `Subscribe()` para se inscrever em um tópico e o método `Poll()` para receber as mensagens.

```go
func receiveMessages(consumer *kafka.Consumer, topic string) error {
	err := consumer.Subscribe(topic, nil)
	if err != nil {
		return err
	}

	for {
		ev := consumer.Poll(100)
		if ev == nil {
			continue
		}

		switch msg := ev.(type) {
		case *kafka.Message:
			fmt.Printf("Mensagem recebida: %s\n", string(msg.Value))
		case kafka.Error:
			return msg
		}
	}
}
```
A função `receiveMessages` é responsável por receber mensagens de um tópico Kafka usando um consumidor Kafka. Ela aceita dois parâmetros: `consumer`, que é uma instância de `*kafka.Consumer` previamente criada; e `topic`, que é o tópico Kafka do qual o consumidor receberá mensagens.

A função começa tentando se inscrever no tópico especificado usando a função `consumer.Subscribe(topic, nil)`. Se ocorrer um erro durante a inscrição, a função retorna esse erro.

Depois de se inscrever no tópico, a função entra em um loop infinito. Dentro desse loop, ela utiliza a função `consumer.Poll(100)` para verificar se há novos eventos a serem processados pelo consumidor. O parâmetro `100` indica que a função `Poll` aguardará até 100 milissegundos por um novo evento antes de retornar `nil` e continuar o loop.

Quando um evento é retornado pela função `Poll`, a função `receiveMessages` verifica o tipo desse evento usando uma instrução `switch`:

1. Se o evento for uma mensagem do tipo `*kafka.Message`, a função imprime o valor da mensagem no console, usando `fmt.Printf("Mensagem recebida: %s\n", string(msg.Value))`.
2. Se o evento for um erro do tipo `kafka.Error`, a função retorna esse erro, encerrando o loop e a função.

Essa função é útil para processar continuamente mensagens recebidas de um tópico Kafka, tratando-as conforme necessário (neste caso, imprimindo-as no console). Para utilizar a função `receiveMessages`, basta chamar essa função com um consumidor Kafka previamente criado e o tópico do qual deseja receber mensagens, e lidar com possíveis erros retornados.

## Tópicos avançados na integração entre Golang e Kafka

### Trabalhando com Kafka Streams

Kafka Streams é uma biblioteca para construção de aplicações e microsserviços de streaming de eventos. Com a integração entre Golang e Kafka Streams, é possível processar, transformar e agregar eventos em tempo real.
```go
package main

import (
	"fmt"
	"strings"

	"github.com/lovoo/goka"
	"github.com/lovoo/goka/codec"
)

var (
	brokers = []string{"localhost:9092"}
	topic   = "input-topic"
	group   = goka.Group("example-group")
)

func processCallback(ctx goka.Context, msg interface{}) {
	input := msg.(string)
	output := strings.ToUpper(input)
	fmt.Printf("Mensagem processada: %s\n", output)
}

func main() {
    context := context.background()
	// Define um novo processador Goka
	processor, err := goka.NewProcessor(brokers, goka.DefineGroup(group,
		goka.Input(goka.Stream(topic), new(codec.String), processCallback),
	))

	if err != nil {
		panic(err)
	}

	// Inicie o processador Goka
	if err = processor.Run(context); err != nil {
		panic(err)
	}
}

```
Neste exemplo, estamos usando a biblioteca Goka para criar um processador Kafka Stream. O processador consome mensagens do tópico "input-topic" e converte o valor de cada mensagem para maiúsculas. O resultado é impresso no console.

A função `processCallback` é a função de processamento que é chamada para cada mensagem no tópico. Ele recebe um `goka.Context` e a mensagem (que é uma string neste caso). A função converte a mensagem para maiúsculas e imprime o resultado no console.

Na função `main`, criamos um novo processador Goka usando `goka.NewProcessor` com o grupo e os tópicos de entrada definidos. Em seguida, iniciamos o processador com `processor.Run(context)`.

### Lidando com erros e exceções

Ao trabalhar com Golang e Kafka, é essencial lidar com erros e exceções. Utilize a declaração `error` do Golang para tratar erros e garantir a resiliência e a estabilidade do sistema.

Aqui estão dois exemplos de tratamento de erro ao trabalhar com o produtor Kafka em Golang.

1. **Tratamento de erro ao criar um produtor Kafka:**

```go
producer, err := kafka.NewProducer(config)
if err != nil {
    log.Fatalf("Erro ao criar produtor: %v", err)
}
```

Este exemplo mostra como tratar erros ao criar um novo produtor Kafka. Ao chamar a função `kafka.NewProducer(config)`, ela retorna um produtor e um erro. Se o erro não for `nil`, isso significa que ocorreu um problema ao criar o produtor, e o programa registra a mensagem de erro usando `log.Fatalf()`.

2. **Tratamento de erro ao enviar uma mensagem para um tópico Kafka:**

```go
deliveryChan := make(chan kafka.Event)
producer.Produce(&kafka.Message{TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny}, Value: []byte(value)}, deliveryChan)

e := <-deliveryChan
msg := e.(*kafka.Message)
if msg.TopicPartition.Error != nil {
    log.Printf("Erro ao enviar mensagem: %v", msg.TopicPartition.Error)
}
```

Este exemplo ilustra como tratar erros ao enviar mensagens para um tópico Kafka. Primeiro, um canal chamado `deliveryChan` é criado para receber eventos de entrega do produtor. Em seguida, a função `producer.Produce()` é chamada para enviar uma mensagem ao tópico Kafka. 

A função `Produce()` recebe dois argumentos: a mensagem a ser enviada e o canal de entrega. A mensagem é criada como uma instância de `kafka.Message` com a partição definida como `kafka.PartitionAny` (permitindo que o produtor escolha a partição) e o valor da mensagem convertido em uma fatia de bytes.

Após enviar a mensagem, o exemplo aguarda um evento de entrega no canal `deliveryChan`. Quando o evento é recebido, ele é convertido de volta para uma mensagem Kafka usando a conversão de tipo `e.(*kafka.Message)`. Se `msg.TopicPartition.Error` não for `nil`, isso indica que ocorreu um erro ao enviar a mensagem, e o programa registra a mensagem de erro usando `log.Printf()`.

## Conclusão

Em conclusão, a combinação de Golang e Kafka oferece um excelente desempenho, escalabilidade e confiabilidade para criar sistemas de streaming de dados distribuídos de alta capacidade. Ambos são amplamente utilizados em empresas de tecnologia e organizações de vários tamanhos.

Um exemplo notável de um sistema de larga escala que utiliza Kafka e Golang é o Uber. A plataforma de transporte utiliza Kafka para processar e analisar grandes volumes de dados em tempo real, como localizações de veículos e solicitações de viagens. Para lidar com essa grande quantidade de dados, o Uber emprega Golang em vários de seus microserviços devido ao seu alto desempenho e eficiência na utilização de recursos.

Outra empresa que usa Kafka e Golang é a Cloudflare, uma empresa de segurança e desempenho na web. A Cloudflare utiliza Kafka para gerenciar e analisar bilhões de eventos de log por dia, ajudando a identificar e bloquear ameaças de segurança. Golang desempenha um papel crucial na criação de processadores de streaming de dados eficientes e escaláveis que podem lidar com essa carga de trabalho massiva.

Esses exemplos demonstram que a integração de Golang e Kafka é uma escolha comprovada e eficaz para construir sistemas distribuídos de larga escala. A combinação dessas duas tecnologias oferece uma base sólida para o desenvolvimento de soluções escaláveis e confiáveis que podem enfrentar os desafios do processamento de dados em tempo real em ambientes altamente exigentes.