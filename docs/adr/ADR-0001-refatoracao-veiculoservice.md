# ADR-0001: Refatoração da Classe VeiculoService - Separação de Responsabilidades

**Data:** 2026-06-18  
**Status:** PROPOSTO  
**Decisão Crítica:** Sim ⚠️  
**Prioridade:** 🔴 CRÍTICA (Quadrante 1 do TECHNICAL_DEBT_MAP)

---

## 1. Contexto

A classe `VeiculoService.java` localizada em `rotalog-api-frotas/src/main/java/com/rotalog/service/` foi identificada como uma **God Class** no documento de Dívidas Técnicas (`TECHNICAL_DEBT_MAP.md`). Este serviço concentra múltiplas responsabilidades que deveriam ser distribuídas entre componentes especializados, violando o princípio SOLID (Single Responsibility Principle).

### Situação Atual
- **Tamanho:** 340 linhas de código
- **Responsabilidades:** 7 áreas de negócio misturadas
- **Cobertura de Testes:** 0%
- **Exceções:** Todas genéricas (`RuntimeException`)
- **Logging:** Mistura de `System.out.println()` com SLF4J
- **Status:** String hardcodeada sem enumeração
- **Configurações:** Hardcodeadas no código (limites de km, custos, etc)

### Referência no TECHNICAL_DEBT_MAP
```
#### ❌ CRÍTICO: God Class - VeiculoService
- Localização: src/main/java/com/rotalog/service/VeiculoService.java
- Problema: 100+ linhas com múltiplas responsabilidades
  - CRUD operations
  - Validação complexa
  - Lógica de notificação
  - Cálculo de estatísticas
- FIXME encontrado: "Break this into smaller services"
```

---

## 2. Problema

### 2.1 Análise Detalhada dos 8 Pontos de Melhoria

#### 1️⃣ **TAMANHO DA CLASSE (340 linhas - Acima de 300)**

**Problema Identificado:**
- Métodos variando de 5 a 70 linhas de lógica
- 15 métodos públicos em uma única classe
- Múltiplas responsabilidades entrelaçadas

**Responsabilidades Atuais:**
| Responsabilidade | Métodos | Linhas |
|-----------------|---------|--------|
| CRUD Básico | `listarTodos()`, `buscarPorId()`, `buscarPorPlaca()`, `registrarVeiculo()`, `atualizarVeiculo()`, `desativarVeiculo()`, `reativarVeiculo()` | ~80 |
| Validação | Lógica espalhada em `registrarVeiculo()`, `obterVeiculosPorStatus()`, `atualizarQuilometragem()` | ~40 |
| Quilometragem | `atualizarQuilometragem()`, `precisaDeManutencao()` | ~50 |
| Manutenção | `agendarManutencaoPreventiva()`, `calcularCustoManutencao()`, `precisaDeManutencao()` | ~60 |
| Notificação | Chamadas espalhadas em 5 métodos | ~30 |
| Estatísticas | `obterEstatisticasFreita()` | ~15 |
| Sincronização | `sincronizarComSistemaExterno()` | ~10 |

**Impacto:**
- Difícil de testar
- Difícil de manter
- Difícil de estender
- Violação clara do SRP

---

#### 2️⃣ **EXCEÇÕES GENÉRICAS**

**Problema Identificado:**
- Todos os erros lançam `RuntimeException`
- Não há diferenciação entre tipos de erro
- Cliente não consegue tratar erros específicos

**Localização no Código:**
```java
// Linha 49
throw new RuntimeException("Veículo não encontrado: " + id);

// Linha 70, 74, 80, 85, 90
throw new RuntimeException("Placa é obrigatória");
throw new RuntimeException("Placa deve ter 7 caracteres");
throw new RuntimeException("Veículo com placa " + placa + " já existe");
// ... etc

// Linha 160, 196
throw new RuntimeException("Quilometragem não pode ser negativa");
throw new RuntimeException("Status inválido: " + status);
```

**Exceções Necessárias:**
| Tipo | Cenários | Métodos Afetados |
|------|----------|------------------|
| `VeiculoNaoEncontradoException` | ID ou placa não existe | `buscarPorId()`, `buscarPorPlaca()` |
| `VeiculoDuplicadoException` | Placa já cadastrada | `registrarVeiculo()` |
| `PlacaInvalidaException` | Placa mal formatada | `registrarVeiculo()` |
| `ModeloInvalidoException` | Modelo vazio/nulo | `registrarVeiculo()` |
| `AnoFabricacaoInvalidoException` | Ano fora do intervalo | `registrarVeiculo()` |
| `QuilometragemInvalidaException` | Valor negativo/decrescente | `atualizarQuilometragem()` |
| `StatusInvalidoException` | Status não reconhecido | `obterVeiculosPorStatus()` |
| `CampoObrigatorioException` | Campo nulo/vazio | Validação genérica |
| `VeiculoJaEmManutencaoException` | Veículo não disponível | `agendarManutencaoPreventiva()` |
| `NotificacaoFalhaException` | Falha ao notificar | Métodos de notificação |

