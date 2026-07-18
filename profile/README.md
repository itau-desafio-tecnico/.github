# desafio-itau

Documentação central do projeto **desafio-itau**: escopo da solução, decisões de arquitetura (ADRs) e diagramas (C4, infraestrutura e fluxo) que amarram os repositórios de código e infraestrutura do sistema.

## Sumário

- [Escopo do projeto](#escopo-do-projeto)
- [Repositórios](#repositórios)
- [Documentação complementar (ADRs e draft)](#documentação-complementar-adrs-e-draft)
- [Diagrama C4 — Nível 2 (Contêineres)](#diagrama-c4--nível-2-contêineres)
- [Diagrama C4 — Nível 3 (Componentes)](#diagrama-c4--nível-3-componentes)
- [Diagrama de Infraestrutura](#diagrama-de-infraestrutura)
- [Diagrama de Fluxo](#diagrama-de-fluxo)

## Escopo do projeto

O desafio-itau é um sistema de **registro de ordens de serviço** ("ordens") vinculadas a um **solicitante** cadastrado previamente. É composto por dois microsserviços poliglotas que se comunicam de forma síncrona (validação) e assíncrona (evento de domínio), além da infraestrutura AWS que os sustenta:

- **[`requester-service`](../requester-service/README.md)** (Java/Spring Boot): mantém o cadastro de solicitantes (documento, nome, e-mail, status ativo/inativo) e expõe um endpoint de validação consumido pelo `order-service`.
- **[`order-service`](../order-service/README.md)** (Python/FastAPI): cria ordens de forma idempotente, validando o solicitante contra o `requester-service`, e publica de forma confiável o evento `OrderCreated` via outbox pattern + SNS/SQS.
- **[`infrastructure`](../infrastructure/README.md)** (Terraform): provisiona todo o ambiente AWS — rede, ECS Fargate, RDS, SNS/SQS, ALB e a stack de observabilidade (OpenTelemetry, Prometheus, Jaeger, Grafana).
- **[`terraform-backend`](../terraform-backend/README.md)** (Terraform): bootstrap do backend remoto de estado (bucket S3 + tabela DynamoDB de lock) usado pelo `infrastructure`.

O rascunho original da solução (ver [draft](#documentação-complementar-adrs-e-draft)) previa também um frontend em Angular para cadastro de solicitantes e criação de ordens; esse frontend **não faz parte dos repositórios entregues** — o escopo implementado cobre as duas APIs backend, a infraestrutura e a observabilidade descritas acima.

Garantias de negócio centrais do sistema (detalhadas nos ADRs):

- **Idempotência** na criação de ordens via header `Idempotency-Key` ([ADR 0003](docs/0003-idempotency-key-header.md)).
- **Consistência entre persistência e mensageria** via transactional outbox ([ADR 0004](docs/0004-outbox-pattern.md)).
- **Entrega confiável** do evento `OrderCreated` via SNS + SQS com DLQ ([ADR 0002](docs/0002-messaging-sns-sqs.md)).
- **Observabilidade vendor-neutral** via OpenTelemetry, com métricas no Prometheus e tracing no Jaeger, visualizados no Grafana ([ADR 0005](docs/0005-observability-opentelemetry.md)).

## Repositórios

| Repositório | Linguagem/Stack | Responsabilidade |
|---|---|---|
| [`order-service`](../order-service/README.md) | Python 3.12 / FastAPI | Criação idempotente de ordens, validação do solicitante, outbox + publicação SNS |
| [`requester-service`](../requester-service/README.md) | Java 21 / Spring Boot 4 | Cadastro e validação de solicitantes |
| [`infrastructure`](../infrastructure/README.md) | Terraform | Provisionamento da infraestrutura AWS de execução (rede, ECS, RDS, mensageria, observabilidade) |
| [`terraform-backend`](../terraform-backend/README.md) | Terraform | Bootstrap do backend remoto de estado do Terraform |
| `documentation` (este repositório) | Markdown | ADRs, draft da solução e diagramas |

## Documentação complementar (ADRs e draft)

As decisões arquiteturais estão registradas como Architecture Decision Records em [`docs/`](docs):

| ADR | Decisão |
|---|---|
| [0001](docs/0001-compute-ecs-fargate.md) | Compute: ECS Fargate (em vez de EKS ou Lambda) |
| [0002](docs/0002-messaging-sns-sqs.md) | Mensageria: SNS + SQS (em vez de MSK/Kafka ou EventBridge) |
| [0003](docs/0003-idempotency-key-header.md) | Idempotência via header `Idempotency-Key` |
| [0004](docs/0004-outbox-pattern.md) | Outbox pattern para publicação confiável de eventos |
| [0005](docs/0005-observability-opentelemetry.md) | Observabilidade: OpenTelemetry + Grafana/Prometheus/Jaeger |
| [0006](docs/0006-liquibase-migrations.md) | Migração de schema com Liquibase nos dois serviços |
| [0007](docs/0007-cicd-static-aws-credentials.md) | CI/CD: GitHub Actions com credenciais AWS estáticas |

O rascunho inicial da solução está em [`docs/draft.png`](docs/draft.png) — um esboço de alto nível do fluxo cliente → frontend → `requester-service`/`order-service` → SNS/SQS, usado como ponto de partida antes do detalhamento em ADRs e nos diagramas C4 abaixo.

## Diagrama C4 — Nível 2 (Contêineres)

```mermaid
C4Container
    title Contêineres do sistema desafio-itau

    Person(cliente, "Cliente da API", "Consumidor dos serviços (aplicação cliente / integração)")

    System_Boundary(sistema, "desafio-itau") {
        Container(alb, "Application Load Balancer", "AWS ALB", "Roteamento HTTP por path para os serviços")
        Container(orderSvc, "order-service", "Python 3.12 / FastAPI", "Cria ordens de forma idempotente, valida solicitante e publica OrderCreated via outbox")
        Container(requesterSvc, "requester-service", "Java 21 / Spring Boot", "Cadastra e valida solicitantes")
        ContainerDb(orderDb, "Order DB", "PostgreSQL 16 (RDS)", "Tabelas orders e outbox_events")
        ContainerDb(requesterDb, "Requester DB", "PostgreSQL 16 (RDS)", "Tabela requesters")
        Container(sns, "Tópico order-created", "AWS SNS", "Publica o evento OrderCreated")
        Container(sqs, "Fila order-processing", "AWS SQS + DLQ", "Fila de consumo do evento, com dead-letter queue")
        Container(otel, "OTel Collector", "OpenTelemetry Collector", "Recebe traces/métricas via OTLP")
        Container(prometheus, "Prometheus", "Prometheus", "Armazena métricas")
        Container(jaeger, "Jaeger", "Jaeger", "Armazena e exibe traces distribuídos")
        Container(grafana, "Grafana", "Grafana", "Dashboards sobre Prometheus e Jaeger")
    }

    Rel(cliente, alb, "HTTPS")
    Rel(alb, orderSvc, "/py-order-service/*", "HTTP")
    Rel(alb, requesterSvc, "/jv-requester-service/*", "HTTP")

    Rel(orderSvc, requesterSvc, "Valida solicitante", "GET /requesters/{id}/validation")
    Rel(orderSvc, orderDb, "Lê/grava order + outbox_events (mesma transação)", "SQL/JDBC")
    Rel(requesterSvc, requesterDb, "Lê/grava requesters", "SQL/JDBC")
    Rel(orderSvc, sns, "Publica OrderCreated (via outbox dispatcher)", "AWS SDK")
    Rel(sns, sqs, "Encaminha evento", "SNS subscription")

    Rel(orderSvc, otel, "Exporta traces/métricas", "OTLP/HTTP")
    Rel(requesterSvc, otel, "Exporta traces/métricas", "OTLP/HTTP")
    Rel(otel, prometheus, "Métricas", "remote write / scrape")
    Rel(otel, jaeger, "Traces", "OTLP/gRPC")
    Rel(grafana, prometheus, "Consulta")
    Rel(grafana, jaeger, "Consulta")
```

## Diagrama C4 — Nível 3 (Componentes)

Ambos os serviços seguem arquitetura hexagonal (ports & adapters): domínio isolado de framework, casos de uso dependendo apenas de ports, e adapters de entrada (HTTP) e saída (persistência, HTTP client, mensageria) plugados nas bordas.

### order-service

```mermaid
C4Component
    title Componentes do order-service

    Container_Boundary(orderSvc, "order-service") {
        Component(router, "orders router", "FastAPI Router", "POST /orders — recebe Idempotency-Key e corpo da requisição")
        Component(useCase, "CreateOrderUseCase", "app/", "Orquestra idempotência, validação do solicitante e persistência com outbox")
        Component(domain, "Order / OutboxEvent", "domain/entities.py", "Regras de negócio: geração de order_number, status")
        Component(ports, "Ports", "domain/ports.py", "OrderRepository, OutboxRepository, RequesterClient, EventPublisher (interfaces)")
        Component(orderRepo, "SqlAlchemyOrderRepository", "infra/db/", "Implementa OrderRepository; salva order + outbox na mesma transação")
        Component(requesterClient, "HttpRequesterClient", "infra/http/", "Implementa RequesterClient via httpx + retry (tenacity)")
        Component(dispatcher, "OutboxDispatcher", "infra/message/", "Poll assíncrono de outbox_events PENDING")
        Component(snsPublisher, "SnsEventPublisher", "infra/message/", "Implementa EventPublisher via boto3 SNS")
    }

    ContainerDb(db, "Order DB", "PostgreSQL")
    Container(requesterSvc, "requester-service", "Java/Spring Boot")
    Container(sns, "SNS Topic", "AWS SNS")

    Rel(router, useCase, "chama")
    Rel(useCase, domain, "usa")
    Rel(useCase, ports, "depende de")
    Rel(orderRepo, ports, "implementa")
    Rel(requesterClient, ports, "implementa")
    Rel(snsPublisher, ports, "implementa")
    Rel(orderRepo, db, "SQL")
    Rel(requesterClient, requesterSvc, "GET /requesters/{id}/validation")
    Rel(dispatcher, orderRepo, "busca eventos PENDING")
    Rel(dispatcher, snsPublisher, "publica")
    Rel(snsPublisher, sns, "AWS SDK")
```

### requester-service

```mermaid
C4Component
    title Componentes do requester-service

    Container_Boundary(requesterSvc, "requester-service") {
        Component(controller, "RequesterController", "interfaces/rest/", "POST/GET /requesters, GET /requesters/{id}/validation")
        Component(createUC, "CreateRequesterUseCase", "app/usecase/", "Valida unicidade de documento e cria o solicitante")
        Component(getUC, "GetRequesterUseCase", "app/usecase/", "Busca solicitante por id")
        Component(validateUC, "ValidateRequesterUseCase", "app/usecase/", "Retorna se o solicitante existe e está ativo")
        Component(domain, "Requester", "domain/model/", "Record com invariantes de negócio (documento, e-mail)")
        Component(port, "RequesterRepository", "domain/repository/", "Port de persistência")
        Component(repoImpl, "RequesterRepositoryImpl", "infra/persistence/", "Implementa o port via Spring Data JPA")
    }

    ContainerDb(db, "Requester DB", "PostgreSQL")
    Container(orderSvc, "order-service", "Python/FastAPI")

    Rel(controller, createUC, "chama")
    Rel(controller, getUC, "chama")
    Rel(controller, validateUC, "chama")
    Rel(createUC, domain, "usa")
    Rel(createUC, port, "depende de")
    Rel(getUC, port, "depende de")
    Rel(validateUC, port, "depende de")
    Rel(repoImpl, port, "implementa")
    Rel(repoImpl, db, "JPA/SQL")
    Rel(orderSvc, controller, "GET /requesters/{id}/validation")
```

## Diagrama de Infraestrutura

Infraestrutura provisionada pelo repositório [`infrastructure`](../infrastructure/README.md) (ver módulos Terraform: `network`, `security`, `database`, `messaging`, `ecs`, `observability`).

![Desenho de Arquitetura](../docs/DI.png)

## Diagrama de Fluxo

Fluxo de criação de uma ordem, incluindo validação síncrona do solicitante e publicação assíncrona confiável via outbox pattern ([ADR 0003](docs/0003-idempotency-key-header.md) e [ADR 0004](docs/0004-outbox-pattern.md)).

```mermaid
sequenceDiagram
    actor Cliente
    participant Order as order-service
    participant Requester as requester-service
    participant DB as Order DB (orders + outbox_events)
    participant Dispatcher as OutboxDispatcher
    participant SNS as SNS (order-created)
    participant SQS as SQS (order-processing)

    Cliente->>Order: POST /orders\nHeader Idempotency-Key\n{requester_id, description}
    Order->>DB: busca order por idempotency_key

    alt chave já existe
        DB-->>Order: order existente
        Order-->>Cliente: 201 Created (mesma order, sem novo evento)
    else nova requisição
        Order->>Requester: GET /requesters/{id}/validation
        alt solicitante inválido/inexistente
            Requester-->>Order: validation=false / 404
            Order-->>Cliente: 422 Unprocessable Entity
        else requester-service indisponível
            Requester-->>Order: 5xx (após retries)
            Order-->>Cliente: 503 Service Unavailable
        else solicitante válido
            Requester-->>Order: validation=true
            Order->>Order: gera order_number, cria Order + OutboxEvent
            Order->>DB: grava order + outbox_events (mesma transação)
            DB-->>Order: commit
            Order-->>Cliente: 201 Created (order criada)
        end
    end

    loop polling periódico (outbox_poll_interval_seconds)
        Dispatcher->>DB: busca outbox_events PENDING
        DB-->>Dispatcher: eventos pendentes
        Dispatcher->>SNS: publish(OrderCreated)
        alt publicação com sucesso
            SNS-->>Dispatcher: ack
            Dispatcher->>DB: marca evento PUBLISHED
            SNS->>SQS: encaminha evento (subscription)
        else falha na publicação
            Dispatcher->>DB: incrementa attempts (FAILED se esgotar tentativas)
        end
    end
```
