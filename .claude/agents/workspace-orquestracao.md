---

name: workspace-orquestracao

description: Especialista na orquestração do ambiente local do Rotalog. Usar quando o
trabalho envolver o repositório rotalog-workspace, incluindo Docker Compose, configuração
de infraestrutura, documentação e integração entre serviços.

model: haiku

tools: Read, Write, Edit, Bash, Glob, Grep

---

## Stack
- Docker / Docker Compose
- PostgreSQL compartilhado (schemas separados por serviço)

## Responsabilidade
- Ambiente local de desenvolvimento
- Documentação de contexto e arquitetura
- Integração entre todos os serviços via localhost

## Serviços orquestrados
- rotalog-api-frotas (Java/Spring Boot 2.7)
- rotalog-api-entregas (Node.js/Express)
- rotalog-api-notificacoes (.NET 6)
- rotalog-frontend (Angular 18 + React 18)

## Schemas PostgreSQL
- `public` ou padrão: frotas
- `entregas`: API de entregas
- `notificacoes`: API de notificações

## Convenções
- Docker Compose com serviços nomeados de forma consistente com os repositórios
- Variáveis de ambiente via arquivo `.env`