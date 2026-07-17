# ADR 0002 — Mensageria: SNS + SQS

## Status
Aceito

## Contexto
O `order-service` precisa publicar o evento `OrderCreated` para processamento assíncrono, com garantia de entrega e possibilidade de múltiplos consumidores no futuro.

## Decisão
Usar SNS (tópico `order-created`) com uma subscription SQS (fila de processamento) e uma Dead Letter Queue (DLQ) para mensagens com falha recorrente.

## Alternativas consideradas
- **Amazon MSK (Kafka)**: mais próximo do que ambientes bancários usam em produção, mas exige cluster gerenciado, mais custo e complexidade de Terraform — desproporcional ao escopo do desafio.
- **EventBridge**: bom para roteamento de eventos por regra entre múltiplos serviços, mas menos natural para o padrão fila-consumidor único que o desafio pede.

## Consequências
- SNS permite adicionar novos consumidores (novas subscriptions SQS) sem alterar o publisher.
- SQS garante entrega *at-least-once* — o consumidor precisa ser idempotente (verificar `event_id` já processado).
- DLQ isola mensagens com falha repetida para investigação, sem bloquear o restante da fila.
