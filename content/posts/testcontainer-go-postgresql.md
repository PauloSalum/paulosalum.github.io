---
title: "Teste de Integração com TestContainers e Golang: Testando Conexão com Banco de Dados PostgreSQL"
date: 2023-05-02T15:47:17-03:00
draft: false
---
Neste artigo, demonstraremos como usar a biblioteca TestContainers em um projeto Golang para executar testes de integração com um banco de dados PostgreSQL. Vamos criar um teste que verifica a conexão com o banco de dados usando um contêiner Docker.

## Importando bibliotecas necessárias

Primeiramente, importamos as bibliotecas necessárias para criar e gerenciar contêineres Docker e realizar testes usando a biblioteca `testify`:

```go
import (
	"context"
	"database/sql"
	"fmt"
	_ "github.com/lib/pq"
	"github.com/stretchr/testify/assert"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
	"testing"
	"time"
)
```

## Função para criar e configurar contêiner Docker

Em seguida, criamos a função `criarConfigurarContainer` que recebe a imagem Docker, as portas expostas e as variáveis de ambiente como parâmetros e retorna um contêiner configurado e iniciado:

```go
func criarConfigurarContainer(imagemDocker string, portasExpostas []string, variaveisAmbiente map[string]string) (testcontainers.Container, error) {
	// Configuração do contêiner
	containerRequest := testcontainers.ContainerRequest{
		Image:        imagemDocker,
		ExposedPorts: portasExpostas,
		Env:          variaveisAmbiente,
		WaitingFor:   wait.ForLog("database system is ready to accept connections").WithStartupTimeout(2 * time.Second),
	}

	// Criação do contêiner
	conteiner, err := testcontainers.GenericContainer(context.Background(), testcontainers.GenericContainerRequest{
		ContainerRequest: containerRequest,
		Started:          true,
	})

	if err != nil {
		return nil, fmt.Errorf("erro ao criar o contêiner: %w", err)
	}

	return conteiner, nil
}
```
A função `criarConfigurarContainer` é responsável por criar e configurar um contêiner Docker utilizando a biblioteca Testcontainers-Go. Ela aceita três parâmetros:

1. `imagemDocker`: a string contendo o nome da imagem Docker que será utilizada para criar o contêiner.
2. `portasExpostas`: uma lista de strings representando as portas que serão expostas pelo contêiner.
3. `variaveisAmbiente`: um mapa (map) onde a chave é a string representando o nome da variável de ambiente e o valor é a string representando o valor da variável de ambiente.

A função inicia definindo a estrutura `testcontainers.ContainerRequest` com as informações fornecidas nos parâmetros. Essa estrutura é usada para configurar o contêiner Docker, incluindo a imagem Docker, as portas expostas e as variáveis de ambiente. A propriedade `WaitingFor` é configurada com um log específico e um tempo limite de inicialização (2 segundos) para garantir que o contêiner esteja pronto para aceitar conexões antes de prosseguir.

Em seguida, a função `testcontainers.GenericContainer` é chamada com um contexto e um objeto `testcontainers.GenericContainerRequest`. A função cria o contêiner Docker com base na configuração fornecida (`containerRequest`) e inicia-o (propriedade `Started` definida como `true`).

Se ocorrer algum erro durante a criação do contêiner, a função retorna `nil` e um erro formatado com informações sobre o problema encontrado. Caso contrário, a função retorna a instância do contêiner criado e `nil` para o erro.

Essa função é útil para simplificar a criação e configuração de contêineres Docker em testes de integração, permitindo que você se concentre no teste em si em vez de lidar com a configuração do Docker.

## Teste de integração com o PostgreSQL

Agora, criamos a função de teste `TestRepository_Ping`, que utiliza a função `criarConfigurarContainer` para iniciar um contêiner PostgreSQL e testar a conexão com o banco de dados:

```go
func TestRepository_Ping(t *testing.T) {
	portasExpostas := []string{"5432/tcp"}
	variaveisAmbiente := map[string]string{"POSTGRES_PASSWORD": "postgres", "POSTGRES_DB": "postgres", "POSTGRES_USER": "postgres"}
	container, err := criarConfigurarContainer("docker.io/postgres:15.2-alpine", portasExpostas, variaveisAmbiente)
	if err != nil {
		t.Fatalf("erro ao criar o contêiner: %s", err)
	}
	defer func(container testcontainers.Container, ctx context.Context) {
		err := container.Terminate(ctx)
		if err != nil {
			t.Fatalf("erro ao finalizar o contêiner: %s", err)
		}
	}(container, context.Background())
```

Após criar e configurar o contêiner PostgreSQL, obtemos a porta mapeada no host e estabelecemos a conexão com o banco de dados:

```go
	portaMapeada, err := container.MappedPort(context.Background(), "5432/tcp")
	if err != nil {
		t.Fatalf("erro ao obter a porta mapeada: %s", err)
	}

	db, err := sql.Open("postgres", fmt.Sprintf("host=%s port=%s user=%s password=%s dbname=%s sslmode=disable", "localhost", portaMapeada.Port(), "postgres", "postgres", "postgres")) 
    if err != nil { 
        t.Fatalf("erro ao conectar no banco de dados: %s", err) 
    }
```

Com a conexão estabelecida, criamos uma instância do repositório e executamos o método `Ping()` para verificar a conexão com o banco de dados:

```go
	repository := NewRepository(db)
	time.Sleep(1 * time.Second)
	err2 := repository.Ping()
	defer db.Close()
	assert.NoError(t, err2)
}
```

Esse trexo do teste realiza as seguintes ações:

1. `repository := NewRepository(db)`: Uma nova instância de um repositório é criada usando a função `NewRepository`, que recebe como parâmetro a conexão de banco de dados `db` previamente estabelecida.

2. `time.Sleep(1 * time.Second)`: Uma pausa de 1 segundo é introduzida antes de executar o teste. Essa pausa pode ser necessária para garantir que o contêiner Docker e o banco de dados estejam prontos para aceitar conexões e realizar operações antes de prosseguir com o teste.

3. `err2 := repository.Ping()`: A função `Ping` é chamada no repositório para verificar se a conexão com o banco de dados está funcionando corretamente. Se houver algum problema na conexão, a função retornará um erro (`err2`).

4. `defer db.Close()`: A função `Close` é chamada com a palavra-chave `defer` para garantir que a conexão com o banco de dados seja fechada após a conclusão do teste, independentemente do resultado. A palavra-chave `defer` em Go permite programar a execução de uma função para o momento em que a função atual retorna. Isso é útil para garantir que os recursos sejam liberados corretamente.

5. `assert.NoError(t, err2)`: A função `NoError` do pacote `assert` é usada para verificar se a função `Ping` não retornou nenhum erro. Se a função `Ping` retornou um erro, o teste falhará com uma mensagem informando o erro retornado.


## Conclusão

Neste artigo, vimos como usar a biblioteca TestContainers em um projeto Golang para executar testes de integração com um banco de dados PostgreSQL em um contêiner Docker. O exemplo demonstrou como criar e configurar um contêiner, estabelecer a conexão com o banco de dados e verificar a conexão utilizando um teste Golang.

A utilização de TestContainers é uma solução prática e eficiente para executar testes de integração, pois permite a criação e configuração de ambientes isolados e controlados, facilitando o desenvolvimento e a verificação da integração entre componentes de software.

O código utilizado nesse artigo encontra-se [aqui](https://github.com/PauloSalum/Examples-Go/tree/main/testcontainers/postgresql)
