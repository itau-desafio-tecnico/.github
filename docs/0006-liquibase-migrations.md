# ADR 0006 — Migração de schema com Liquibase

## Status
Aceito

## Contexto
Os dois serviços precisam versionar o schema do banco (tabelas `solicitantes`, `orders`, `outbox_events`) de forma reprodutível entre ambientes.

## Decisão
Usar Liquibase nos dois serviços. No `requester-service` (Java/Spring Boot), integração nativa via `liquibase-core`, executado automaticamente no startup da aplicação. No `order-service` (Python/FastAPI), sem integração nativa com o ecossistema Python, o Liquibase roda como etapa separada (container/CLI), executado antes de subir a aplicação — localmente via Docker Compose, e no pipeline de deploy como um passo prévio à atualização do serviço ECS.

## Alternativas consideradas
- **Alembic no order-service** (nativo do ecossistema Python/SQLAlchemy): reduziria a dependência de uma ferramenta externa ao Python, mas divergiria da ferramenta de migração entre os dois serviços, tornando a documentação/operação menos uniforme.
- **Flyway no requester-service**: alternativa equivalente ao Liquibase no mundo Java; optamos por Liquibase por ser a ferramenta pedida explicitamente.

## Consequências
- Um único padrão de changelog (XML/YAML) documentado para os dois serviços, mesmo com integrações diferentes.
- `order-service` precisa de uma etapa de infraestrutura (container Liquibase) fora do processo Python, o que adiciona uma dependência de execução a mais no pipeline de deploy.