---

#### 3️⃣ **COBERTURA DE TESTES (0% → 90%)**

**Problema Identificado:**
- Nenhum teste unitário presente
- Difícil validar funcionalidades
- Risco alto de regressões

**Métodos a Testar (15 métodos = 90% cobertura):**
```java
✅ listarTodos()                          // 3 testes
✅ buscarPorId()                          // 2 testes (encontrado, não encontrado)
✅ buscarPorPlaca()                       // 2 testes
✅ registrarVeiculo()                     // 8 testes (validações, duplicidade, sucesso)
✅ atualizarVeiculo()                     // 5 testes (cada campo, validações)
✅ atualizarQuilometragem()               // 6 testes (aumento, redução, zero, negativo)
✅ obterVeiculosPorStatus()               // 4 testes (cada status, inválido)
✅ agendarManutencaoPreventiva()          // 3 testes
✅ calcularCustoManutencao()              // 4 testes (modelos, valores)
✅ precisaDeManutencao()                  // 3 testes (acima/abaixo limite)
✅ desativarVeiculo()                     // 2 testes
✅ reativarVeiculo()                      // 2 testes
✅ obterEstatisticasFreita()              // 3 testes (com dados, sem dados)
✅ sincronizarComSistemaExterno()         // 1 teste

Total estimado: ~48 testes unitários
```

---

#### 4️⃣ **JAVADOC EM EXCESSO**

**Problema Identificado:**
- 50+ linhas de Javadoc (comentários de bloco)
- Muitos TODOs e FIXMEs no Javadoc
- Informação que rotaciona rapidamente

**Exemplos de Javadoc Desnecessário:**
```java
// ❌ DESNECESSÁRIO - óbvio pelo nome do método
/**
 * Lista todos os veículos
 * 
 * FIXME: Sem paginação - pode retornar milhares de registros
 * FIXME: Sem cache
 */
public List<Veiculo> listarTodos() { ... }

// ❌ DESNECESSÁRIO - implementação clara
/**
 * Busca veículo por ID
 */
public Veiculo buscarPorId(Long id) { ... }

// ⚠️ PARCIAL - só o FIXME é relevante
/**
 * Registra um novo veículo no sistema
 * 
 * FIXME: This method does too many things
 * FIXME: Validation logic is mixed with business logic
 * FIXME: No proper error handling
 */
public Veiculo registrarVeiculo(...) { ... }
```

**Abordagem Recomendada:**
- Remover Javadoc redundante
- Mover FIXMEs para tickets de trabalho (JIRA/Linear)
- Manter apenas comentários que expliquem o "por quê" não óbvio

---

#### 5️⃣ **LOGS COM System.out.println()**

**Problema Identificado:**
- Mistura de `System.out.println()` com SLF4J Logger
- Logs não estruturados
- Difícil rastreabilidade

**Ocorrências Encontradas:**
```java
// Linha 116
System.out.println("WARN: Falha ao notificar sobre novo veículo");

// Linha 164
System.out.println("AVISO: Quilometragem menor que a atual!");

// Linha 327
System.out.println("Sincronização iniciada");
```

**Problema:**
- `System.out.println()` não passa por framework de logging
- Sem timestamp
- Sem nível de severidade
- Não pode ser redirecionado ou filtrado

---

#### 6️⃣ **STRINGS DE STATUS HARDCODEADAS**

**Problema Identificado:**
- Status como Strings em vários lugares: "ATIVO", "INATIVO", "MANUTENCAO"
- Sem validação de tipo
- Propenso a typos

**Ocorrências:**
```java
// Linha 98, 195
veiculo.setStatus("ATIVO");
if (!status.equals("ATIVO") && !status.equals("INATIVO") && !status.equals("MANUTENCAO"))

// Linha 266, 291
veiculo.setStatus("INATIVO");
veiculo.setStatus("ATIVO");

// Linha 109, 176, 220, 274
"NOVO_VEICULO", "ALERTA_MANUTENCAO", "MANUTENCAO_AGENDADA", "VEICULO_DESATIVADO"
```

