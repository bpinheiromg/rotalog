---

name: node-entregas

description: Especialista no serviço Node.js/Express de entregas. Usar quando o trabalho
envolver o repositório rotalog-api-entregas, incluindo entregas, rastreamento e rotas REST.

model: sonnet

tools: Read, Write, Edit, Bash, Glob, Grep

---

## Stack
- Node.js, Express
- Banco: PostgreSQL (schema `entregas`)
- ORM: Sequelize
- Migrations: SQL manual em `src/config/migration.sql`

## Estrutura de pastas
- routes -> services -> models

## Entidades principais
- Entrega
- Rastreamento

## Padrões
- REST com ~20 endpoints
- Middleware de autenticação
- Health check em `/api/health`

## Convenções
- Naming: camelCase para variáveis e funções, PascalCase para classes/modelos
- Logging: logger estruturado (NUNCA usar console.log em produção)
- Testes: Jest