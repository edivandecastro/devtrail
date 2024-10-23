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
3. Ter o ruby na versão ≥ 2
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

Depois adicione o código abaixo ao arquivo `/Rakefile`


```ruby
require "sneakers/tasks"
```

:::note
Esse código no `Rakefile` adiciona uma task ao rails que irá executar os jobs implementados com o `Sneakers`
:::
<br /><br />

![Extensão para docker](@/assets/rabbit-shell2.webp)
[ampliar imagem](https://postimg.cc/0r8Dx88L)
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

## Implementação da aplicação BackgroundJob

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

![Extensão para docker](@/assets/rabbit-shell3.webp)
[ampliar imagem](https://postimg.cc/fST9vv2w)
<br /><br />

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
