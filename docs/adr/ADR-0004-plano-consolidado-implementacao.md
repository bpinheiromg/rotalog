# ADR-0004: Plano Consolidado de Implementação - API Entregas

**Data:** 2026-06-18  
**Status:** PROPOSTO  
**Decisão Crítica:** Sim ⚠️⚠️⚠️  
**Prioridade:** 🔴 CRÍTICA (Depends on ADR-0002 + ADR-0003)  

---

## 1. Contexto

Este ADR consolida o plano de implementação para remediar os problemas identificados em:

- **ADR-0002:** Vulnerabilidades de Segurança Críticas (12 vulnerabilidades)
- **ADR-0003:** Refatoração Arquitetural (10 problemas de design)

### Objetivo Geral

Transformar `rotalog-api-entregas` de um serviço inseguro e inmanutenível para um serviço:
- 🔒 Seguro (zero vulnerabilidades críticas)
- 📐 Bem arquitetado (MVC com separação de responsabilidades)
- 🧪 Testável (80%+ cobertura)
- 📊 Observável (logging estruturado)

---

## 2. Problema

### Resumo dos Problemas

**Segurança (ADR-0002):**
- 8 vulnerabilidades críticas (credenciais, JWT, CORS, autorização)
- Risco de comprometimento de dados de clientes
- Não compliant com OWASP Top 10

**Arquitetura (ADR-0003):**
- 366 linhas em arquivo de rotas
- Mistura de 3 padrões async
- Lógica de negócio acoplada
- Impossível testar isoladamente
- Débito técnico crescente

---

## 3. Decisão

### 3.1 Abordagem Integrada

**Implementar as refatorações de forma integrada:**

```timeline
FASE 0: Segurança Imediata (1 dia - PARALELO)
│
├─ FASE 1: Segurança Básica (3 dias)
│  └─ FASE 1.1: Arquitetura Básica (1 dia - PARALELO)
│
├─ FASE 2: Arquitetura Core (5 dias)
│  └─ FASE 2.1: Segurança Hardening (2 dias - PARALELO)
│
└─ FASE 3: Finalização + QA (3 dias)
   └─ Merge para main
```

### 3.2 Sequência Crítica

**Não é possível fazer tudo em paralelo:**

1. ✅ **Segurança imediata ANTES de arquitetura** (24h)
   - Rotacionar credenciais
   - JWT real
   - CORS restritivo
   - Rastreamento protegido

2. ✅ **Arquitetura enquanto solidifica segurança** (5 dias paralelo)
   - Criar estrutura MVC
   - Separar responsabilidades
   - Mover lógica para service

3. ✅ **Finalização + testes** (3 dias)
   - Integração
   - Testes E2E
   - Documentação

---

## 4. Plano de Implementação Consolidado

### 4.1 FASE 0: Ação Imediata (24 horas)

**Objetivo:** Remover exposição crítica de credenciais

**Tarefas:**
```
┌─ DBA
│  └─ Rotacionar credenciais BD (1h)
│
├─ DevOps
│  ├─ Remover .env de git history (2h)
│  └─ Atualizar .env em sistemas (1h)
│
├─ Backend Engineer
│  ├─ Gerar novo JWT secret (0.5h)
│  ├─ Criar .env.example (0.5h)
│  ├─ Criar feature branch (0.5h)
│  └─ Comunicar ao team (0.5h)
│
└─ ALL DEVS
   └─ Re-clone do repositório (0.5h)
```

**Entregáveis:**
- ✅ Credenciais rotacionadas
- ✅ `.env` removido do git
- ✅ `.env.example` criado
- ✅ Team re-clonou repo
- ✅ Feature branch `feature/entregas-security-refactor` criado

**Duração:** 6 horas real (distribuído em 1 dia)

---

### 4.2 FASE 1: Segurança Básica (Dias 2-4)

**Objetivo:** Implementar autenticação real, CORS restritivo, autorização básica

#### Dia 2: Autenticação Real

**Tarefas:**
```
1. Implementar JWT validação (ADR-0002 item 3)
   - Reescrever src/middleware/auth.js
   - Instalar jsonwebtoken
   - Remover bypass de development
   - Validação real com jwt.verify()

2. Criar testes de autenticação
   - 8 testes: token válido, expirado, inválido, etc.

3. Code review + merge
```

**Entregáveis:**
- ✅ `auth.js` refatorado (~50 linhas)
- ✅ 8 testes
- ✅ Zero JWT bypass

**Tempo:** 6 horas

