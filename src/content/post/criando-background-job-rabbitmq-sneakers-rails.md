---
title: "Criando um background job com RabbitMQ e Sneakers no Rails"
description: "Sneakers é um serviço de execução de tarefas em backgroud que utiliza o RabbitMQ para realizar a execução de tarefas."
publishDate: "01 October 2022"
tags: ["rabbitmq", "sneakers", "ruby on rails", "gem", "backgroud job"]
coverImage:
  src: "./covers/criando-background-job-rabbitmq-sneakers-rails.png"
  alt: "Astro build wallpaper"
---

## Introdução
Um dos serviços mais utilizados no Rails de background jobs é o Sidekiq, recentemente descobrir uma excelente alternativa chamada Sneakers. Sneakers é um serviço de execução de tarefas em backgroud que utiliza o RabbitMQ
para realizar a execução de tarefas, nesse post veremos como implementar e quais as vantagens de se
utilizá-la.

## Características do Sneakers

1. Tem um processamento de jobs em backgroud  de alta performance e disponibilidade;
2. Usa o modelo de execução híbrida onde muitos processos são gerados (como o Unicorn) e muitas threads são
   usadas por processo (como Puma), para que todos os seus núcleos atinjam
   o máximo e tenha o melhor dos dois mundos;
3. Executa mais de 1000 req/s;
4. Possui uma estrutura de processamento altamente disponível
   (tendo as mesmas garantias que o RabbitMQ oferece);
5. Possui uma DSL/API familiar que também suporta semânticas avançadas de mensagens, como rejeitar,
   reenfileirar, reconhecer, etc.

## Requisitos Necessários

Para configuração e execução do projeto será necessário atender aos seguintes requisitos:

1. Ter o docker instalado;
2. Ter Rails na versão ≥ 6;
3. Ter o ruby na versão ≥ 3
4. Sneakers na versão ≥ 2.11.0

## Instalação do RabbitMQ

O RabbitMQ será utilizado para armazenar as informações (mensagens) que serão processados pelos works da nossa aplicação, para isso vamos usar o docker para baixar uma imagem já pronta para utilizarmos no projeto.

Crie uma pasta chamada `rabbitmq-work` e dentro desta pasta criei o  arquivo `docker-compose.yml` com o
seguinte conteúdo:

```yaml
version: '3.7'

services:
  rabbitmq:
    container_name: rabbitmq-worker
    image: rabbitmq:management-alpine
    ports:
      - 5672:5672
      - 15672:15672
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq
      RABBITMQ_DEFAULT_PASS: rabbitmq
volumes:
  rabbitmq-data:
```
depois rode o comando:

```bash
docker-compose up -d
```
o docker irá baixar a imagem do rabbitmq e subir o container, para acessar o sistema de administração do
rabbitmq é necessário habilitar o gerenciamento de plugins com o comando:

```bash
rabbitmq-plugins enable rabbitmq_management
```

o comando acima deve ser executado no terminal dentro do container do rabbitmq  uma das maneiras de fazer
isso é usar o plugin de gerenciamento do docker do VS Code instale em seu Visual Code utilizando o menu de
extensions
<br /><br />