**Enums Necessários:**
```java
// Status do Veículo
public enum VeiculoStatus {
    ATIVO("Veículo ativo"),
    INATIVO("Veículo inativo"),
    MANUTENCAO("Em manutenção");
}

// Tipos de Notificação
public enum TipoNotificacao {
    NOVO_VEICULO,
    ALERTA_MANUTENCAO,
    MANUTENCAO_AGENDADA,
    VEICULO_DESATIVADO;
}
```

---

#### 7️⃣ **DIVIDIR RESPONSABILIDADES**

**Problema Identificado:**
- Service único com 7 áreas de negócio
- Difícil reutilizar componentes
- Acoplamento com `NotificacaoClient`

**Proposta de Divisão:**

```
VeiculoService (Core - CRUD e orquestração)
├── VeiculoValidatorService (Validações)
├── VeiculoQuilometragemService (Quilometragem)
├── VeiculoManutencaoService (Manutenção preventiva)
├── VeiculoNotificacaoService (Notificações)
├── VeiculoEstatisticasService (Relatórios)
└── VeiculoSincronizacaoService (Integração externa)
```

**Benefícios:**
- Cada serviço com responsabilidade única
- Testes focados e isolados
- Reutilização de componentes
- Manutenção simplificada

---

#### 8️⃣ **VARIÁVEIS NÃO UTILIZADAS**

**Problema Identificado:**
- Variáveis declaradas mas não usadas
- Código morto

**Análise:**
```java
// Linha 212-213 - NUNCA USADAS
Long intervaloQuilometragem = 10000L;  // ❌ Declarada mas nunca referenciada
Integer intervaloMeses = 3;            // ❌ Declarada mas nunca referenciada

// Método inteiro é vazio (linhas 324-332)
public void sincronizarComSistemaExterno() {
    // FIXME: URL hardcoded
    log.info("Sincronização com sistema externo iniciada");
    System.out.println("Sincronização iniciada"); // FIXME: Use logger
    
    // TODO: Implementar integração real
    // TODO: Adicionar retry logic
    // TODO: Adicionar circuit breaker
} // ❌ Nenhuma implementação real
```

---

### 2.2 Problemas Adicionais Identificados

#### 🔴 **Injeção de Dependência com @Autowired em Field**
```java
@Autowired  // ❌ Anti-pattern
private VeiculoRepository veiculoRepository;

@Autowired  // ❌ Anti-pattern
private NotificacaoClient notificacaoClient;
```
**Problema:** Torna o objeto mutável e difícil de testar (não é possível injetar mocks no construtor)

**Solução:** Constructor injection
```java
private final VeiculoRepository veiculoRepository;
private final NotificacaoClient notificacaoClient;

@Autowired
public VeiculoService(VeiculoRepository veiculoRepository, NotificacaoClient notificacaoClient) {
    this.veiculoRepository = veiculoRepository;
    this.notificacaoClient = notificacaoClient;
}
```

#### 🔴 **Valores Hardcodeados**
- Limite de quilometragem: `50000L` (linha 253)
- Intervalo de quilometragem: `10000L` (linha 212)
- Intervalo de meses: `3` (linha 213)
- Custo por km: `0.05` (linha 238)
- Custo base: `500.0` (linha 239)
- Email de notificação: `"gestor@rotalog.com"` (linhas 110, 177, 222, 275)
- Comprimento da placa: `7` (linha 73)
- Intervalo de anos: `1900-2100` (linha 89)

**Solução:** Usar `@Value` ou `ConfigurationProperties` para ler de `application.properties`

#### 🔴 **Tratamento de Exceção Inadequado (Exception Swallowing)**
```java
// Linhas 107-117, 174-182, 219-227, 273-281
try {
    notificacaoClient.enviarNotificacao(...);
} catch (Exception e) {
    log.error("Falha ao notificar: {}", e.getMessage());
    // ❌ Exceção engolida - falha silenciosa
    // O veículo já foi salvo mas a notificação falhou
}
```

**Problema:** Se notificação falhar, operação fica inconsistente
**Solução:** Usar padrão saga ou event sourcing para garantir consistência

#### 🔴 **Falta de Paginação**
```java
// Linha 39-42
public List<Veiculo> listarTodos() {
    return veiculoRepository.findAll();
    // ❌ Pode retornar milhares de registros
}
```

