# ADR 0003 — Idempotência via header Idempotency-Key

## Status
Aceito

## Contexto
A mesma requisição de criação de ordem pode chegar repetida (retries de cliente/gateway). O `numero_ordem` é gerado pelo próprio `order-service` no momento da criação, então não pode servir como chave de deduplicação — ele não existe antes do processamento.

## Decisão
O cliente da API envia um header `Idempotency-Key` (UUID gerado pelo próprio cliente) em `POST /orders`. Essa chave tem constraint única na tabela `orders`. Se a chave já existe, a resposta armazenada da primeira execução é retornada (mesmo `numero_ordem`, mesmo status HTTP), sem reprocessar nem publicar um novo evento.

## Alternativas consideradas
- **Chave de negócio derivada do corpo da requisição** (ex.: hash de `solicitante_id` + campos da ordem): funciona sem exigir header extra, mas exige uma definição rígida de quais campos compõem a igualdade "mesma ordem", o que é frágil se o payload mudar sutilmente entre retries (ex.: timestamp do cliente).
- **Sem controle de idempotência, confiando em deduplicação da fila**: SQS/SNS não removem duplicidade de chamadas HTTP; não resolveria o requisito.

## Consequências
- Contrato de API exige o header em toda chamada de criação — precisa estar documentado no README/OpenAPI.
- Requer tabela auxiliar (ou coluna) guardando a resposta original associada à chave, para retorno idempotente exato.
- Simples de testar: dado o mesmo `Idempotency-Key`, duas chamadas retornam o mesmo `numero_ordem` e não duplicam linha em `orders` nem evento em `outbox_events`.