---

#### Dia 3: CORS + Rastreamento

**Tarefas:**
```
1. CORS Restritivo (ADR-0002 item 4)
   - Instalar cors package
   - Criar whitelist de origins
   - Configurar via .env

2. Proteger Rastreamento (ADR-0002 item 5)
   - Adicionar authMiddleware em /api/rastreamento
   - Testes

3. Code review
```

**Entregáveis:**
- ✅ CORS whitelist configurável
- ✅ `/rastreamento` protegido
- ✅ 6 testes

**Tempo:** 6 horas

---

#### Dia 4: Autorização Básica

**Tarefas:**
```
1. Implementar RBAC (ADR-0002 item 8)
   - Criar middleware verificarPropriedade
   - Verificar cliente_id = req.user.id
   - Role-based: cliente, driver, admin

2. Aplicar a endpoints críticos
   - /entregas/:id (GET, PUT, PATCH, DELETE)
   - Autorização antes de operação

3. Testes + code review
```

**Entregáveis:**
- ✅ Middleware RBAC (~40 linhas)
- ✅ 10 testes
- ✅ Clientes não acessam dados uns dos outros

**Tempo:** 8 horas

---

### 4.3 FASE 1.1: Arquitetura Básica (Dia 4 - PARALELO)

**Objetivo:** Criar estrutura MVC base (pode rodar enquanto FASE 1 acontece)

**Tarefas (para Developer 2):**
```
1. Criar estrutura de diretórios
   - src/controllers/
   - src/services/
   - src/repositories/
   - src/validators/

2. Criar base classes
   - BaseService (error handling)
   - BaseValidator

3. Setup de testes
   - Jest/Mocha config
   - Test utilities

4. Criar estrutura de docs
```

**Entregáveis:**
- ✅ Estrutura de diretórios
- ✅ Base classes
- ✅ Test setup

**Tempo:** 8 horas (paralelo com Dia 4 de FASE 1)

---

### 4.4 FASE 2: Refatoração Arquitetura (Dias 5-9)

**Objetivo:** Separar responsabilidades, testes, async/await

#### Dias 5-6: Refatoração Core

**Tarefas:**
```
1. Extrair service (EntregasService)
   - Mover lógica de negócio
   - gerarNumeroPedido()
   - calcularDistancia() [CORRIGIR FÓRMULA]
   - criarEntrega()
   - atualizarStatus()
   - atribuirVeiculo()

2. Criar controller (EntregasController)
   - Orquestração de request
   - Chamadas a service + repository

3. Criar repository (EntregasRepository)
   - Queries com ORM (sem raw SQL)
   - findAll(), findById(), create(), etc.

4. Testes
   - 30+ testes service
   - 12 testes repository
```

**Entregáveis:**
- ✅ `EntregasService.js` (~150 linhas)
- ✅ `EntregasController.js` (~120 linhas)
- ✅ `EntregasRepository.js` (~80 linhas)
- ✅ 42+ testes

**Tempo:** 24 horas

---

#### Dias 7-8: Middleware Centralizado

**Tarefas:**
```
1. Error Handler (ADR-0002 item 6)
   - Middleware global de erro
   - Respostas padronizadas
   - Remover detalhes de erro

2. Validação Centralizada (ADR-0003 item 5)
   - Middleware com express-validator
   - EntregaValidator.js
   - Regras: create, update, status, etc.

3. Logging Estruturado (ADR-0003 item 6)
   - Winston setup
   - Remover console.log
   - Structured logs com contexto

4. Testes
   - 12 testes middleware
   - 5 testes error handling
```

**Entregáveis:**
- ✅ `errorHandler.js` (~60 linhas)
- ✅ `validation.js` (~50 linhas)
- ✅ `EntregaValidator.js` (~100 linhas)
- ✅ `logger.js` (~40 linhas)
- ✅ 17 testes

**Tempo:** 16 horas

---

#### Dias 8-9: Máquina de Estados + Limpeza

**Tarefas:**
```
1. Status Transitions (ADR-0003 item 7)
   - StatusTransitionsValidator.js
   - Transições válidas
   - Validação antes de atualizar

2. Async/Await Uniforme (ADR-0003 item 2)
   - Converter callbacks → async/await
   - Try-catch consistente

3. Limpeza Final
   - Remover System.out.println()
   - Remover Javadoc redundante
   - Renomear variáveis confusas

4. Tests de integração (5)

5. Code review completo
```