![Extensão para docker](@/assets/docker.webp)
[ampliar imagem](https://postimg.cc/62KR8KC4)
<br /><br />

clique no menu do docker então clique com o botão direito no container do rabbitmq  e clique em `Attach Shell`
<br /><br />

![Extensão para docker](@/assets/rabbit-shell.webp)
[ampliar imagem](https://postimg.cc/HJXXWn8C)
<br /><br />

será aberto o terminal dentro do container
<br /><br />

![Extensão para docker](@/assets/rabbit-shell1.webp)
[ampliar imagem](https://postimg.cc/tZz6PczS)
<br /><br />

:::tip
Também é possível realizar o procedimento mostrado acima com a execução do comando:
`docker-compose run rabbitmq bash` no seu terminal
:::

então execute o comando:

```bash
rabbitmq-plugins enable rabbitmq_management
```
<br /><br />

veja como fica na imagem abaixo:

![Extensão para docker](@/assets/rabbit-shell2.webp)
[ampliar imagem](https://postimg.cc/0r8Dx88L)
<br /><br />

execute:

```bash
docker-compose up -d
```

Pronto com o rabbitmq_management habilitado é possível acessar
`http://localhost:15672` usando o login e a senha `rabbitmq`



## Implementação da aplicação BackgroundJob

Dependências:

`Ruby 3.3.0`

`Rails 6.1.7.9`

Entre na pasta rabbitmq-work e rode:

```bash
rails new background-job
```

após a aplicação ser criada com sucesso adicione o `sneakers` ao gemfile do rails

```bash
bundle add sneakers
```
depois dentro da pasta da aplicação crie o arquivo `/config/initializers/sneakers.rb` com o seguinte conteúdo:

```ruby
# frozen_string_literal: true

require 'sneakers/metrics/logging_metrics'

Sneakers.configure(
  amqp:      'amqp://rabbitmq:rabbitmq@localhost:5672', # Connection with RabbitMQ
  metrics:   Sneakers::Metrics::LoggingMetrics.new,     # A metrics provider implementation
  workers:   4,                                         # Number of per-cpu processes to run
  threads:   5,                                         # Threadpool size (good to match prefetch)
  prefetch:  5,                                         # Grab 5 jobs together. Better speed.
  durable:   true,                                      # Is queue durable?
  ack:       true,                                      # Must we acknowledge?
  heartbeat: 2,                                         # Keep a good connection with broker
  env:       'development')                             # Environment

Sneakers.logger = Rails.logger
Sneakers.logger.level = Logger::INFO
```
Aqui está uma explicação detalhada sobre essas configurações:

```ruby
require 'sneakers/metrics/logging_metrics'
```
Esta linha importa a classe LoggingMetrics do módulo Sneakers::Metrics, que será usada para registrar
métricas.

```ruby
Sneakers.configure(
  amqp:      'amqp://rabbitmq:rabbitmq@localhost:5672', # Connection with RabbitMQ
  metrics:   Sneakers::Metrics::LoggingMetrics.new,     # A metrics provider implementation
  workers:   4,                                         # Number of per-cpu processes to run
  threads:   5,                                         # Threadpool size (good to match prefetch)
  prefetch:  5,                                         # Grab 5 jobs together. Better speed.
  durable:   true,                                      # Is queue durable?
  ack:       true,                                      # Must we acknowledge?
  heartbeat: 2,                                         # Keep a good connection with broker
  env:       'development')
```

:::note
Essas configurações podem ser complementadas ou sobrescritas dentro de cada classe de work.
:::

Esta configuração define vários parâmetros para o funcionamento do Sneakers:

* amqp: URL de conexão com o RabbitMQ.
* metrics: Instância de LoggingMetrics para registrar métricas.
* workers: Número de processos por CPU.
* threads: Tamanho do pool de threads.
* prefetch: Número de jobs que serão pré-carregados.
* durable: Define se a fila é durável.
* ack: Define se é necessário reconhecimento (acknowledgment).
* heartbeat: Intervalo de heartbeat para manter a conexão com o broker.
* env: Ambiente de execução (neste caso, 'development').

```ruby
Sneakers.logger = Rails.logger
Sneakers.logger.level = Logger::INFO
```
Estas linhas configuram o logger do Sneakers para usar o logger do Rails e definem o nível de log para INFO.

Este código é essencial para integrar o processamento de filas com RabbitMQ na aplicação, garantindo que as tarefas sejam gerenciadas de forma eficiente e que as métricas e logs sejam devidamente registrados.

após isso crie a pasta `/app/workers` e crie o arquivo `/app/workers/profiling_worker.rb` com o código a
seguir:

```ruby
class ProfilingWorker
  include Sneakers::Worker

  from_queue 'downloads',
             exchange: 'download_process',
             timeout_job_after: 1

  def work(message)
    logger.info JSON.parse(message)
    ack!
  rescue StandardError => e
    logger.error e
  end
end
```
O código acima define uma classe ProfilingWorker que utiliza a biblioteca Sneakers para processar mensagens de uma fila RabbitMQ.
Este worker é configurado para processar mensagens da fila 'downloads', registrar o conteúdo da mensagem e reconhecer a mensagem como processada. Se ocorrer um erro durante o processamento, o erro será registrado.

Aqui está uma explicação detalhada do código:

```ruby
class ProfilingWorker
  include Sneakers::Worker
```
* A classe `ProfilingWorker` inclui o módulo `Sneakers::Worker`, o que a transforma em um worker que pode processar mensagens de uma fila RabbitMQ.

```ruby
from_queue 'downloads',
             exchange: 'download_process',
             timeout_job_after: 1
```
* O método `from_queue` configura o worker para consumir mensagens da fila chamada `downloads`.
* exchange: `download_process` especifica o exchange RabbitMQ associado à fila.
* timeout_job_after: 1 define um tempo limite de 1 segundo para o processamento de cada mensagem.

```ruby
 def work(message)
    logger.info JSON.parse(message)
    ack!
  rescue StandardError => e
    logger.error e
  end
```
* O método `work` é chamado para processar cada mensagem recebida da fila.
* `logger.info JSON.parse(message)` registra a mensagem recebida após convertê-la de JSON para um objeto Ruby.
* `ack!` reconhece a mensagem, informando ao RabbitMQ que foi processada com sucesso.
* O bloco `rescue` captura qualquer exceção (StandardError) que ocorra durante o processamento da mensagem.
`logger.error` e registra o erro ocorrido.

Depois adicione o código

```ruby
require "sneakers/tasks"
```

ao arquivo `/Rakefile`

```ruby
# Add your own tasks in files placed in lib/tasks ending in .rake,
# for example lib/tasks/capistrano.rake, and they will automatically be available to Rake.

require_relative "config/application"
require "sneakers/tasks"

Rails.application.load_tasks
```

:::note
Esse código no `Rakefile` adiciona uma task ao rails que irá executar os jobs implementados com o `Sneakers`
execute no terminal `rake -T` e procure por `rake sneakers:run # Start work (set $WORKERS=Klass1,Klass2)`
para confirmar se a configuração funciona.
:::

![Extensão para docker](@/assets/rabbit-shell3.webp)
[ampliar imagem](https://postimg.cc/fST9vv2w)
<br /><br />

rode o comando:

```ruby
WORKERS=ProfilingWorker rake sneakers:run
```

:::note
No caso acima apenas o work `ProfilingWorker` será executado caso queira executar mais outro work separe seu
nome por virgula dessa forma `WORKERS=ProfilingWorker,OtherWork rake sneakers:run` ou caso queira executar
todos os works execute `rake sneakers:run`
:::

abra o admin do `RabbitMQ` e acesse a área onde fica as filas, procure a fila `downloads` e publique a
 mensagem abaixo:

```json
{
  "id": "035a0633-bc58-4f21-9a9f-d422b88dd0bd",
  "codigo": "654",
  "nome": "Banco A.J.Renner S.A.",
  "ordem": null,
  "ativo": false
}
```

:::note
Não é necessário criar a exchange e a fila no RabbitMQ pois quando o work é iniciado ele se encarrega de criar
essas configurações.
:::
<br />

Ao executar a mensagem acima a saída no console da aplicação deverá ser como é mostrado na imagem abaixo.
<br /><br />

![Extensão para docker](@/assets/terminal1.webp)
[ampliar imagem](https://postimg.cc/Cz58Bmn7)
<br /><br />

no log registra-se quando o worker inicia, quanto tempo levou para ser executado, informa que o ACK foi enviado para o RabbitMQ  e quando o work finaliza.
<br /><br />

![Extensão para docker](@/assets/log.webp)
[ampliar imagem](https://postimg.cc/Bj82yMqb)
<br /><br />

## Fluxo de Execução de Works

Abaixo é exibido de forma visual como um work funciona. Quando se executa o comando `rake sneakers:run` o work ProfillingWorker irá ser iniciado e ficará escutando a fila `downloads` ele irá executar qualquer mensagem
que seja enviado para essa fila. Do outro lado podemos ter uma ou várias aplicações que enviam mensagens para
a fila de `download`, o que é interessante de se vê no fluxo é que as aplicações que produzem dados para o
worker podem ser implementadas em diferentes linguagens.
<br /><br />

![Extensão para docker](@/assets/sneakers.webp)
[ampliar imagem](https://postimg.cc/0rcmyLjr)
<br /><br />

### Github

https://github.com/edivandecastro/rabbitmq-work/commit/a913fd2295417b86805da5df55197d6862e796de

### Referências

https://jondot.github.io/sneakers/
