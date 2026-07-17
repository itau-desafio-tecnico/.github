# ADR 0001 — Compute: ECS Fargate

## Status
Aceito

## Contexto
Precisamos rodar dois microsserviços (Python/FastAPI e Java/Spring Boot) na AWS, com deploy via GitHub Actions e Terraform, minimizando esforço operacional dado o escopo de um desafio técnico.

## Decisão
Usar Amazon ECS com Fargate (containers gerenciados, sem servidor) em vez de EKS (Kubernetes) ou Lambda.

## Alternativas consideradas
- **EKS**: mais alinhado a ambientes corporativos de grande escala, mas exige cluster Kubernetes, add-ons (ingress, autoscaler) e Terraform bem mais complexo — custo de setup desproporcional ao escopo.
- **Lambda**: ótimo para o consumidor assíncrono, mas adaptar Spring Boot a cold start/runtime serverless traz fricção; FastAPI teria melhor fit mas perderíamos uniformidade entre os dois serviços.

## Consequências
- Terraform mais simples (cluster, task definition, service, ALB).
- Deploy via atualização de task definition, fácil de automatizar no GitHub Actions.
- Menos elasticidade fina que Lambda para picos, mas adequado ao volume esperado do desafio.