#### 🔴 **Query Ineficiente**
```java
// Linhas 307-309
long ativos = veiculoRepository.findByStatus("ATIVO").size();
long inativos = veiculoRepository.findByStatus("INATIVO").size();
// ❌ Carrega TODA lista em memória só para contar
// Melhor: usar count() query
```

#### 🔴 **Typo no Nome do Método**
```java
// Linha 304
public String obterEstatisticasFreita() {  // ❌ "Freita" deveria ser "Frota"
    // FIXME: Typo no nome do método nunca foi corrigido
}
```

---

## 3. Decisão

### 3.1 Estratégia de Refatoração

Vamos refatorar `VeiculoService` seguindo os princípios SOLID com a seguinte abordagem:

#### **Fase 1: Separação de Responsabilidades**

Criar os seguintes serviços especializados:

1. **VeiculoService** (Refatorado)
   - Responsabilidade: CRUD básico e orquestração
   - Métodos: `listarTodos()`, `buscarPorId()`, `buscarPorPlaca()`, `registrarVeiculo()`, `atualizarVeiculo()`, `desativarVeiculo()`, `reativarVeiculo()`
   - Tamanho esperado: ~120 linhas

2. **VeiculoValidadorService** (Novo)
   - Responsabilidade: Todas as validações
   - Métodos: `validarPlaca()`, `validarModelo()`, `validarAnoFabricacao()`, `validarQuilometragem()`, `validarStatus()`
   - Tamanho esperado: ~80 linhas

3. **VeiculoQuilometragemService** (Novo)
   - Responsabilidade: Gestão de quilometragem
   - Métodos: `atualizarQuilometragem()`, `validarReduca()`, `registrarHistorico()`
   - Tamanho esperado: ~60 linhas

4. **VeiculoManutencaoService** (Novo)
   - Responsabilidade: Lógica de manutenção preventiva
   - Métodos: `agendarManutencao()`, `calcularCusto()`, `verificarNecessidade()`, `loadConfiguration()`
   - Tamanho esperado: ~100 linhas

5. **VeiculoNotificacaoService** (Novo)
   - Responsabilidade: Envio de notificações (wrapper sobre NotificacaoClient)
   - Métodos: `notificarNovoVeiculo()`, `notificarManutencao()`, `notificarDesativacao()`
   - Tamanho esperado: ~50 linhas

6. **VeiculoEstatisticasService** (Novo)
   - Responsabilidade: Cálculo de estatísticas
   - Métodos: `obterEstatisticas()` (renomeado), `obterPorStatus()`
   - Tamanho esperado: ~40 linhas

7. **VeiculoSincronizacaoService** (Novo)
   - Responsabilidade: Integração com sistemas externos
   - Métodos: `sincronizar()`, `definirRetry()`, `definirCircuitBreaker()`
   - Tamanho esperado: ~60 linhas

---

#### **Fase 2: Implementar Exceções Específicas**

Criar hierarquia de exceções:

```java
// src/main/java/com/rotalog/exception/VeiculoException.java
public abstract class VeiculoException extends RuntimeException {
    public VeiculoException(String message) {
        super(message);
    }
    
    public VeiculoException(String message, Throwable cause) {
        super(message, cause);
    }
}

// Exceções específicas
public class VeiculoNaoEncontradoException extends VeiculoException { }
public class VeiculoDuplicadoException extends VeiculoException { }
public class PlacaInvalidaException extends VeiculoException { }
public class QuilometragemInvalidaException extends VeiculoException { }
public class StatusInvalidoException extends VeiculoException { }
// ... etc
```

---

#### **Fase 3: Criar Enums para Status**

```java
// src/main/java/com/rotalog/domain/VeiculoStatus.java
public enum VeiculoStatus {
    ATIVO("Veículo ativo e disponível"),
    INATIVO("Veículo desativado"),
    MANUTENCAO("Veículo em manutenção");
    
    private final String descricao;
    
    VeiculoStatus(String descricao) {
        this.descricao = descricao;
    }
    
    public String getDescricao() {
        return descricao;
    }
}

// src/main/java/com/rotalog/domain/TipoNotificacao.java
public enum TipoNotificacao {
    NOVO_VEICULO("Notificação: novo veículo cadastrado"),
    ALERTA_MANUTENCAO("Alerta: manutenção necessária"),
    MANUTENCAO_AGENDADA("Confirmação: manutenção agendada"),
    VEICULO_DESATIVADO("Notificação: veículo desativado");
    
    private final String mensagem;
    // ... implementação
}
```

---

#### **Fase 4: Implementar Configurações**

