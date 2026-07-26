# Roadmap RabbitMQ — do zero ao avançado

Stack: Spring Boot 4.1 · Java 21 · Spring AMQP · JPA/H2 · RabbitMQ (compose local)

Regra do jogo: **cada fase só começa quando você consegue explicar a anterior sem olhar.**
Não pule a Fase 5 (DLQ) — é onde 90% dos bugs de produção moram.

---

## Fase 0 — Modelo mental (sem código)

Antes de qualquer linha, entenda o caminho de uma mensagem:

```
Producer → [Exchange] --binding(routing key)--> [Queue] → Consumer
```

Pontos que precisam estar claros:

- [ ] O produtor **nunca** publica direto numa fila. Ele publica num **exchange**.
- [ ] O **binding** é a regra que liga exchange → fila. A **routing key** é o "endereço" da mensagem.
- [ ] Exchange e queue são objetos **independentes**. Nomes iguais não criam relação nenhuma entre eles — só o binding cria.
- [ ] Sem binding, o exchange **descarta a mensagem em silêncio**. Sem erro, sem log. A UI só avisa com um discreto "Message published, but not routed".
- [ ] **Exceção:** o *default exchange* (nome `""`) tem binding automático para toda fila, com routing key = nome da fila. É por isso que tutoriais de "hello world" parecem funcionar sem binding — eles usam o default sem avisar. Assim que você cria seu próprio exchange, o binding vira responsabilidade sua.
- [ ] **Connection** (TCP) vs **Channel** (sessão lógica multiplexada). Você abre 1 connection e N channels.
- [ ] **Queue** é um buffer com nome. Mensagem consumida é mensagem apagada.
- [ ] **vhost** é o namespace de isolamento (o padrão é `/`).

