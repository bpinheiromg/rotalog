# 📊 Mapa de Dívidas Técnicas - Rotalog Workspace
**Data de análise:** 2026-06-17  
**Status:** Workspace com múltiplas dívidas técnicas críticas identificadas

---

## 📋 Índice
1. [Sumário Executivo](#sumário-executivo)
2. [API Entregas (Node.js)](#1-rotalog-api-entregas-nodejs)
3. [API Frotas (Java)](#2-rotalog-api-frotas-java)
4. [API Notificações (.NET)](#3-rotalog-api-notificações-net)
5. [Frontend (Angular/React)](#4-rotalog-frontend-angularreact)
6. [Workspace (Infraestrutura)](#5-rotalog-workspace-infraestrutura)
7. [Matriz de Priorização](#-matriz-de-priorização)

---

## 🚨 Sumário Executivo

| Métrica | Status | Detalhes |
|---------|--------|----------|
| **Problemas Críticos** | 🔴 8+ | Credenciais hardcodeadas em 3 serviços |
| **Vulnerabilidades de Segurança** | 🔴 CRÍTICO | CORS aberto em todos os serviços |
| **God Classes** | 🟠 4+ | Falta separação de responsabilidades |
| **Cobertura de Testes** | 🔴 0% | 3 APIs sem testes |
| **Dependências Outdated** | 🟠 3 | Spring Boot 2.7 EOL, Prettier 2.6.2 |
| **Infraestrutura** | 🟠 MÉDIA | Sem health checks, logging ou métricas |

**Recomendação:** Resolver dívidas críticas de segurança em 1ª prioridade

---

## 1. ROTALOG-API-ENTREGAS (Node.js)

### 🔐 Segurança

#### ❌ CRÍTICO: Credenciais Hardcodeadas no `.env` Commitado
- **Localização:** `.env` (arquivo raiz)
- **Problema:** Arquivo`.env` está commitado no repositório
  ```
  DB_PASSWORD=rotalog123
  JWT_SECRET=super-secret-key-that-should-not-be-hardcoded
  NOTIFICACAO_SERVICE_URL=http://api-notificacoes:3003
  FROTAS_SERVICE_URL=http://api-frotas:8080
  ```
- **Impacto:** Qualquer pessoa com acesso ao repo tem credenciais do banco
- **Ação:** 
  - [ ] Remover `.env` do histórico git (git filter-branch ou BFG)
  - [ ] Rotacionar credenciais do banco
  - [ ] Adicionar `.env` ao `.gitignore`
  - [ ] Usar `.env.example` com placeholders
  - [ ] Implementar secrets manager

#### ❌ CRÍTICO: CORS Aberto Para Todos
- **Localização:** `src/index.js`
- **Problema:** 
  ```javascript
  app.use(cors()); // Permite requisições de qualquer origem
  ```
- **Impacto:** Qualquer website pode fazer requisições para a API
- **Ação:**
  - [ ] Configurar CORS restritivo com whitelist
  - [ ] Ler origins permitidas de variável de ambiente

#### ⚠️ ALTO: Rastreamento Endpoint Sem Autenticação
- **Localização:** `GET /rastreamento/:pedidoId`
- **Problema:** Expõe dados de entrega sem validação de acesso
- **Ação:**
  - [ ] Adicionar autenticação/autorização
  - [ ] Validar que usuário só pode ver suas próprias entregas

#### ⚠️ ALTO: Serviços Externos Sem Circuit Breaker
- **Localização:** `src/services/notificacaoService.js`, `frotasService.js`
- **Problema:** Chamadas HTTP diretas sem retry ou timeout
- **FIXME encontrado:** "Sem circuit breaker", "Fire-and-forget error handling"
- **Ação:**
  - [ ] Implementar circuit breaker (axios-retry ou similar)
  - [ ] Adicionar timeouts
  - [ ] Implementar retry com backoff exponencial

### 🏗️ Arquitetura

#### ❌ ALTO: Mistura de Padrões Async
- **Problema:** 60% callbacks + async/await no mesmo código
- **Localização:** Múltiplos serviços
- **Exemplo problemas:**
  - Callbacks de erro não tratados
  - Mistura de `.then()` com `await`
  - Fire-and-forget sem error handling
- **Ação:**
  - [ ] Migrar tudo para async/await
  - [ ] Implementar error handling consistente
  - [ ] Usar Promises ao invés de callbacks

#### ⚠️ ALTO: Sem Estratégia Centralizada de Erro
- **Problema:** Cada endpoint lida com erros manualmente
- **Ação:**
  - [ ] Implementar middleware de erro global
  - [ ] Criar classe de erro estruturada
  - [ ] Padronizar responses de erro

#### ⚠️ ALTO: Health Check Hardcodeado
- **Problema:** Endpoint `/health` retorna status fixo, não verifica dependências
- **Ação:**
  - [ ] Verificar conexão com PostgreSQL
  - [ ] Verificar conectividade com serviços externos
  - [ ] Retornar status detalhado

### 📦 Dependências

#### ℹ️ MÉDIO: Versões Moderadamente Desatualizadas
| Dependência | Versão | Status |
|-------------|--------|--------|
| express | 4.18.2 | ⚠️ 2 versões atrás |
| sequelize | 6.28.0 | ⚠️ Desatualizado |
| pg | 8.9.0 | ✅ Razoável |
| dotenv | 16.0.3 | ✅ Atual |

- **Ação:**
  - [ ] Executar `npm audit`
  - [ ] Atualizar Express para 4.21+
  - [ ] Avaliar Sequelize 6.35+

### 🔍 Inconsistência de Código

#### ⚠️ MÉDIO: Logging Manual com console.log
- **Problema:** Sem biblioteca de logging, logs não estruturados
- **Localização:** `src/index.js` e serviços
- **Ação:**
  - [ ] Implementar winston ou pino
  - [ ] Estruturar logs com níveis (debug, info, warn, error)
  - [ ] Adicionar correlationId para rastreamento

#### ⚠️ MÉDIO: URLs Hardcodeadas
- **Localização:** `src/services/notificacaoService.js`, `frotasService.js`
- **Problema:** URLs como `http://api-notificacoes:3003` hardcodeadas
- **FIXME encontrado**
- **Ação:**
  - [ ] Mover para variáveis de ambiente
  - [ ] Centralizar configurações

### ⚙️ Infraestrutura

#### ❌ ALTO: Sem Graceful Shutdown
- **Problema:** Servidor não aguarda requisições ativas ao desligar
- **Ação:**
  - [ ] Implementar listener para SIGTERM
  - [ ] Aguardar conexões ativas fecharem
  - [ ] Fechar conexão do banco

#### ⚠️ MÉDIO: Sem Métricas
- **Problema:** Sem visibilidade de performance
- **Ação:**
  - [ ] Implementar prometheus-client
  - [ ] Métrica de requisições por endpoint
  - [ ] Latência de banco de dados

#### ⚠️ MÉDIO: Sem Rate Limiting
- **Problema:** Sem proteção contra abuso
- **Ação:**
  - [ ] Implementar express-rate-limit
  - [ ] Configurar limite por IP/usuário

### ✅ Testes

#### ❌ CRÍTICO: Zero Cobertura de Testes
- **Problema:** `package.json` tem `"test": "echo \"Error: no test specified\""`
- **Ação:**
  - [ ] Implementar Jest
  - [ ] Adicionar testes unitários para serviços
  - [ ] Adicionar testes de integração

---

## 2. ROTALOG-API-FROTAS (Java)

### 🔐 Segurança

#### ❌ CRÍTICO: CORS Aberto Para Todos
- **Localização:** `src/main/java/com/rotalog/config/CorsConfig.java`
- **Problema:**
  ```java
  allowedOrigins("*") // Permite requisições de qualquer origem
  ```
- **FIXME encontrado:** "Permite tudo - deveria ser restritivo"
- **Ação:**
  - [ ] Configurar lista whitelist de origins
  - [ ] Ler origins de propriedade configurável por ambiente
  - [ ] Adicionar `@Value` ou `ConfigurationProperties`

#### ⚠️ ALTO: Sem Validação de Input
- **Localização:** `VeiculoController.java`
- **Problema:** Faltam `@Valid` nas anotações de entrada
- **FIXME encontrado:** "No @Valid annotations"
- **Ação:**
  - [ ] Adicionar `@Valid` em RequestBody
  - [ ] Implementar custom validators
  - [ ] Testar casos extremos

### 🏗️ Arquitetura

#### ❌ CRÍTICO: God Class - VeiculoService
- **Localização:** `src/main/java/com/rotalog/service/VeiculoService.java`
- **Problema:** 100+ linhas com múltiplas responsabilidades
  - CRUD operations
  - Validação complexa
  - Lógica de notificação
  - Cálculo de estatísticas
- **FIXME encontrado:** "Break this into smaller services"
- **Ação:**
  - [ ] Extrair validação para `VeiculoValidator`
  - [ ] Extrair notificação para `VeiculoNotificationService`
  - [ ] Criar `VeiculoStatisticsService`
  - [ ] Usar dependency injection adequado

#### ❌ CRÍTICO: God Class - VeiculoController
- **Localização:** `src/main/java/com/rotalog/controller/VeiculoController.java`
- **Problema:** Múltiplas responsabilidades
  - Roteamento
  - Validação
  - Formatação de resposta
  - Error handling manual
- **FIXME encontrado:** "No global exception handling"
- **Ação:**
  - [ ] Criar `@ControllerAdvice` para centralizar tratamento de erros
  - [ ] Delegar validação ao Spring/Bean Validation
  - [ ] Simplificar controller

#### ⚠️ ALTO: Sem @ControllerAdvice (Centralizado Error Handling)
- **Problema:** Cada controller retorna HashMap com erros
- **Exemplo:** `new HashMap<>("error", "message")`
- **Ação:**
  - [ ] Criar classe `GlobalExceptionHandler` com `@ControllerAdvice`
  - [ ] Implementar `ProblemDetail` (RFC 7807)
  - [ ] Retornar responses estruturadas

#### ⚠️ ALTO: Retornando Entities Diretamente
- **Problema:** DTOs não são usados, expõe estrutura do banco
- **Localização:** Controllers e serviços
- **Ação:**
  - [ ] Criar DTOs para request/response
  - [ ] Usar MapStruct para mapeamento
  - [ ] Separar modelo de banco do modelo de API

#### ⚠️ ALTO: Sem Paginação
- **Localização:** `VeiculoController.java`
- **FIXME encontrado:** "No pagination in list endpoints"
- **Problema:** GET `/veiculos` retorna tudo
- **Ação:**
  - [ ] Implementar `Pageable` do Spring Data
  - [ ] Adicionar sorting
  - [ ] Documentar no Swagger

#### ⚠️ MÉDIO: Sync HTTP Calls Para Inter-Service Communication
- **Problema:** RestTemplate bloqueante
- **Ação:**
  - [ ] Implementar WebClient (não-bloqueante)
  - [ ] Adicionar timeout
  - [ ] Implementar circuit breaker com Resilience4j

#### ⚠️ MÉDIO: Typo em Nome de Método
- **Localização:** `VeiculoController.java`
- **Problema:** `obterEstatisticasFreita()` (deveria ser `Frota`)
- **Ação:**
  - [ ] Renomear método
  - [ ] Renomear endpoint

#### ⚠️ MÉDIO: Health Check Manual
- **Problema:** Sem Spring Actuator, implementado manualmente
- **Ação:**
  - [ ] Adicionar `spring-boot-starter-actuator`
  - [ ] Implementar `HealthIndicator` customizado
  - [ ] Verificar dependências (DB, outros serviços)

### 📦 Dependências

#### ❌ CRÍTICO: Spring Boot 2.7.14 (EOL)
- **Versão:** Spring Boot 2.7.14
- **Status:** End-of-Life desde novembro 2023
- **Problema:** Sem suporte de segurança
- **Ação:**
  - [ ] Migrar para Spring Boot 3.3+ (LTS)
  - [ ] Verificar compatibilidade de dependências
  - [ ] Testar integração com Spring Cloud

#### ⚠️ ALTO: Java 11
- **Versão:** Java 11
- **Status:** LTS mas antigo (suporte até janeiro 2027)
- **Ação:**
  - [ ] Migrar para Java 21 LTS
  - [ ] Atualizar pom.xml
  - [ ] Testar compatibilidade

| Dependência | Versão | Status |
|-------------|--------|--------|
| spring-boot | 2.7.14 | 🔴 EOL |
| java | 11 | ⚠️ Antigo |
| postgresql | 42.5.1 | ✅ Atual |
| flyway | 8.5.1 | ✅ Atual |
| lombok | 1.18.34 | ⚠️ Desatualizado |

### 🔍 Inconsistência de Código

#### ⚠️ ALTO: Injeção de Dependência com @Autowired
- **Problema:** `@Autowired` em fields ao invés de constructor
- **Ação:**
  - [ ] Usar constructor injection
  - [ ] Benefício: Imutabilidade, melhor testabilidade

#### ⚠️ MÉDIO: Duplicação de Error Handling
- **Problema:** Cada controller tem try-catch manual
- **Ação:**
  - [ ] Usar `@ControllerAdvice` globalizado
  - [ ] Remover duplicação

#### ⚠️ MÉDIO: Sem DTOs para Respostas
- **Problema:** Retorna entidades JPA diretamente
- **Ação:**
  - [ ] Criar response DTOs
  - [ ] Usar MapStruct

### ✅ Testes

#### ❌ CRÍTICO: Zero Testes
- **Problema:** Nenhum teste encontrado
- **Ação:**
  - [ ] Implementar JUnit 5
  - [ ] Adicionar testes unitários com Mockito
  - [ ] Adicionar testes de integração com Testcontainers

---

## 3. ROTALOG-API-NOTIFICAÇÕES (.NET)

### 🔐 Segurança

#### ❌ CRÍTICO: Credenciais Hardcodeadas em appsettings.json
- **Localização:** `appsettings.json` (commitado)
- **Problema:**
  ```json
  {
    "EmailSettings": {
      "SenderPassword": "hardcoded-password-in-config"
    },
    "SmsSettings": {
      "ApiKey": "hardcoded-sms-api-key"
    },
    "ConnectionStrings": {
      "DefaultConnection": "...Password=rotalog123;..."
    }
  }
  ```
- **Impacto:** Expõe credenciais de email, SMS e banco
- **FIXME encontrado:** "TODO: Move credentials to secrets manager"
- **Ação:**
  - [ ] Mover para User Secrets (desenvolvimento) ou Azure Key Vault (produção)
  - [ ] Remover do histórico git
  - [ ] Rotacionar credenciais
  - [ ] Usar `IConfiguration` para ler credenciais seguras

#### ❌ CRÍTICO: Sem Health Checks
- **Localização:** `Program.cs`
- **Problema:** Aplicação continua rodando mesmo sem DB
- **FIXME encontrado:** "Missing health check packages"
- **Ação:**
  - [ ] Adicionar pacote `AspNetCore.HealthChecks.*`
  - [ ] Implementar health checks para DB e serviços externos
  - [ ] Falhar startup se dependências críticas indisponíveis

#### ⚠️ ALTO: Sem HTTPS Redirection em Development
- **Localização:** `Program.cs`
- **Problema:** Configuração comentada
- **Ação:**
  - [ ] Descomentar ou configurar corretamente
  - [ ] Gerar certificado auto-assinado local

### 🏗️ Arquitetura

#### ❌ CRÍTICO: God Class - NotificacaoService
- **Localização:** `Services/NotificacaoService.cs`
- **Problema:** 30+ FIXMEs documentando problemas
  ```
  - Mixed responsibilities: CRUD + sending + template + retry logic
  - No interface (can't mock for tests)
  - Fake email/SMS sending (simulates with Thread.Sleep)
  - No circuit breaker
  - No dead letter queue
  - Locking/concurrency issues
  - Hardcoded batch size
  - Inline validation
  - Sync sending (should be async queue)
  - No retry with backoff
  - Manual mapping instead of AutoMapper
  - Loads all data into memory
  - Random 10% failure simulation
  ```
- **Ação:**
  - [ ] Extrair Email Service
  - [ ] Extrair SMS Service
  - [ ] Extrair Template Engine
  - [ ] Extrair Retry Logic
  - [ ] Criar `INotificacaoService` interface
  - [ ] Implementar async queue (Hangfire ou Azure Service Bus)

#### ❌ CRÍTICO: Usando MediatR Mas Não Implementado
- **Problema:** MediatR 11.1.0 configurado mas não usado
- **FIXME encontrado:** "MediatR configured but not used"
- **Problema:** Clean Architecture abandonada
- **Ação:**
  - [ ] Implementar Command/Query handlers
  - [ ] Usar MediatR para desacoplar camadas
  - [ ] Implementar pipeline behaviors para logging/validação

#### ❌ CRÍTICO: NotificacaoService Sem Interface
- **Problema:** Classe concreta injetada em `Program.cs`
- **Impacto:** Impossível fazer mock em testes
- **Ação:**
  - [ ] Criar `INotificacaoService` interface
  - [ ] Registrar na DI como interface
  - [ ] Benefício: Testabilidade, mock fácil

#### ⚠️ ALTO: Simulação de Email/SMS com Thread.Sleep
- **Problema:** `Thread.Sleep(1000)` para simular envio real
- **FIXME encontrado**
- **Impacto:** Requisições lentas em produção
- **Ação:**
  - [ ] Usar provider real de email (SendGrid, AWS SES)
  - [ ] Usar provider real de SMS (Twilio, AWS SNS)
  - [ ] Implementar fila assíncrona

#### ⚠️ ALTO: Random 10% Failure Simulation
- **Problema:** Código deliberadamente simula falhas
- **FIXME encontrado:** "Random 10% failure simulation"
- **Ação:**
  - [ ] Remover código de teste
  - [ ] Implementar testes adequados

#### ⚠️ ALTO: Sync Notification Sending
- **Problema:** Notificações enviadas sincronamente
- **Ação:**
  - [ ] Implementar fila (Hangfire, RabbitMQ)
  - [ ] Retornar response imediato
  - [ ] Processar assincronamente

#### ⚠️ ALTO: Carrega Tudo na Memória
- **Problema:** `Notificacoes.ToList()` para operações em lote
- **FIXME encontrado:** "Loads all data into memory"
- **Ação:**
  - [ ] Usar batch processing com `Take()`/`Skip()`
  - [ ] Implementar streaming
  - [ ] Usar paginação

#### ⚠️ ALTO: Batch Size Hardcodeado
- **Problema:** `Take(50)` fixo
- **Ação:**
  - [ ] Mover para configuração
  - [ ] Tornar configurável por ambiente

#### ⚠️ MÉDIO: Template com String.Replace
- **Problema:** Sem template engine real
- **Ação:**
  - [ ] Implementar Liquid, Scriban ou Handlebars.Net
  - [ ] Separar templates em arquivos

#### ⚠️ MÉDIO: Manual Mapping ao invés de AutoMapper
- **Localização:** `NotificacaoService.cs`
- **Ação:**
  - [ ] Adicionar AutoMapper
  - [ ] Criar mapping profiles
  - [ ] Reduzir boilerplate

#### ⚠️ MÉDIO: Sem FluentValidation
- **Problema:** Validação inline
- **TODO encontrado**
- **Ação:**
  - [ ] Implementar FluentValidation
  - [ ] Criar validators separados
  - [ ] Usar pipeline behavior do MediatR

#### ⚠️ MÉDIO: Sem Polly Para Resilience
- **Problema:** Sem circuit breaker, retry ou timeout
- **TODO encontrado**
- **Ação:**
  - [ ] Adicionar Polly package
  - [ ] Implementar circuit breaker
  - [ ] Implementar retry com backoff exponencial

#### ⚠️ MÉDIO: CORS Aberto Para Tudo
- **Localização:** `Program.cs`
- **Problema:** `AllowAnyOrigin()`
- **Ação:**
  - [ ] Configurar whitelist de origins
  - [ ] Ler de configuration

#### ⚠️ MÉDIO: EnsureCreated ao invés de Migrations
- **Localização:** `Program.cs`
- **Problema:** `Database.EnsureCreated()` é para prototipagem
- **Ação:**
  - [ ] Usar EF Core Migrations
  - [ ] Executar migrations na startup
  - [ ] Manter schema.sql sincronizado

#### ⚠️ MÉDIO: Sem Retry com Backoff
- **Problema:** FIXME encontrada
- **Ação:**
  - [ ] Implementar exponential backoff
  - [ ] Usar Polly

#### ⚠️ MÉDIO: Sem Cache para Estatísticas
- **Problema:** Queries não otimizadas
- **Ação:**
  - [ ] Implementar caching (Redis ou in-memory)
  - [ ] Invalidar cache quando apropriado

#### ⚠️ MÉDIO: Sem Distributed Locking
- **Problema:** Concorrência em operações em lote
- **FIXME encontrado:** "Locking/concurrency issues"
- **Ação:**
  - [ ] Implementar distributed lock (Redis ou Postgres)
  - [ ] Usar para operações críticas

#### ⚠️ MÉDIO: Sem Dead Letter Queue
- **Problema:** Mensagens falhadas não são capturadas
- **FIXME encontrado**
- **Ação:**
  - [ ] Implementar DLQ em fila
  - [ ] Monitorar e alertar falhas

### 📦 Dependências

#### ⚠️ ALTO: MediatR Version Pinned
- **Versão:** 11.1.0
- **Status:** Pode estar desatualizado
- **FIXME encontrado:** "MediatR version pinned - should be updated"
- **Ação:**
  - [ ] Avaliar versão 12+
  - [ ] Testar compatibilidade

| Dependência | Versão | Status |
|-------------|--------|--------|
| MediatR | 11.1.0 | ⚠️ Pinned |
| EntityFrameworkCore | 6.0.12 | ⚠️ Desatualizado |
| Npgsql.EF | 6.0.8 | ⚠️ Desatualizado |
| Swashbuckle | 6.5.0 | ⚠️ Desatualizado |

- **Ação:**
  - [ ] Atualizar para .NET 8 LTS
  - [ ] Atualizar todos os packages

### 🔍 Inconsistência de Código

#### ⚠️ MÉDIO: Logging com Console.WriteLine
- **Problema:** Sem structured logging
- **Ação:**
  - [ ] Implementar Serilog
  - [ ] Configurar sinks (console, arquivo, elk)
  - [ ] FIXME encontrada mencionando isso

#### ⚠️ MÉDIO: Registering Concrete Class ao invés de Interface
- **Localização:** `Program.cs`
- **Problema:**
  ```csharp
  services.AddScoped<NotificacaoService>();
  // Deveria ser:
  services.AddScoped<INotificacaoService, NotificacaoService>();
  ```
- **Ação:**
  - [ ] Usar interfaces para todas as dependências
  - [ ] Benefício: Testabilidade, mockabilidade

### ✅ Testes

#### ❌ CRÍTICO: Zero Testes
- **Problema:** Nenhum teste encontrado
- **Ação:**
  - [ ] Implementar xUnit
  - [ ] Adicionar testes unitários com Moq
  - [ ] Adicionar testes de integração com Testcontainers

---

## 4. ROTALOG-FRONTEND (Angular/React)

### 🏗️ Arquitetura

#### ❌ CRÍTICO: God Objects em Componentes Angular
- **Localização:** `apps/painel-admin/src/app/components/`
- **Problema:** Componentes com 500+ linhas combinando:
  - List de dados
  - Formulário
  - Detalhes
  - Edição
- **FIXME encontrado em CLAUDE.md**
- **Ação:**
  - [ ] Quebrar em componentes menores
    - `ListComponent` (apenas list)
    - `DetailComponent` (apenas detalhe)
    - `FormComponent` (apenas formulário)
    - `ContainerComponent` (orquestra)
  - [ ] Usar smart/dumb component pattern
  - [ ] Implementar lazy loading

#### ❌ CRÍTICO: 70% Class Components em React
- **Localização:** `apps/rastreamento/`
- **Problema:** Usando class components ao invés de functional + hooks
- **Ação:**
  - [ ] Migrar para functional components
  - [ ] Usar hooks (useState, useEffect, useContext)
  - [ ] Removeu componentDidMount/componentWillUnmount

#### ⚠️ ALTO: CSS Global Scope
- **Problema:** Estilos globais causam cascata
- **Impacto:** Mudança em um componente afeta outros
- **Ação:**
  - [ ] Usar CSS Modules
  - [ ] Ou Styled-components/emotion
  - [ ] Ou SCSS com scoping

#### ⚠️ ALTO: Widespread `any` Type
- **Localização:** React app (rastreamento)
- **Problema:** 70% do código com `any` para bypass TypeScript strict
- **Ação:**
  - [ ] Habilitar strict mode
  - [ ] Criar interfaces/types
  - [ ] Gradualmente remover `any`

#### ⚠️ ALTO: Usando fetch() Diretamente
- **Localização:** Angular components
- **Problema:** Sem centralized HTTP error handling
- **Ação:**
  - [ ] Usar `HttpClient` do Angular
  - [ ] Implementar interceptor para erro global
  - [ ] Adicionar loading/error states

#### ⚠️ MÉDIO: Redux Sem Redux Toolkit
- **Problema:** Redux manual com actions/reducers boilerplate
- **Ação:**
  - [ ] Migrar para Redux Toolkit
  - [ ] Usar slices
  - [ ] Reduzir código boilerplate 70%

#### ⚠️ MÉDIO: Fake Zoom Buttons
- **Localização:** `apps/rastreamento/`
- **Problema:** Botões de zoom que não fazem nada
- **Ação:**
  - [ ] Remover ou implementar
  - [ ] Se não usado, deletar

#### ⚠️ MÉDIO: Sem Lazy Loading
- **Problema:** Todos os módulos carregados no início
- **Ação:**
  - [ ] Implementar lazy loading de routes
  - [ ] Usar `loadChildren` no routing
  - [ ] Benefício: Startup time reduzido

#### ⚠️ MÉDIO: Mixed Patterns (Angular + React)
- **Problema:** Monorepo com ambos frameworks
- **Ação:**
  - [ ] Usar shared lib apenas (NX libs)
  - [ ] Migrar gradualmente para um único framework
  - [ ] Ou documentar clara separação

### 📦 Dependências

#### ⚠️ MÉDIO: Prettier 2.6.2 Desatualizado
| Dependência | Versão | Status |
|-------------|--------|--------|
| Angular | 18.2.0 | ✅ Atual |
| React | 18.3.1 | ✅ Atual |
| TypeScript | 5.5.0 | ✅ Atual |
| NX | 19.8.4 | ✅ Atual |
| Prettier | 2.6.2 | ⚠️ Desatualizado |

- **Ação:**
  - [ ] `npm update prettier` para 3.x
  - [ ] Revisar `.prettierrc` para mudanças

### 🔍 Inconsistência de Código

#### ⚠️ ALTO: Estrutura de Libs Não Utilizada
- **Problema:** Monorepo tem libs mas componentes duplicados
- **Localização:** `libs/` vs `apps/*/src/app/`
- **Ação:**
  - [ ] Mover componentes compartilhados para libs
  - [ ] Mover services para libs
  - [ ] Mover models para libs

#### ⚠️ MÉDIO: Sem Linting Uniforme
- **Problema:** Diferentes apps podem ter diferentes regras
- **Ação:**
  - [ ] Configurar ESLint root level
  - [ ] Enforce em todos os apps
  - [ ] Pre-commit hooks

### ✅ Testes

#### ⚠️ MÉDIO: Testes Presentes Mas Incompletos
- **Arquivo encontrado:** `app.component.spec.ts`
- **Problema:** Cobertura baixa dada complexidade
- **Ação:**
  - [ ] Aumentar cobertura para 80%+
  - [ ] Testes E2E com Cypress/Playwright
  - [ ] Tests coverage reporting

---

## 5. ROTALOG-WORKSPACE (Infraestrutura)

### 🔐 Segurança

#### ❌ CRÍTICO: Credenciais Hardcodeadas em docker-compose.yml
- **Localização:** `docker-compose.yml`
- **Problema:**
  ```yaml
  services:
    postgres:
      environment:
        POSTGRES_PASSWORD: rotalog123
        POSTGRES_USER: rotalog_admin
  ```
- **Impacto:** Qualquer dev que clone o repo tem as credenciais
- **Ação:**
  - [ ] Usar `.env` file (Docker Compose suporta)
  - [ ] Criar `.env.example` com placeholders
  - [ ] Gitignore `.env`
  - [ ] Usar secrets no Docker Swarm/Kubernetes

### 🏗️ Arquitetura

#### ❌ CRÍTICO: Múltiplos Serviços Comentados (Nunca Implementados)
- **Localização:** `docker-compose.yml`
- **Serviços comentados:**
  - Redis (cache)
  - Kafka (message queue)
  - Elasticsearch (logging)
  - Prometheus (metrics)
  - Grafana (visualization)
- **Problema:** Código morto, TODOs nunca implementados
- **Ação:**
  - [ ] Remover comentários ou implementar
  - [ ] Se não vai usar, documentar decisão arquitetural

#### ⚠️ CRÍTICO: Sem Health Checks em Containers
- **Localização:** `docker-compose.yml`
- **FIXME encontrado**
- **Problema:** Sem verificação de readiness
- **Ação:**
  - [ ] Adicionar `healthcheck` para cada serviço
  - [ ] API: GET /health
  - [ ] Postgres: `pg_isready`
  - [ ] Configurar `depends_on` com condition

#### ⚠️ ALTO: Sem Resource Limits
- **FIXME encontrado**
- **Problema:** Containers podem consumir todos os recursos
- **Ação:**
  - [ ] Adicionar `resources.limits` e `requests`
  - [ ] CPU e memory limits

#### ⚠️ ALTO: Sem Estratégia de Backup
- **FIXME encontrado:** "Backup strategy missing"
- **Problema:** Dados PostgreSQL sem backup
- **Ação:**
  - [ ] Implementar volume backup
  - [ ] Usar serviço de backup externo
  - [ ] Testar restore

#### ⚠️ MÉDIO: Sem Logging Centralizado
- **Problema:** Sem ELK, Datadog, ou similar
- **Ação:**
  - [ ] Implementar logging centralizado
  - [ ] Ou pelo menos usar `logging` driver no Docker

#### ⚠️ MÉDIO: Database Initialization
- **Localização:** SQL scripts em `tools/scripts/`
- **Problema:** Sem clear init strategy
- **Ação:**
  - [ ] Documentar processo de inicialização
  - [ ] Usar volume bind mount para scripts
  - [ ] Ou usar init container

#### ⚠️ MÉDIO: Sem Documentação de Setup Local
- **Problema:** Como rodar localmente não está claro
- **Ação:**
  - [ ] Criar SETUP.md
  - [ ] Incluir docker-compose up
  - [ ] Listar todos os serviços e portas

### 📊 Observabilidade

#### ❌ CRÍTICO: Sem Métricas
- **Problema:** Nenhum Prometheus/Datadog
- **Ação:**
  - [ ] Implementar Prometheus
  - [ ] Coletar métricas de aplicação
  - [ ] Visualizar em Grafana

#### ❌ CRÍTICO: Sem Logging Centralizado
- **Problema:** Logs apenas em stdout/files
- **Ação:**
  - [ ] Implementar ELK Stack ou Loki
  - [ ] Correlate logs across services

#### ⚠️ ALTO: Sem Distributed Tracing
- **Problema:** Difícil rastrear requisições entre serviços
- **Ação:**
  - [ ] Implementar Jaeger ou Zipkin
  - [ ] Adicionar instrumentation

### 🔍 Documentação

#### ⚠️ ALTO: Sem Architecture Decision Records (ADRs)
- **Problema:** Decisões arquiteturais não documentadas
- **Ação:**
  - [ ] Criar `docs/adr/` directory
  - [ ] Documentar cada decisão arquitetural

---

## 🎯 Matriz de Priorização

### Quadrante 1: CRÍTICO - Fazer Imediatamente
| Item | Projeto | Risco | Esforço | Ação |
|------|---------|-------|--------|------|
| Credenciais hardcodeadas | Entregas, Notificações, Workspace | 🔴 Máximo | 🟠 Médio | Remover, rotacionar, usar secrets |
| CORS aberto | Entregas, Frotas, Notificações | 🔴 Alto | 🟢 Baixo | Whitelist origins |
| God Classes | Frotas (VeiculoService), Notificações (NotificacaoService), Frontend (Components) | 🟠 Alto | 🔴 Alto | Refatorar em serviços menores |
| Zero testes | Entregas, Frotas, Notificações | 🟠 Alto | 🔴 Alto | Implementar Jest/JUnit/xUnit |

### Quadrante 2: IMPORTANTE - Próximas Sprints
| Item | Projeto | Risco | Esforço |
|------|---------|-------|--------|
| Spring Boot 2.7 EOL | Frotas | 🟠 Médio | 🔴 Alto |
| Sem validação de input | Frotas | 🟠 Médio | 🟢 Baixo |
| Sem graceful shutdown | Entregas | 🟠 Médio | 🟢 Baixo |
| Logging estruturado | Todos | 🟠 Médio | 🟢 Baixo |
| Health checks | Todos | 🟠 Médio | 🟢 Baixo |

### Quadrante 3: IMPORTANTE - Após Estabilização
| Item | Projeto | Risco | Esforço |
|------|---------|-------|--------|
| Remover class components | Frontend | 🟡 Baixo | 🟠 Médio |
| Lazy loading | Frontend | 🟡 Baixo | 🟢 Baixo |
| Distribuído tracing | Workspace | 🟡 Baixo | 🟠 Médio |
| Métricas e monitoring | Workspace | 🟡 Baixo | 🟠 Médio |

### Quadrante 4: NICE-TO-HAVE
| Item | Projeto |
|------|---------|
| Redux Toolkit migration | Frontend |
| CSS Modules | Frontend |
| Performance optimization | Frontend |

---

## 📝 Plano de Ação Recomendado

### Fase 1: Segurança (Semana 1-2)
**Objetivo:** Remover exposições críticas

- [ ] Remover e rotacionar credenciais de todos os repositórios
- [ ] Implementar `.env` gitignore em todos os serviços
- [ ] Configurar CORS restritivo
- [ ] Implementar autenticação no endpoint de rastreamento
- [ ] Migrar segredos para gerenciador seguro

**Esforço:** 40-60 horas  
**Risco Mitigado:** 🔴 CRÍTICO

---

### Fase 2: Qualidade de Código (Semana 3-6)
**Objetivo:** Estrutura de testes e refatoração base

- [ ] Implementar framework de testes em cada projeto
- [ ] Quebrar god classes em serviços menores
- [ ] Implementar validação centralizada
- [ ] Adicionar error handling global

**Esforço:** 80-120 horas  
**Prioridade:** 🟠 ALTA

---

### Fase 3: Dependências (Semana 4-5)
**Objetivo:** Atualizar stack técnico

- [ ] Migrar Spring Boot 2.7 → 3.3+
- [ ] Atualizar Java 11 → 21
- [ ] Atualizar .NET 6 → 8
- [ ] Revisar npm packages em Node/Frontend

**Esforço:** 60-100 horas  
**Paralelo com Fase 2:** Possível

---

### Fase 4: Observabilidade (Semana 6-8)
**Objetivo:** Visibilidade operacional

- [ ] Implementar logging estruturado (Serilog, Winston, Pino)
- [ ] Adicionar métricas (Prometheus)
- [ ] Implementar health checks
- [ ] Adicionar distributed tracing (Jaeger)

**Esforço:** 40-60 horas

---

### Fase 5: Arquitetura (Semana 9-12)
**Objetivo:** Escalabilidade e manutenibilidade

- [ ] Implementar async queues (Hangfire, RabbitMQ)
- [ ] Adicionar circuit breakers (Polly, Resilience4j)
- [ ] Implementar cache distribuído (Redis)
- [ ] Refatorar frontend components

**Esforço:** 100-150 horas

---

## 📊 Sumário de Estimativas

| Fase | Semanas | Esforço (horas) | Equipe Recomendada |
|------|---------|-----------------|-------------------|
| Segurança | 1-2 | 50 | 1-2 pessoas |
| Qualidade | 3-6 | 100 | 2-3 pessoas |
| Dependências | 4-5 | 80 | 1 pessoa |
| Observabilidade | 6-8 | 50 | 1 pessoa |
| Arquitetura | 9-12 | 125 | 2 pessoas |
| **Total** | **~12 semanas** | **~405 horas** | **2-3 pessoas** |

**Timeline realista:** 3 meses com equipe dedicada de 2-3 pessoas

---

## 🔗 Referências Externas

### OWASP Top 10 2024
- [A01: Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
- [A02: Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/)
- [A04: Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/)

### Boas Práticas
- [12 Factor App](https://12factor.net/)
- [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Microservices Patterns](https://microservices.io/patterns/index.html)

### Spring Boot
- [Migration Guide Spring Boot 2.7 → 3.0](https://spring.io/blog/2022/05/19/spring-boot-3-0-goes-ga)

### .NET
- [Migration Guide .NET 6 → .NET 8](https://learn.microsoft.com/en-us/dotnet/core/migration/upgrade-assistant-overview)

---

## ✅ Próximos Passos

1. **Comunicar com stakeholders** sobre timeline de 12 semanas
2. **Alocar equipe** dedicada (2-3 pessoas)
3. **Criar sprints** seguindo as fases recomendadas
4. **Implementar CI/CD** com checks (testes, linting, segurança)
5. **Revisar este documento** mensalmente e atualizar status

---

**Documento gerado:** 2026-06-17  
**Próxima revisão recomendada:** 2026-07-01
