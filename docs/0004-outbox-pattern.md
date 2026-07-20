# ADR 0004 — Outbox pattern para publicação de eventos

## Status
Aceito (revisado — ver seção "Revisão: claim pattern para múltiplas réplicas")

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

## Revisão: claim pattern para múltiplas réplicas

**Contexto da revisão.** O desenho original assumia implicitamente uma única instância do `order-service` rodando o dispatcher (`desired_count = 1`). Ao avaliar escalabilidade horizontal (ver módulo `ecs` do `infrastructure`, que passou a suportar Application Auto Scaling), identificamos que o polling original (`SELECT ... WHERE status = 'PENDING'`, sem lock) duplicaria publicações no SNS se duas ou mais instâncias rodassem o dispatcher ao mesmo tempo: ambas poderiam selecionar e publicar o mesmo evento no mesmo ciclo.

**Decisão revisada.** O dispatcher passa a **reivindicar** (`claim`) eventos em vez de apenas lê-los. Numa única transação curta: `SELECT ... FOR UPDATE SKIP LOCKED` sobre eventos `PENDING` (ou `PROCESSING` presos além de um timeout configurável) e, na mesma transação, `UPDATE` marcando essas linhas como `PROCESSING` com `claimed_at = now()`, antes de liberar o lock. A partir daí, o que impede outra instância de pegar o mesmo evento não é mais o lock (liberado em milissegundos) — é o status `PROCESSING`, que só volta a `PENDING`/reivindicável se a publicação falhar ou se o timeout de processamento expirar (instância morta no meio do trabalho, ex.: task do ECS substituída em um deploy).

**Por que não segurar o lock durante a publicação no SNS**: chamar uma API externa (rede, potencialmente lenta) com um lock de linha aberto é um antipadrão — outras instâncias ficariam bloqueadas (não puladas) por toda a duração da chamada HTTP. O claim resolve isso reivindicando rápido (uma transação de milissegundos) e usando o status como sinalizador de posse pelo tempo que a publicação de fato levar.

**Consequências da revisão**:
- Nova coluna `claimed_at` em `outbox_events` e novo status `PROCESSING` no enum `OutboxStatus`.
- Sem essa mudança, ligar Application Auto Scaling no `order-service` teria introduzido duplicação silenciosa de eventos `OrderCreated` publicados no SNS — o SKIP LOCKED sozinho, aplicado ingenuamente ao método de leitura original, não teria efeito real, já que a sessão/transação da leitura fechava (liberando o lock) antes do dispatcher sequer começar a publicar.
- Uma instância que morre com eventos em `PROCESSING` não perde esses eventos permanentemente — eles voltam a ser reivindicáveis após `outbox_processing_timeout_seconds` (padrão 60s), evitando perda silenciosa de evento, um risco mais grave do que a duplicação que este ADR já assumia como aceitável via "at-least-once".