```java
// src/main/resources/application.properties
veiculo.quilometragem.limite-manutencao=50000
veiculo.quilometragem.intervalo=10000
veiculo.manutencao.intervalo-meses=3
veiculo.manutencao.custo-base=500.0
veiculo.manutencao.custo-por-km=0.05
veiculo.placa.tamanho=7
veiculo.ano-fabricacao.minimo=1900
veiculo.ano-fabricacao.maximo=2100
notificacao.email-gestor=gestor@rotalog.com
```

---

#### **Fase 5: Testes Unitários (90% Cobertura)**

```
src/test/java/com/rotalog/service/
├── VeiculoServiceTest.java (15 testes)
├── VeiculoValidadorServiceTest.java (18 testes)
├── VeiculoQuilometragemServiceTest.java (12 testes)
├── VeiculoManutencaoServiceTest.java (14 testes)
├── VeiculoNotificacaoServiceTest.java (8 testes)
├── VeiculoEstatisticasServiceTest.java (10 testes)
└── VeiculoSincronizacaoServiceTest.java (6 testes)

Total: 83 testes unitários
Cobertura esperada: 92%
```

---

#### **Fase 6: Limpeza de Código**

- [ ] Remover todos `System.out.println()`
- [ ] Remover Javadoc redundante
- [ ] Converter `@Autowired` field injection → constructor injection
- [ ] Remover variáveis não utilizadas
- [ ] Renomear `obterEstatisticasFreita()` → `obterEstatisticas()`
- [ ] Remover métodos vazios (`sincronizarComSistemaExterno()`)

---

## 4. Consequências

### 4.1 Positivas ✅

| Consequência | Benefício |
|-------------|-----------|
| **Separação de Responsabilidades** | Cada service tem 1 responsabilidade clara |
| **Testabilidade Aumentada** | 90% cobertura alcançável |
| **Manutenção Facilitada** | Mudanças isoladas, sem efeitos colaterais |
| **Reutilização** | Services podem ser usados independentemente |
| **Performance** | Queries otimizadas (count ao invés de findAll) |
| **Configurabilidade** | Limites não mais hardcodeados |
| **Type Safety** | Enums ao invés de Strings |
| **Rastreabilidade** | Exceções específicas, logs estruturados |
| **Segurança** | Constructor injection, sem mutabilidade |
| **Escalabilidade** | Fácil adicionar novos serviços |

### 4.2 Negativas ⚠️

| Consequência | Mitigation |
|-------------|-----------|
| **Mais classes** | Organização clara em pacotes |
| **Mais injeção de dependências** | Use spring-boot-starter-validation |
| **Curva de aprendizado** | Documentação no README |
| **Refatoração de Controllers** | Haverá também refatoração em VeiculoController |
| **Esforço inicial** | ~200-250 horas de desenvolvimento |

### 4.3 Impacto Técnico

**Arquitetura Antes:**
```
VeiculoController → VeiculoService (God Class) → VeiculoRepository + NotificacaoClient
```

**Arquitetura Depois:**
```
VeiculoController 
    → VeiculoService (Orquestração)
        ├→ VeiculoValidadorService
        ├→ VeiculoQuilometragemService
        ├→ VeiculoManutencaoService
        ├→ VeiculoNotificacaoService
        ├→ VeiculoEstatisticasService
        └→ VeiculoSincronizacaoService
            → VeiculoRepository
            → NotificacaoClient
            → Configurations
```

---

## 5. Alternativas Consideradas

### 5.1 Alternativa 1: Manter uma Única Classe (❌ Rejeitada)

**Descrição:** Não fazer nada, manter `VeiculoService` como está

**Prós:**
- Sem refatoração
- Sem gasto de tempo

**Contras:**
- Viola SRP
- Impossível de testar
- Difícil de manter
- Será debt crescente

**Decisão:** ❌ Rejeitada - identificada como CRÍTICA no TECHNICAL_DEBT_MAP

---

### 5.2 Alternativa 2: Usar Padrão Strategy (⚠️ Parcial)

**Descrição:** Criar strategies para cada tipo de operação

**Prós:**
- Separa lógica
- Mais flexível

**Contras:**
- Ainda mantém classe grande
- Complexidade adicional sem resolver SRP

**Decisão:** ⚠️ Parcial - será combinada com Separação de Serviços

---

### 5.3 Alternativa 3: Separação em Services + Event-Driven (✅ Considerada)

**Descrição:** Além de separação, usar event sourcing para notificações

**Prós:**
- Consistência eventual
- Auditoria completa
- Escalabilidade

