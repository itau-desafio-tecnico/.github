# ADR 0003 — Idempotência via header Idempotency-Key

## Status
Aceito (revisado — ver seção "Revisão: escopo por solicitante")

## Contexto
A mesma requisição de criação de ordem pode chegar repetida (retries de cliente/gateway). O `numero_ordem` é gerado pelo próprio `order-service` no momento da criação, então não pode servir como chave de deduplicação — ele não existe antes do processamento.

## Decisão
O cliente da API envia um header `Idempotency-Key` (UUID gerado pelo próprio cliente) em `POST /orders`. Se a chave já existe **para o mesmo solicitante**, a resposta armazenada da primeira execução é retornada (mesmo `numero_ordem`, mesmo status HTTP), sem reprocessar nem publicar um novo evento. Ver detalhe do escopo da chave na revisão abaixo.

## Alternativas consideradas
- **Chave de negócio derivada do corpo da requisição** (ex.: hash de `solicitante_id` + campos da ordem): funciona sem exigir header extra, mas exige uma definição rígida de quais campos compõem a igualdade "mesma ordem", o que é frágil se o payload mudar sutilmente entre retries (ex.: timestamp do cliente).
- **Sem controle de idempotência, confiando em deduplicação da fila**: SQS/SNS não removem duplicidade de chamadas HTTP; não resolveria o requisito.

## Consequências
- Contrato de API exige o header em toda chamada de criação — precisa estar documentado no README/OpenAPI.
- Requer tabela auxiliar (ou coluna) guardando a resposta original associada à chave, para retorno idempotente exato.
- Simples de testar: dado o mesmo `Idempotency-Key` **e o mesmo `requester_id`**, duas chamadas retornam o mesmo `numero_ordem` e não duplicam linha em `orders` nem evento em `outbox_events`.

## Revisão: escopo por solicitante (`idempotency_key` + `requester_id`)

**Contexto da revisão.** A constraint original era unicidade apenas sobre `idempotency_key` (`uk_orders_idempotency_key`). Isso foi identificado como um defeito durante a gravação do vídeo de demonstração: uma requisição com `Idempotency-Key = x` e `requester_id = y` seguida de outra requisição com a **mesma** `Idempotency-Key = x` mas `requester_id = z` diferente retornava os dados da ordem criada para `y`, entregando indevidamente a ordem de um solicitante para outro.

**Decisão revisada.** A unicidade passa a ser uma constraint composta sobre `(idempotency_key, requester_id)` (`uk_orders_idempotency_key_requester_id`), aplicada via changeset Liquibase que remove a constraint antiga e cria a nova. A busca por ordem existente (`OrderRepository.get_by_idempotency_key`) agora recebe e filtra por ambos os valores.

Isso muda o comportamento observável para:
- `chave x` + `solicitante 1` → cria nova ordem.
- `chave x` + `solicitante 2` → cria **outra** nova ordem (mesma chave, solicitante diferente — não é considerado o mesmo pedido).
- `chave y` + `solicitante 1` → cria nova ordem.
- `chave x` + `solicitante 2` (repetida) → retorna a ordem já criada para `solicitante 2` no passo 2 (idempotência correta, escopada ao solicitante).

**Alternativa considerada e descartada nesta revisão**: validar em código, no caso de colisão de `idempotency_key`, se o `requester_id` da requisição bate com o da ordem já existente e, se não bater, retornar erro (ex.: `409 Conflict`) em vez de permitir a criação de uma segunda ordem. Foi descartada porque trata como erro um cenário que não é, de fato, um conflito — `Idempotency-Key` é um token técnico gerado pelo cliente (não uma chave de negócio global), então nada garante nem exige que dois solicitantes distintos usem valores diferentes; a colisão do mesmo valor entre dois clientes/solicitantes é uma coincidência esperada e inofensiva, não uma tentativa de reuso indevido. Rejeitar a requisição nesse caso puniria um cliente legítimo por uma colisão de token que não causou nenhum dado inconsistente. Escopar a unicidade por `(idempotency_key, requester_id)` resolve o problema na fonte, sem introduzir um caso de erro artificial.

**Consequências da revisão**:
- Constraint de unicidade no banco passa a ser composta; migração via novo changeset Liquibase (não altera o changeset original, para preservar o checksum de ambientes já migrados).
- `OrderRepository.get_by_idempotency_key` e o fallback de corrida em `save_with_outbox` passam a filtrar por `idempotency_key` **e** `requester_id`.
- Testes unitários e de integração (in-memory) cobrem explicitamente o cenário de regressão: mesma `Idempotency-Key`, `requester_id` diferentes → duas ordens distintas.