**Exercício** (http://localhost:15672, `admin`/`admin`) — sem código nenhum:

1. **Exchanges** → *Add a new exchange*: nome `helloworld`, tipo `fanout`
2. **Queues** → *Add a new queue*: nome `helloworld`, tipo `quorum`
3. Publique no exchange **antes** de criar o binding. Repare no aviso **"Message published, but not routed"** e confirme que a fila continua vazia. Esse passo é obrigatório: é o comportamento que você precisa ter visto uma vez.
4. **Queues** → `helloworld` → *Bindings* → ligue ao exchange `helloworld` (routing key vazia — fanout ignora)
5. Publique de novo e veja a mensagem chegar
6. Leia com *Get messages* (use `Ack Mode: Nack message requeue true` para espiar sem consumir)

Inspecionar pela API em vez da UI (útil para diagnosticar "cadê minha mensagem?"):

```bash
curl -s -u admin:admin http://localhost:15672/api/bindings/%2F | python3 -m json.tool
```

---

## Fase 1 — Hello World

**Aprender:** default exchange, `RabbitTemplate`, `@RabbitListener`.

- [ ] Configurar `spring.rabbitmq.*` no `application.yml` (host, port, username, password)
- [ ] Declarar uma `Queue` como `@Bean`
- [ ] Publicar com `rabbitTemplate.convertAndSend("nome-da-fila", "olá")`
- [ ] Consumir com `@RabbitListener(queues = "nome-da-fila")`
- [ ] Expor um endpoint REST `POST /publish` para disparar mensagens

**Pegadinha a entender:** por que `convertAndSend("fila", msg)` funciona sem exchange nenhum? Porque existe o *default exchange* (`""`), que roteia pela routing key = nome da fila. É conveniente e é um vício — a partir da Fase 2 você declara tudo explicitamente.

**Validar:** derrube o consumidor, publique 10 mensagens, veja acumular na UI, suba o consumidor e veja drenar.

---

## Fase 2 — Exchanges e roteamento

**Aprender:** os 4 tipos de exchange e quando usar cada um.

- [ ] **Direct** — routing key exata. Ex.: `pedido.criado` → fila de pedidos
- [ ] **Fanout** — ignora routing key, entrega para **todas** as filas ligadas (pub/sub puro)
- [ ] **Topic** — padrões com wildcards: `*` (uma palavra), `#` (zero ou mais). Ex.: `pedido.*.brasil`, `log.#`
- [ ] **Headers** — roteia por headers em vez de routing key (raro, mas saiba que existe)
- [ ] Declarar tudo via `@Bean`: `Queue`, `TopicExchange`, `BindingBuilder.bind(q).to(ex).with("chave")`

**Exercício:** um `TopicExchange` `eventos` com três filas:
- `fila.pedidos` ← `pedido.#`
- `fila.pagamentos` ← `pagamento.#`
- `fila.auditoria` ← `#` (recebe tudo)

Publique `pedido.criado`, `pagamento.aprovado`, `pedido.cancelado` e confira quem recebeu o quê.

**Conceito-chave:** uma mensagem sem fila casada é **descartada silenciosamente**. Guarde isso — a Fase 6 resolve.

---

## Fase 3 — Serialização e contratos

**Aprender:** parar de trafegar `String` e passar a trafegar objetos.

- [ ] Criar DTOs (`records` do Java 21 caem bem aqui)
- [ ] Registrar um `MessageConverter` JSON como `@Bean`
  - ⚠️ Spring Boot 4 usa Jackson 3 → a classe é `JacksonJsonMessageConverter`. O nome antigo `Jackson2JsonMessageConverter` está depreciado. Confirme qual existe no seu classpath.
- [ ] Entender o header `__TypeId__` e o `DefaultJacksonJavaTypeMapper`
- [ ] Configurar o mapeamento de tipos para **não** acoplar produtor e consumidor pelo nome do pacote

**Exercício:** publique um `PedidoCriado(String id, BigDecimal valor, Instant criadoEm)`. Depois **renomeie o pacote do DTO só no consumidor** e veja quebrar. Conserte com type mapping explícito.

**Conceito-chave:** versionamento de contrato. O que acontece quando você adiciona um campo? E quando remove? (Adicionar é compatível; remover não é.)

---

## Fase 4 — Confiabilidade do consumidor

**Aprender:** o que acontece quando o consumidor falha. Aqui começa o RabbitMQ "de verdade".

- [ ] `AcknowledgeMode`: `AUTO` (padrão do Spring) vs `MANUAL` vs `NONE`
- [ ] Ack manual: injetar `Channel` + `deliveryTag`, chamar `basicAck` / `basicNack`
- [ ] **Prefetch** (`spring.rabbitmq.listener.simple.prefetch`) — quantas mensagens não-ackadas o consumidor segura. Prefetch alto = throughput; prefetch baixo = distribuição justa
- [ ] Concorrência: `concurrency` / `max-concurrency`
- [ ] `requeue=true` vs `requeue=false` — e por que `requeue=true` num erro determinístico gera **loop infinito de CPU**

**Exercício obrigatório:** lance uma exceção no listener com requeue habilitado. Observe a mesma mensagem sendo reprocessada em loop, gastando 100% de CPU. **Esse é o erro que você precisa ter cometido uma vez.** Depois corrija com `AmqpRejectAndDontRequeueException`.

---

## Fase 5 — DLQ, retry e backoff ⭐

A fase mais importante do roadmap.

- [ ] **Dead Letter Exchange**: argumentos `x-dead-letter-exchange` e `x-dead-letter-routing-key` na declaração da fila
- [ ] Entender os 3 gatilhos de dead-letter: rejeição com `requeue=false`, TTL expirado, fila cheia (`x-max-length`)
- [ ] Retry local do Spring (`spring.rabbitmq.listener.simple.retry.*`) — e por que ele **bloqueia a thread** do consumidor
- [ ] Retry com **backoff exponencial** via DLX + `x-message-ttl` (o padrão "wait queue"): fila principal → DLX → fila de espera com TTL → volta pra principal
- [ ] **Parking lot queue**: depois de N tentativas, a mensagem sai do ciclo e vai para inspeção manual
- [ ] Contar tentativas pelo header `x-death`

**Exercício:** pipeline `pedidos` → falha → `pedidos.retry.5s` → `pedidos.retry.30s` → `pedidos.parking-lot`. Adicione um endpoint REST para **reprocessar** o que está na parking lot.

**Conceito-chave:** distinguir erro **transitório** (banco fora do ar → vale retentar) de erro **permanente** (JSON malformado → retentar nunca vai funcionar). Tratamentos diferentes.

---

## Fase 6 — Garantias do produtor

**Aprender:** como saber se a mensagem realmente chegou.

- [ ] `spring.rabbitmq.publisher-confirm-type=correlated` + `ConfirmCallback`
- [ ] `publisher-returns=true` + `mandatory=true` + `ReturnsCallback` — captura a mensagem que **não casou com nenhuma fila** (o problema silencioso da Fase 2)
- [ ] Durabilidade: fila `durable` + mensagem `PERSISTENT`. **As duas coisas** — uma sem a outra não sobrevive a restart
- [ ] Transações AMQP (`channel.txSelect`) e por que praticamente ninguém usa (ordens de magnitude mais lento que confirms)

**Exercício:** publique para uma routing key inexistente e capture no `ReturnsCallback`. Depois: publique 1000 mensagens persistentes, `docker restart rabbitmq`, e confirme que sobreviveram.

---

## Fase 7 — Idempotência e Transactional Outbox

Aqui o JPA/H2 que já está no `pom.xml` entra em cena.

- [ ] Entender que RabbitMQ entrega **at-least-once**. Duplicatas vão acontecer — projete para isso
- [ ] "Exactly-once" não existe no transporte; existe **efeito exatamente uma vez**, obtido com idempotência no consumidor
- [ ] Deduplicação: tabela `processed_messages` com o message id como chave única
- [ ] **Transactional Outbox**: gravar entidade + evento na **mesma transação** do banco, e um publisher separado envia depois
- [ ] Por que `@Transactional` + `rabbitTemplate.send()` no mesmo método é uma **dual-write bug** (o commit do banco pode falhar depois do envio, ou vice-versa)

**Exercício:** `POST /pedidos` grava `Pedido` + `OutboxEvent` numa transação. Um `@Scheduled` varre a outbox e publica. Force um rollback depois do send ingênuo e prove a inconsistência — depois corrija com outbox.

---

## Fase 8 — Padrões avançados

- [ ] **Request-Reply (RPC)**: `replyTo` + `correlationId`, `rabbitTemplate.convertSendAndReceive()`, filas exclusivas de resposta
- [ ] **Priority queues**: `x-max-priority` (e as armadilhas: só funciona com mensagens acumuladas)
- [ ] **Delayed messages**: plugin `rabbitmq_delayed_message_exchange` (não vem na imagem — precisa instalar o `.ez`) vs a alternativa TTL+DLX
- [ ] **Quorum queues** (`x-queue-type: quorum`) vs classic — o padrão moderno para produção. `delivery-limit` resolve o poison message nativamente
- [ ] **Lazy queues** para filas gigantes (mensagens direto em disco)
- [ ] **Streams** (`rabbitmq_stream`, porta 5552): log append-only e replay, o "modo Kafka" do Rabbit
- [ ] **Consumer priority** e **single active consumer** (quando ordem importa)

---

## Fase 9 — Produção e operação

- [ ] Cluster, replicação e por que quorum queues > classic mirrored (depreciadas)
- [ ] **Flow control** e alarmes de memória/disco — o broker aplica backpressure e o produtor trava. Saiba diagnosticar
- [ ] Métricas: plugin Prometheus (`rabbitmq_prometheus`) + Actuator + Micrometer
- [ ] HTTP Management API para automação
- [ ] `vhosts`, usuários, permissões, políticas (**policies** > argumentos hardcoded no código)
- [ ] **Shovel** e **Federation** para ligar brokers/datacenters
- [ ] Connection pooling, heartbeats, recuperação automática de conexão

**Testes (faça isso desde a Fase 3, não deixe pro fim):**
- [ ] `@SpringBootTest` + **Testcontainers** com `RabbitMQContainer`
- [ ] `spring-rabbit-test`: `RabbitListenerTestHarness`, `@SpringRabbitTest`
- [ ] Teste de contrato entre produtor e consumidor

---

## Fase 10 — Consolidação

Um projeto que amarre tudo. Sugestão: **e-commerce de mentira**, 3 serviços no mesmo repo (ou módulos Maven):

```
pedidos ──pedido.criado──► [topic: ecommerce] ──► pagamentos
                                    │                  │
                                    │                  └──pagamento.aprovado──┐
                                    ▼                                         ▼
                              notificacoes ◄─────────────────────────────  estoque
```

Requisitos: outbox no produtor, idempotência nos consumidores, DLQ com backoff em todos, publisher confirms, quorum queues, métricas no Prometheus, testes com Testcontainers.

---

## Comparação final (quando você já tiver bagagem)

Não comece por aqui, mas termine por aqui: **RabbitMQ vs Kafka**. Roteamento inteligente + filas efêmeras vs log particionado + replay. São ferramentas para problemas diferentes, e entender a fronteira é o que separa quem sabe usar de quem sabe escolher.

---

## Referências

- Tutoriais oficiais (os 6 primeiros são canônicos): https://www.rabbitmq.com/tutorials
- Spring AMQP reference: https://docs.spring.io/spring-amqp/reference/
- *RabbitMQ in Depth* — Gavin Roy (para a Fase 9)
