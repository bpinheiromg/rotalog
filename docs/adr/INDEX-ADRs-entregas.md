# 📋 Índice de ADRs - Rotalog API Entregas

**Data de Atualização:** 2026-06-18  
**Status:** 3 ADRs PROPOSTOS - Aguardando Aprovação  

---

## 🎯 Visão Geral

Três Architecture Decision Records (ADRs) foram criados para documentar e coordenar a remediação de:

1. **12 vulnerabilidades críticas de segurança** (ADR-0002)
2. **10 problemas arquiteturais graves** (ADR-0003)
3. **Plano consolidado de implementação** (ADR-0004)

---

## 📑 ADRs Criados

### ADR-0001: Refatoração da Classe VeiculoService

- **Status:** PROPOSTO (criado anteriormente)
- **Prioridade:** 🔴 CRÍTICA
- **Escopo:** rotalog-api-frotas (Java)
- **Objetivo:** Separar God Class em 7 serviços especializados
- **Esforço:** 247-337 horas (6-8 semanas)
- **Arquivo:** [`ADR-0001-refatoracao-veiculoservice.md`](ADR-0001-refatoracao-veiculoservice.md)

---

### ADR-0002: Vulnerabilidades de Segurança Críticas - API Entregas

- **Status:** ✅ PROPOSTO
- **Prioridade:** 🔴 CRÍTICA (Requer ação em 24h)
- **Escopo:** rotalog-api-entregas (Node.js)
- **Problemas:** 12 vulnerabilidades críticas

#### Vulnerabilidades Principais

| # | Vulnerabilidade | Risco | CVSS |
|---|-----------------|-------|------|
| 1 | Credenciais hardcodeadas em `.env` | Comprometimento total | 9.8 |
| 2 | JWT validação fake | Bypass autenticação | 9.7 |
| 3 | Auth bypass em development | Sem segurança staging | 8.5 |
| 4 | CORS completamente aberto | CSRF, XSS, data theft | 8.2 |
| 5 | Rastreamento sem autenticação | Privacy violation | 7.8 |
| 6 | Sem autorização (RBAC) | Data leakage | 7.9 |
| 7 | Raw SQL sem parametrização | SQL Injection | 8.0 |
| 8 | Detalhes de erro expostos | Information disclosure | 5.3 |
| 9 | Logging com console.log | - | - |
| 10 | Sem máquina de estados | Business logic violation | - |
| 11 | Geração de ID com Math.random | Previsibilidade | - |
| 12 | Health check fake | Indisponibilidade não detectada | - |

#### Ações Imediatas (24h)

1. Rotacionar credenciais de BD
2. Remover `.env` do histórico git
3. Criar `.env.example` com placeholders
4. Comunicar ao team para re-clone

#### Fases de Remediação

- **FASE 0:** Ação imediata (1 dia)
- **FASE 1:** Segurança básica (3 dias) - Auth real, CORS, Rastreamento
- **FASE 2:** Hardening (2 dias) - Error handling, SQL injection, health checks
- **FASE 3:** QA (1 dia) - Testes, validação

**Esforço:** 94 horas (2 semanas, 1-2 pessoas)

**Arquivo:** [`ADR-0002-vulnerabilidades-seguranca-entregas.md`](ADR-0002-vulnerabilidades-seguranca-entregas.md)

---

### ADR-0003: Refatoração Arquitetural - Separação de Responsabilidades

- **Status:** ✅ PROPOSTO
- **Prioridade:** 🟠 ALTA (Quadrante 2 do TECHNICAL_DEBT_MAP)
- **Escopo:** rotalog-api-entregas (Node.js) - arquivo `src/routes/entregas.js`
- **Problemas:** 10 problemas arquiteturais

#### Problemas Principais

| # | Problema | Impacto |
|---|----------|---------|
| 1 | 366 linhas em um arquivo | Tamanho excessivo |
| 2 | Mistura de 3 padrões async | Inconsistência |
| 3 | Lógica de negócio na rota | Sem reutilização |
| 4 | Fórmula de distância incorreta | Cálculos errados (50% erro) |
| 5 | SQL queries diretas | SQL injection risk |
| 6 | Validação inline duplicada | Sem padronização |
| 7 | Logging com console.log | Não estruturado |
| 8 | Sem máquina de estados | Transições inválidas |
| 9 | Sem autorização (RBAC) | Cross-tenant data access |
| 10 | Sem testes possível | Cobertura 0% |

#### Refatoração Proposta

**Arquitetura MVC:**
```
routes/entregas.js (90 linhas)
  ↓
controllers/entregasCtrl.js (120 linhas)
  ↓
services/entregasService.js (150 linhas)
  ↓
repositories/entregasRepo.js (80 linhas)

+ Middleware centralizado (validation, error, auth)
+ Logging estruturado
+ 75+ testes (80%+ cobertura)
```

#### Fases de Refatoração