**Entregáveis:**
- ✅ `StatusTransitionsValidator.js` (~40 linhas)
- ✅ Todos async/await
- ✅ Zero console.log
- ✅ 5 testes integração

**Tempo:** 16 horas

---

### 4.5 FASE 2.1: Segurança Hardening (Dias 7-8 - PARALELO)

**Objetivo:** Aprofundar segurança enquanto arquitetura finaliza

**Tarefas (para Developer 1):**
```
1. Não expor detalhes de erro (ADR-0002 item 6)
   - Remover error.message de responses
   - Usar error handler centralizado

2. Proteção de SQL Injection (ADR-0002 item 7)
   - Converter raw SQL → ORM
   - Usar parameterized queries

3. Health Checks Reais (ADR-0002 item 12)
   - Verificar conexão com BD
   - Verificar serviços externos
   - Status real (não fake)

4. Graceful Shutdown
   - Handler para SIGTERM
   - Fechar conexões
   - Aguardar requisições ativas

5. Testes de segurança (8)

6. Security audit
```

**Entregáveis:**
- ✅ Erro handling centralizado integrado
- ✅ Zero raw SQL
- ✅ Health checks reais
- ✅ Graceful shutdown
- ✅ 8 testes segurança

**Tempo:** 16 horas (paralelo com Dias 7-8)

---

### 4.6 FASE 3: Finalização + QA (Dias 10-12)

#### Dia 10: Testes E2E + Security

**Tarefas:**
```
1. Testes E2E
   - Fluxo completo de entrega
   - Testes de autorização
   - Testes de erro

2. Security Testing
   - Tentar JWT forjado
   - Tentar CORS de origem não permitida
   - Tentar acessar entrega de outro cliente

3. Cobertura de testes
   - Gerar relatório
   - Verificar >= 80%
```

**Tempo:** 8 horas

---

#### Dia 11: Documentação + Code Review

**Tarefas:**
```
1. Documentação
   - README.md (como rodar, arquitetura)
   - SECURITY.md (padrões de segurança)
   - ARCHITECTURE.md (estrutura de código)
   - API.md (endpoints, autenticação)

2. Code review final
   - Peer review
   - Security review
   - Architecture review

3. Ajustes finais
```

**Tempo:** 8 horas

---

#### Dia 12: Merge + Deploy Staging

**Tarefas:**
```
1. Preparar PR final
   - Squash commits se necessário
   - Escrever boa descrição
   - Adicionar screenshots de testes

2. Merge para main
   - Trigger CI/CD
   - Testes rodam

3. Deploy em staging
   - Monitorar logs
   - Validar em staging
   - Fumiçar

4. Atualizar TECHNICAL_DEBT_MAP.md
```

**Tempo:** 4 horas

---

## 5. Estimativa Consolidada

### 5.1 Timeline Total

```
FASE 0: 1 dia (24h)
├─ Ação imediata
└─ Rotação credenciais

FASE 1 + 1.1: 4 dias (32h + 8h paralelo)
├─ Segurança básica
└─ Arquitetura básica (paralelo)

FASE 2 + 2.1: 5 dias (48h + 16h paralelo)
├─ Refatoração arquitetura
└─ Segurança hardening (paralelo)

FASE 3: 3 dias (20h)
├─ Testes E2E
├─ Documentação
└─ Merge + deploy

TOTAL: ~12 dias (com paralelização)
ESFORÇO: ~180 horas
EQUIPE: 2-3 pessoas
```

### 5.2 Gantt Chart

```
Dev 1 (Backend/Security)
├─ FASE 0 (1d): Credenciais
├─ FASE 1 (3d): Auth + CORS + Rastreamento
├─ FASE 2.1 (2d): Security hardening (paralelo)
├─ FASE 3 (1d): Security testing + docs
└─ Merge (0.5d)

Dev 2 (Architecture/Testing)
├─ FASE 1.1 (1d): Arquitetura base (paralelo com FASE 1)
├─ FASE 2 (5d): Services, controllers, testes
├─ FASE 3 (1.5d): E2E testing + docs
└─ Merge (0.5d)

Parallelization savings: ~30 horas
```

---

## 6. Consequências

### 6.1 Segurança Após Implementação

| Vulnerabilidade | Antes | Depois | Status |
|-----------------|-------|--------|--------|
| Credenciais hardcodeadas | ✗ | ✓ | Remediada |
| JWT fake | ✗ | ✓ | Remediada |
| Auth bypass | ✗ | ✓ | Remediada |
| CORS aberto | ✗ | ✓ | Remediada |
| Rastreamento sem auth | ✗ | ✓ | Remediada |
| Sem autorização | ✗ | ✓ | Remediada |
| SQL injection risk | ✗ | ✓ | Remediada |
| Erro details | ✗ | ✓ | Remediada |