**Contras:**
- Complexidade aumentada
- Requer infraestrutura (Kafka/RabbitMQ)

**Decisão:** ✅ Considerada para Fase 2 (pós-refatoração básica)

---

### 5.4 Alternativa 4: Usar Domain-Driven Design (DDD) (✅ Recomendado)

**Descrição:** Implementar Entities, Value Objects, Aggregates do DDD

**Prós:**
- Modelo rico de domínio
- Alinhado com negócio
- Facilita escala

**Contras:**
- Aprendizado necessário
- Mais abstração

**Decisão:** ✅ Recomendado - combinar com Service Separation

---

## 6. Plano de Implementação

### 6.1 Timeline Estimada

```timeline
Semana 1-2: Preparação
├── Criar exceções customizadas (8-10h)
├── Criar enums (VeiculoStatus, TipoNotificacao) (4-6h)
├── Adicionar configurations (6-8h)
└── Criar estrutura de testes (4-6h)
   Total: 22-30h

Semana 3-4: Refatoração Core
├── Refatorar VeiculoService (20-25h)
├── Criar VeiculoValidadorService (15-20h)
├── Criar VeiculoQuilometragemService (12-15h)
├── Criar testes unitários (30-40h)
└── Atualizar VeiculoController (10-15h)
   Total: 87-115h

Semana 5-6: Refatoração Complementar
├── Criar VeiculoManutencaoService (15-20h)
├── Criar VeiculoNotificacaoService (10-15h)
├── Criar VeiculoEstatisticasService (8-12h)
├── Criar VeiculoSincronizacaoService (10-15h)
├── Testes complementários (40-50h)
└── Integração e smoke tests (15-20h)
   Total: 98-132h

Semana 7: Finalização
├── Limpeza de código (10-15h)
├── Documentação (10-15h)
├── Code review e ajustes (15-20h)
└── Deploy em staging (5-10h)
   Total: 40-60h

TOTAL GERAL: 247-337 horas (≈ 6-8 semanas, equipe de 1-2 pessoas)
```

### 6.2 Fases Detalhadas

#### **FASE 1: Preparação (Semana 1-2)**

**Tarefas:**
- [ ] Criar diretório `com/rotalog/exception/` com exceções
- [ ] Criar diretório `com/rotalog/enums/` com enums
- [ ] Criar classe `VeiculoProperties` com `@ConfigurationProperties`
- [ ] Atualizar `application.properties` com configurações
- [ ] Criar teste base `VeiculoServiceTest.java` skeleton
- [ ] Documentar decisão em ADR (este documento)

**Entregáveis:**
- 9 arquivos de exceção
- 2 arquivos de enums
- 1 arquivo de configuração
- 1 arquivo properties atualizado

---

#### **FASE 2: Refatoração Core (Semana 3-4)**

**Tarefas:**
- [ ] Refatorar `VeiculoService` removendo métodos de validação
- [ ] Refatorar `VeiculoService` removendo métodos de quilometragem
- [ ] Criar `VeiculoValidadorService` completo
- [ ] Criar `VeiculoQuilometragemService` completo
- [ ] Converter `@Autowired` field → constructor injection
- [ ] Escrever 33 testes unitários
- [ ] Remover `System.out.println()` de todos os files
- [ ] Remover Javadoc redundante

**Entregáveis:**
- `VeiculoService.java` refatorado (≈120 linhas)
- `VeiculoValidadorService.java` (≈80 linhas)
- `VeiculoQuilometragemService.java` (≈60 linhas)
- 33 testes em VeiculoServiceTest.java

**Cobertura esperada:** 65%

---

#### **FASE 3: Refatoração Complementar (Semana 5-6)**

**Tarefas:**
- [ ] Criar `VeiculoManutencaoService` completo
- [ ] Criar `VeiculoNotificacaoService` completo
- [ ] Criar `VeiculoEstatisticasService` completo
- [ ] Criar `VeiculoSincronizacaoService` completo (com placeholder real)
- [ ] Implementar factory/builder para Veiculo
- [ ] Escrever 50 testes adicionais
- [ ] Atualizar `VeiculoController` para usar novos services
- [ ] Executar code coverage analysis

**Entregáveis:**
- 4 novos services
- 50 testes adicionais
- VeiculoController refatorado
- Coverage report (92%+)

**Cobertura esperada:** 92%

---

#### **FASE 4: Finalização (Semana 7)**

**Tarefas:**
- [ ] Code review completo de todos os files
- [ ] Atualizar javadoc com exemplos
- [ ] Criar README.md com arquitetura
- [ ] Adicionar testes de integração (opcional)
- [ ] Deploy em branch feature
- [ ] Preparar pull request
- [ ] Atualizar TECHNICAL_DEBT_MAP.md