- **FASE 1:** Estrutura base (2 dias)
- **FASE 2:** Refatoração core (5 dias)
- **FASE 3:** Middleware centralizado (2 dias)
- **FASE 4:** Async/Await + Logging (2 dias)
- **FASE 5:** Máquina de estados (1 dia)
- **FASE 6:** Finalização (1 dia)

**Esforço:** 108 horas (3 semanas, 1-2 pessoas)

**Arquivo:** [`ADR-0003-refatoracao-arquitetura-entregas.md`](ADR-0003-refatoracao-arquitetura-entregas.md)

---

### ADR-0004: Plano Consolidado de Implementação

- **Status:** ✅ PROPOSTO
- **Prioridade:** 🔴 CRÍTICA
- **Escopo:** Consolidação de ADR-0002 + ADR-0003
- **Objetivo:** Timeline integrada de implementação

#### Abordagem Integrada

**As refatorações de segurança e arquitetura são executadas em paralelo:**

```
FASE 0 (1d):  Segurança imediata
├─ FASE 1 (3d): Segurança básica
│  └─ FASE 1.1 (1d - paralelo): Arquitetura base
├─ FASE 2 (5d): Refatoração arquitetura
│  └─ FASE 2.1 (2d - paralelo): Segurança hardening
└─ FASE 3 (3d): Testes E2E + Finalização
   └─ Merge para main + Deploy staging
```

**Equipe:** 2-3 pessoas  
**Timeline:** 12 dias (com paralelização)  
**Esforço Total:** ~180 horas

#### Ganho da Paralelização

- **Sequential:** ~220 horas / 4 semanas
- **Paralelo:** ~180 horas / 2 semanas
- **Ganho:** 40 horas (18%)

#### Entregáveis Finais

✅ Zero vulnerabilidades críticas  
✅ JWT validação real  
✅ CORS restritivo  
✅ Autorização implementada  
✅ Arquitetura MVC limpa  
✅ 75+ testes (85% cobertura)  
✅ Logging estruturado  
✅ Documentação completa  

**Arquivo:** [`ADR-0004-plano-consolidado-implementacao.md`](ADR-0004-plano-consolidado-implementacao.md)

---

## 🔗 Relações entre ADRs

```
ADR-0001 (Java - Frotas)
    ↓
    └─ Padrão arquitetural estabelecido

ADR-0002 (Segurança - Entregas)
    ├─ Depende de: Decisão de implementar
    └─ Alimenta: ADR-0004

ADR-0003 (Arquitetura - Entregas)
    ├─ Depende de: Decisão de refatorar
    └─ Alimenta: ADR-0004

ADR-0004 (Plano Consolidado)
    ├─ Depende de: ADR-0002 + ADR-0003
    ├─ Coordena: Implementação em paralelo
    └─ Resultado: Serviço seguro + limpo
```

---

## 📊 Matriz de Decisão

| Critério | ADR-0002 | ADR-0003 | Prioridade |
|----------|----------|----------|-----------|
| **Urgência** | 24h | 2 semanas | 🔴 0002 primeiro |
| **Impacto** | Segurança crítica | Qualidade código | 🔴 0002 primeiro |
| **Dependência** | Nenhuma | Nenhuma | ✅ Podem rodar paralelo |
| **Risco se não fizer** | Dados comprometidos | Débito crescente | 🔴 Fazer ambas |
| **Equipe requerida** | 1-2 backend | 1-2 backend | ✅ 2 pessoas rodando paralelo |

---

## ✅ Checklist Pré-Implementação

### Aprovações Necessárias

- [ ] CTO/Tech Lead aprova ADR-0002
- [ ] CTO/Tech Lead aprova ADR-0003
- [ ] CTO/Tech Lead aprova ADR-0004
- [ ] Security Officer revisa ADR-0002
- [ ] Product Owner aprova timeline

### Setup Inicial

- [ ] Feature branch criado: `feature/entregas-security-refactor`
- [ ] Estrutura de diretórios criada
- [ ] Base classes criadas
- [ ] Jest/Mocha configurado
- [ ] `.env.example` criado

### Comunicação

- [ ] Dev team notificado
- [ ] Timeline compartilhada
- [ ] `.env.example` disponível
- [ ] Daily standups agendados

### Recursos

- [ ] 2-3 pessoas alocadas
- [ ] 12 dias no calendário
- [ ] ~180 horas orçadas
- [ ] Ferramentas (jwt, cors, winston, etc.)

---

## 📈 Métricas de Sucesso Consolidadas

### Segurança (ADR-0002)

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| Vulnerabilidades críticas | 8 | 0 | ✓ |
| Endpoints sem auth | 1 | 0 | ✓ |
| Credenciais em git | Sim | Não | ✓ |
| JWT validação real | Não | Sim | ✓ |
| CORS whitelist | Não | Sim | ✓ |
| RBAC implementado | Não | Sim | ✓ |

