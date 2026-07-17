# ADR 0007 — CI/CD: GitHub Actions com credenciais AWS estáticas

## Status
Aceito

## Contexto
Os workflows de build/deploy e o `terraform apply` precisam autenticar contra a AWS a partir do GitHub Actions.

## Decisão
Usar `aws-actions/configure-aws-credentials@v4` com `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` armazenadas como GitHub Secrets, região `sa-east-1` fixa.

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: sa-east-1
```

Cada workflow também é disparável manualmente (`workflow_dispatch`), além do gatilho automático em push/PR.

## Alternativas consideradas
- **OIDC (OpenID Connect) entre GitHub Actions e AWS**: elimina a necessidade de chaves de longa duração armazenadas como secret, é a prática recomendada pela AWS. Não foi adotada nesta rodada por decisão explícita — o ambiente AWS do desafio já usa autenticação estática.

## Consequências
- Simplicidade de configuração: basta cadastrar duas secrets no repositório.
- Chaves de longa duração ficam armazenadas como GitHub Secret — exigem rotação manual periódica e princípio de menor privilégio na política IAM associada.
- Migração futura para OIDC é possível sem mudança de arquitetura da aplicação, apenas do step de autenticação do workflow.