**Entregáveis:**
- PR completo pronto para review
- Documentação atualizada
- Zero warnings de cobertura

---

### 6.3 Estrutura de Arquivos

```
src/main/java/com/rotalog/
├── controller/
│   └── VeiculoController.java (refatorado)
│
├── service/
│   ├── VeiculoService.java (refatorado)
│   ├── VeiculoValidadorService.java (novo)
│   ├── VeiculoQuilometragemService.java (novo)
│   ├── VeiculoManutencaoService.java (novo)
│   ├── VeiculoNotificacaoService.java (novo)
│   ├── VeiculoEstatisticasService.java (novo)
│   └── VeiculoSincronizacaoService.java (novo)
│
├── domain/
│   ├── Veiculo.java (existente)
│   ├── VeiculoStatus.java (novo)
│   └── TipoNotificacao.java (novo)
│
├── exception/
│   ├── VeiculoException.java (novo - base abstrata)
│   ├── VeiculoNaoEncontradoException.java (novo)
│   ├── VeiculoDuplicadoException.java (novo)
│   ├── PlacaInvalidaException.java (novo)
│   ├── ModeloInvalidoException.java (novo)
│   ├── AnoFabricacaoInvalidoException.java (novo)
│   ├── QuilometragemInvalidaException.java (novo)
│   ├── StatusInvalidoException.java (novo)
│   ├── CampoObrigatorioException.java (novo)
│   ├── VeiculoJaEmManutencaoException.java (novo)
│   └── NotificacaoFalhaException.java (novo)
│
├── config/
│   └── VeiculoProperties.java (novo)
│
└── repository/
    └── VeiculoRepository.java (existente)

src/test/java/com/rotalog/
└── service/
    ├── VeiculoServiceTest.java (novo - 15 testes)
    ├── VeiculoValidadorServiceTest.java (novo - 18 testes)
    ├── VeiculoQuilometragemServiceTest.java (novo - 12 testes)
    ├── VeiculoManutencaoServiceTest.java (novo - 14 testes)
    ├── VeiculoNotificacaoServiceTest.java (novo - 8 testes)
    ├── VeiculoEstatisticasServiceTest.java (novo - 10 testes)
    └── VeiculoSincronizacaoServiceTest.java (novo - 6 testes)

src/main/resources/
└── application.properties (atualizado com VeiculoProperties)

docs/
└── adr/
    └── ADR-0001-refatoracao-veiculoservice.md (este documento)
```

---

## 7. Riscos e Mitigation

| Risco | Probabilidade | Impacto | Mitigation |
|-------|------------|--------|-----------|
| Refatoração quebra funcionalidades existentes | 🟠 Média | 🔴 Alto | Testes unitários 90%+, code review rigoroso |
| Controller também precisa refatoração | 🟠 Média | 🟠 Médio | Planejado em FASE 2, pode ser paralelo |
| Descoberta de novos requirements | 🟡 Baixa | 🟡 Baixo | Iteração incremental, PR por feature |
| Equipe não familiar com arquitetura | 🟡 Baixa | 🟠 Médio | Documentação detalhada, pair programming |
| Conflitos em merge com outras branches | 🟡 Baixa | 🟡 Baixo | Feature branch, frequent rebase |
| Complexidade de orquestração aumenta | 🟠 Média | 🟡 Baixo | Design pattern (facade/orchestrator) |

---

## 8. Métricas de Sucesso

### 8.1 Métricas Quantitativas

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| Linhas por classe | 340 | 120-100 | < 150 |
| Métodos por classe | 15 | 5-7 | < 10 |
| Complexidade Ciclomática | Alto | Baixo-Médio | < 5 |
| Cobertura de Testes | 0% | 92% | >= 90% |
| Exceções específicas | 0 | 10 | >= 5 |
| System.out.println() | 3 | 0 | 0 |
| Javadoc lines | 50+ | 15 | < 20 |
| Variáveis não usadas | 2 | 0 | 0 |
| Services independentes | 1 | 7 | >= 6 |

### 8.2 Métricas Qualitativas

- ✅ Cada service tem responsabilidade única clara
- ✅ Código facilmente testável (não precisa de reflection)
- ✅ Fácil adicionar novas validações
- ✅ Fácil estender manuntenção preventiva
- ✅ Logs estruturados e rastreáveis
- ✅ Type safety com enums
- ✅ Configurações não hardcodeadas