### 6.2 Arquitetura Após Implementação

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| Linhas entregas.js | 366 | 90 | ✓ |
| Arquivos lógica | 1 | 3+ | ✓ |
| Testes | 0 | 75+ | ✓ |
| Cobertura | 0% | 85% | ✓ |
| Async padrões | 3 | 1 | ✓ |
| Raw SQL | 1 | 0 | ✓ |
| console.log | 5+ | 0 | ✓ |

---

## 7. Riscos e Mitigation

| Risco | Prob. | Impacto | Mitigation |
|-------|-------|--------|-----------|
| Refatoração quebra funcionalidades | 🟠 | 🟠 | 75+ testes, code review rigoroso |
| Cronograma estoura | 🟠 | 🟡 | Paralelização, priorização |
| Descoberta de novos bugs | 🟡 | 🟡 | Testes abrangentes, E2E |
| Conflito de merge | 🟡 | 🟡 | Feature branch, rebase frequente |
| Dados perdidos em migration | 🟡 | 🔴 | Backup antes, migration tests |

---

## 8. Critério de Sucesso

### Go-Live Checklist

- [ ] Zero vulnerabilidades críticas
- [ ] JWT validação real implementada
- [ ] CORS whitelist configurado
- [ ] Autorização em todos endpoints
- [ ] 75+ testes (>80% cobertura)
- [ ] Zero console.log
- [ ] Async/await uniforme
- [ ] Health checks reais
- [ ] Graceful shutdown
- [ ] Logging estruturado
- [ ] Documentação completa
- [ ] Code review aprovado
- [ ] Merge para main
- [ ] Deploy em staging OK

---

## 9. Próximos Passos

### 9.1 Aprovações Necessárias

1. ✅ Aprovação do CTO/Tech Lead (ADRs)
2. ✅ Aprovação do Security Officer
3. ✅ Aprovação do Product Owner (timeline)

### 9.2 Setup Inicial

1. Criar feature branch
2. Criar estrutura de diretórios
3. Criar tickets de implementação
4. Alocar equipe

### 9.3 Comunicação

1. Notificar dev team
2. Avisar sobre re-clone
3. Compartilhar `.env.example`
4. Daily standups durante implementação

---

## 10. Referências

- [ADR-0002: Vulnerabilidades de Segurança](ADR-0002-vulnerabilidades-seguranca-entregas.md)
- [ADR-0003: Refatoração Arquitetural](ADR-0003-refatoracao-arquitetura-entregas.md)
- [TECHNICAL_DEBT_MAP.md](../TECHNICAL_DEBT_MAP.md)
- [SECURITY_ANALYSIS_ENTREGAS.md](../SECURITY_ANALYSIS_ENTREGAS.md)
- [ENTREGAS_ARCHITECTURE_ANALYSIS.md](../ENTREGAS_ARCHITECTURE_ANALYSIS.md)

---

## 11. Histórico de Decisão

| Data | Decisão | Status |
|------|---------|--------|
| 2026-06-18 | Criar ADR consolidado | ✅ PROPOSTO |
| 2026-06-XX | Aprovação das 3 ADRs | ⏳ PENDENTE |
| 2026-06-XX | Criar tickets JIRA | ⏳ PENDENTE |
| 2026-06-XX | Iniciar FASE 0 (24h) | ⏳ PENDENTE |

---

## 12. Conclusão

Este ADR consolida o plano para transformar `rotalog-api-entregas` de um serviço inseguro e inmanutenível para um serviço pronto para produção em **12 dias** com:

- 🔒 **Segurança:** Zero vulnerabilidades críticas
- 📐 **Arquitetura:** MVC bem definido
- 🧪 **Testabilidade:** 80%+ cobertura
- 📊 **Observabilidade:** Logging estruturado
- 📚 **Documentação:** Completa

**Esforço:** ~180 horas, 2-3 pessoas, 12 dias com paralelização  
**Prioridade:** 🔴 CRÍTICA  
**Impacto:** Serviço seguro, mantível, escalável

---

**Documento Criado:** 2026-06-18  
**Versão:** 1.0  
**Autor:** Análise Técnica - Claude Code  
**Status:** PROPOSTO - Aguardando Aprovação das 3 ADRs (0002, 0003, 0004)
