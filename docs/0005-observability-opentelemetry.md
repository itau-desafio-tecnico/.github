# ADR 0005 — Observabilidade: OpenTelemetry + Grafana/Prometheus/Jaeger

## Status
Aceito

## Contexto
Os dois microsserviços (stacks diferentes — Python e Java) precisam de logs, métricas e tracing distribuído correlacionados, especialmente para depurar a chamada `order-service` → `requester-service` e a publicação/consumo do evento assíncrono.

## Decisão
Instrumentar ambos os serviços com OpenTelemetry (OTel SDK/auto-instrumentation), exportando via OTLP para um OTel Collector. O Collector encaminha métricas para Prometheus, traces para Jaeger e logs estruturados (correlacionados por `trace_id`) via Grafana/Loki. Toda a stack roda self-hosted em ECS Fargate.

## Alternativas consideradas
- **AWS nativo (CloudWatch + X-Ray)**: menos peças de infra e integração mais direta com ECS, mas cria acoplamento com a AWS (vendor lock-in) e X-Ray tem suporte mais limitado para correlação profunda entre serviços poliglotas.
- **Híbrido (OTel instrumentando, exportando para CloudWatch/X-Ray)**: reduz infra própria, mas perde as dashboards/consultas mais ricas do Grafana e a portabilidade de trocar de cloud sem reinstrumentar.

## Consequências
- Instrumentação vendor-neutral: os serviços não conhecem o backend de observabilidade, apenas exportam via OTLP.
- Mais peças de infraestrutura para provisionar e manter (Collector, Prometheus, Grafana, Jaeger) via Terraform.
- Dashboards e traces ficam completamente sob nosso controle, sem depender de consoles específicos da AWS.
