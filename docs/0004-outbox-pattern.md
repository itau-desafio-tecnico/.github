# ADR 0004 — Outbox pattern para publicação de eventos

## Status
Aceito

## Contexto
Após criar a ordem, o serviço precisa publicar o evento `OrderCreated` no SNS. Escrever no banco e publicar no SNS são duas operações contra sistemas diferentes — sem cuidado, um pode ter sucesso e o outro falhar (dual-write problem), quebrando a consistência entre o que foi persistido e o que foi comunicado.

## Decisão
Usar o padrão **Transactional Outbox**: na mesma transação local que grava a ordem, gravar também uma linha em `outbox_events` (com o payload do evento e `status = PENDING`). Um dispatcher assíncrono (worker em background no próprio processo) faz polling periódico dessa tabela, publica no SNS e marca o evento como `PUBLISHED`.

## Alternativas consideradas
- **Publicar direto no SNS dentro do use case, antes/depois do commit**: mais simples, mas sujeito ao dual-write problem — se o commit falhar após publicar (ou o publish falhar após o commit), banco e evento ficam inconsistentes.
- **CDC com Debezium lendo o WAL do Postgres**: elimina o polling, mas exige Kafka Connect na infra — incompatível com a decisão de usar SNS/SQS (ADR 0002) e desproporcional ao escopo.

## Consequências
- Garante que todo `OrderCreated` persistido será eventualmente publicado (at-least-once), e nenhum evento é publicado sem a ordem correspondente existir.
- Exige tabela extra (`outbox_events`) e um processo de dispatch com retry/backoff e contagem de tentativas.
- Consumidor da fila deve ser idempotente, já que a combinação outbox + SQS garante "pelo menos uma vez", não "exatamente uma vez" ponta a ponta.