### Arquitetura (ADR-0003)

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| Linhas entregas.js | 366 | 90 | ✓ |
| Arquivos lógica | 1 | 3+ | ✓ |
| Testes | 0 | 75+ | ✓ |
| Cobertura | 0% | 85% | ✓ |
| Async padrões | 3 | 1 | ✓ |
| console.log | 5+ | 0 | ✓ |

### Consolidado

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| **Vulnerabilidades críticas** | 8 | 0 | ✓ |
| **Problemas arquiteturais** | 10 | 0 | ✓ |
| **Cobertura de testes** | 0% | 85% | ✓ |
| **Linhas em entregas.js** | 366 | 90 | ✓ |
| **CVSS Score total** | 62.2 | 0 | ✓ |

---

## 🚀 Próximos Passos

### Semana 1 (Aprovação)

1. **Revisar ADRs**
   - [ ] CTO review (3 dias)
   - [ ] Security review (2 dias)
   - [ ] Feedback + ajustes (1 dia)

2. **Aprovação Final**
   - [ ] Assinatura das 3 ADRs
   - [ ] Comunicado ao team

3. **Setup Inicial**
   - [ ] Criar feature branch
   - [ ] Criar estrutura
   - [ ] Allocar equipe

### Semana 2-3 (Implementação FASE 0-1)

1. **FASE 0 (24h)**
   - [ ] Rotacionar credenciais
   - [ ] Remover .env de git

2. **FASE 1 (3 dias)**
   - [ ] Auth real implementada
   - [ ] CORS restritivo
   - [ ] Rastreamento protegido

### Semana 3-4 (Implementação FASE 2)

1. **Refatoração arquitetura**
   - [ ] Services criados
   - [ ] Controllers criados
   - [ ] Repositories criados

2. **Testes + finalização**
   - [ ] 75+ testes
   - [ ] Code review
   - [ ] Merge para main

---

## 📞 Pontos de Contato

### Áreas de Responsabilidade

| Área | ADR | Lead | Tempo |
|------|-----|------|-------|
| **Segurança** | 0002 | Backend Engineer | 94h |
| **Arquitetura** | 0003 | Backend Engineer | 108h |
| **Coordenação** | 0004 | Tech Lead | 12 dias |
| **DevOps/Infra** | 0002 | DevOps | 24h |
| **Security Review** | 0002 | Security Officer | 8h |

---

## 📚 Documentação de Apoio

**Análises Detalhadas:**
- [`SECURITY_ANALYSIS_ENTREGAS.md`](../SECURITY_ANALYSIS_ENTREGAS.md) - Análise de 12 vulnerabilidades
- [`ENTREGAS_ARCHITECTURE_ANALYSIS.md`](../ENTREGAS_ARCHITECTURE_ANALYSIS.md) - Análise de arquitetura
- [`REMEDIATION_PLAN.md`](../REMEDIATION_PLAN.md) - Plano de remediação

**Referências do Projeto:**
- [`TECHNICAL_DEBT_MAP.md`](../TECHNICAL_DEBT_MAP.md) - Mapa geral de dívidas
- [`CLAUDE.md`](../CLAUDE.md) - Instruções do projeto
- [`docs/adr/ADR-0001-refatoracao-veiculoservice.md`](ADR-0001-refatoracao-veiculoservice.md) - ADR anterior

---

## 📝 Histórico

| Data | Evento | Status |
|------|--------|--------|
| 2026-06-18 | Criar ADR-0002 (Segurança) | ✅ Criado |
| 2026-06-18 | Criar ADR-0003 (Arquitetura) | ✅ Criado |
| 2026-06-18 | Criar ADR-0004 (Consolidado) | ✅ Criado |
| 2026-06-18 | Criar índice de ADRs | ✅ Este documento |
| 2026-06-XX | Aprovação das 3 ADRs | ⏳ Aguardando |
| 2026-06-XX | Implementação FASE 0 | ⏳ Pendente |
| 2026-06-XX | Implementação FASE 1-3 | ⏳ Pendente |
| 2026-06-XX | Merge para main | ⏳ Pendente |

---

## 🎯 Conclusão

Este conjunto de ADRs fornece um caminho claro e coordenado para:

1. **Eliminar 12 vulnerabilidades críticas de segurança** que expõem dados de clientes
2. **Refatorar arquitetura de um serviço inmanutenível** para MVC limpo e testável
3. **Executar ambas as refatorações em paralelo** em 12 dias com 2-3 pessoas

**Resultado Final:**
- 🔒 Serviço seguro (OWASP compliant)
- 📐 Arquitetura limpa (SOLID principles)
- 🧪 Altamente testável (85% cobertura)
- 📊 Observável (logging estruturado)
- 📚 Bem documentado

**Começar por:** Aprovar as 3 ADRs

---

**Índice Criado:** 2026-06-18  
**Versão:** 1.0  
**Autor:** Análise Técnica - Claude Code  
**Status:** PROPOSTO - Aguardando Aprovação