### 8.3 Checklist de Validação

- [ ] Todas as 7 novas classes criadas
- [ ] Todas as 10 exceções criadas e testadas
- [ ] 2 enums criados (VeiculoStatus, TipoNotificacao)
- [ ] 83+ testes unitários escritos
- [ ] Coverage >= 90%
- [ ] Zero System.out.println()
- [ ] Zero Javadoc redundante
- [ ] Constructor injection em todas as classes
- [ ] application.properties atualizado
- [ ] VeiculoController refatorado
- [ ] Code review aprovado
- [ ] Merge para main sem conflitos

---

## 9. Próximos Passos

### 9.1 Imediato (Esta Semana)

1. **Aprovação da ADR**
   - Discussão com tech leads
   - Feedback da equipe
   - Ajustes se necessário

2. **Criação de Tickets**
   - Dividir em stories JIRA/Linear
   - Estimar cada story
   - Alocar à sprint

3. **Comunicação**
   - Notificar equipe de frontend
   - Preparar documentação
   - Agendar pairing sessions

### 9.2 Curto Prazo (Próximas 2 Semanas)

1. **Setup**
   - Criar feature branch
   - Criar estrutura de diretórios
   - Começar FASE 1 (Preparação)

2. **Desenvolvimento**
   - Criar exceções
   - Criar enums
   - Criar configurações

3. **Review**
   - Code review inicial
   - Feedback antes de avançar

### 9.3 Médio Prazo (Próximas 6-8 Semanas)

1. **Implementação**
   - Executar FASES 2, 3, 4
   - Iterações frequentes
   - Testes contínuos

2. **Validação**
   - Coverage analysis
   - Performance tests
   - Security review

3. **Finalização**
   - Code freeze
   - PR final
   - Merge para main
   - Documentação

---

## 10. Referências e Recursos

### 10.1 SOLID Principles
- [Single Responsibility Principle (SRP)](https://en.wikipedia.org/wiki/Single-responsibility_principle)
- [Open/Closed Principle (OCP)](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle)
- [Liskov Substitution Principle (LSP)](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
- [Interface Segregation Principle (ISP)](https://en.wikipedia.org/wiki/Interface_segregation_principle)
- [Dependency Inversion Principle (DIP)](https://en.wikipedia.org/wiki/Dependency_inversion_principle)

### 10.2 Spring Boot Best Practices
- [Spring Framework Reference - Service Layer](https://spring.io/guides/gs/securing-web/)
- [Spring Dependency Injection](https://spring.io/guides/gs/messaging-stomp-websocket/)
- [Exception Handling in Spring REST APIs](https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc)

### 10.3 Testing
- [JUnit 5 Documentation](https://junit.org/junit5/docs/current/user-guide/)
- [Mockito Framework](https://site.mockito.org/)
- [Testing Best Practices](https://www.baeldung.com/spring-boot-testing)

### 10.4 Referência Interna
- [TECHNICAL_DEBT_MAP.md](../TECHNICAL_DEBT_MAP.md) - Dívidas técnicas do projeto
- [CLAUDE.md](../CLAUDE.md) - Instruções do projeto

---

## 11. Histórico de Decisão

| Data | Decisão | Status |
|------|---------|--------|
| 2026-06-18 | Criar ADR para refatoração de VeiculoService | ✅ PROPOSTO |
| 2026-06-XX | Revisão e aprovação com tech leads | ⏳ PENDENTE |
| 2026-06-XX | Criar tickets de implementação | ⏳ PENDENTE |
| 2026-06-XX | Iniciar FASE 1 (Preparação) | ⏳ PENDENTE |

---

## 12. Conclusão

A refatoração da classe `VeiculoService` é **CRÍTICA** para a saúde do projeto. Esta ADR propõe uma separação clara de responsabilidades em 7 serviços especializados, implementação de exceções específicas, cobertura de testes 90%+, e eliminação de anti-patterns.

Com a execução conforme plano, o projeto terá:
- ✅ Código mais manutenível
- ✅ Testes confiáveis
- ✅ Arquitetura escalável
- ✅ Developer experience melhorado
- ✅ Facilitação de novos features

**Tempo estimado:** 247-337 horas (6-8 semanas)  
**Prioridade:** 🔴 CRÍTICA  
**Impacto:** Alto no longo prazo

---

**Documento Criado:** 2026-06-18  
**Versão:** 1.0  
**Autor:** Análise Técnica - Claude Code  
**Status:** PROPOSTO - Aguardando Aprovação
