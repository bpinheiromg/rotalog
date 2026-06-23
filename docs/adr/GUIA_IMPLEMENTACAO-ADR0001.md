# Guia de Implementação - Refatoração VeiculoService

**Data:** 2026-06-18  
**Referência:** ADR-0001  
**Status:** DOCUMENTO DE PLANEJAMENTO

---

## 1. Diagrama de Arquitetura

### Antes (God Class - Atual)
```
┌──────────────────────────────────────────────────────┐
│                 VeiculoController                    │
│  (requisições HTTP)                                  │
└────────────────┬─────────────────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────────────────┐
│          VeiculoService (God Class)                  │
│                                                      │
│  • CRUD ────────────────────────────────────────┐   │
│  • Validação ────────────────────────────────┐  │   │
│  • Quilometragem ──────────────────────────┐ │  │   │
│  • Manutenção ──────────────────────────┐  │ │  │   │
│  • Notificação ─────────────────────┐   │  │ │  │   │
│  • Estatísticas ─────────────────┐  │   │  │ │  │   │
│  • Sincronização ──────────────┐ │  │   │  │ │  │   │
└──────┬──────────────────┬──────┬─┴┬──┴───┴──┴─┴──┘   │
       │                  │      │  │         (340 L)   │
       ▼                  ▼      ▼  ▼                    │
   ┌─────────┐      ┌──────────────────┐
   │Repository│      │NotificacaoClient │
   └─────────┘      └──────────────────┘
```

### Depois (Serviços Especializados)
```
┌──────────────────────────────────────────────────────────┐
│               VeiculoController (Refatorado)             │
│  (validação delegada, uso de DTOs)                       │
└────────────────┬─────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │ VeiculoService         │
        │ (Orquestrador)         │
        │ ~120 linhas            │
        └────┬───┬───┬───┬───┬──┬─┘
             │   │   │   │   │  │
    ┌────────┘   │   │   │   │  └──────────────────┐
    │            │   │   │   │                     │
    ▼            ▼   ▼   ▼   ▼                     ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│Validador │ │Quilometr.│ │Manutenção│ │Notificação│ │Estatísticas│ │Sinc. ext.│ │Repository│
│Service   │ │Service   │ │Service   │ │Service   │ │Service     │ │Service   │ │(Existente)
│~80 L     │ │~60 L     │ │~100 L    │ │~50 L     │ │~40 L       │ │~60 L     │ │
└──────────┘ └──────────┘ └──────────┘ └──────┬───┘ └────────────┘ └──────────┘ └──────────┘
                                               │
                                               ▼
                                        ┌──────────────────┐
                                        │NotificacaoClient │
                                        │(Existente)       │
                                        └──────────────────┘
```

**Benefício:** Cada serviço com responsabilidade única e clara

---

## 2. Matriz de Distribuição de Responsabilidades

```
┌──────────────────────┬───────────────────────────────────────────────────────────┐
│  Responsabilidade    │ ANTES (VeiculoService)      │ DEPOIS (7 Serviços)        │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ CRUD Básico          │ VeiculoService              │ VeiculoService             │
│ (Create, Read,       │ • registrarVeiculo()       │ • registrarVeiculo()       │
│  Update, Delete)     │ • buscarPorId()            │ • buscarPorId()            │
│                      │ • buscarPorPlaca()         │ • buscarPorPlaca()         │
│                      │ • listarTodos()            │ • listarTodos()            │
│                      │ • atualizarVeiculo()       │ • atualizarVeiculo()       │
│                      │ • desativarVeiculo()       │ • desativarVeiculo()       │
│                      │ • reativarVeiculo()        │ • reativarVeiculo()        │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Validação            │ VeiculoService (40 linhas) │ VeiculoValidadorService    │
│ • Placa              │ • Lógica espalhada no      │ • validarPlaca()           │
│ • Modelo             │   registrarVeiculo()       │ • validarModelo()          │
│ • Ano Fabricação     │ • Sem método específico    │ • validarAnoFabricacao()   │
│ • Quilometragem      │ • Sem reutilização        │ • validarQuilometragem()   │
│ • Status             │                            │ • validarStatus()          │
│                      │                            │ • validarCampoObrigatorio()│
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Quilometragem        │ VeiculoService (50 linhas) │ VeiculoQuilometragemSvc    │
│ • Atualizar          │ • atualizarQuilometragem() │ • atualizarQuilometragem() │
│ • Validar redução    │ • Misturado com notif.     │ • validarReducao()         │
│ • Histórico          │ • Sem auditoria            │ • registrarHistorico()     │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Manutenção           │ VeiculoService (60 linhas) │ VeiculoManutencaoService   │
│ Preventiva           │ • agendarManutencaoPrev()  │ • agendarManutencao()      │
│ • Agenda             │ • calcularCustoManutencao()│ • calcularCusto()          │
│ • Custo              │ • precisaDeManutencao()    │ • verificarNecessidade()   │
│ • Verificar          │ • Valores hardcodeados     │ • loadConfiguration()      │
│   necessidade        │ • Sem config               │ • validateManutenance()    │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Notificações         │ VeiculoService (30 linhas) │ VeiculoNotificacaoService  │
│ • Novo veículo       │ • Try-catch em 4 métodos   │ • notificarNovoVeiculo()   │
│ • Manutenção         │ • Engolição de exceção     │ • notificarManutencao()    │
│ • Desativação        │ • Email hardcodeado        │ • notificarDesativacao()   │
│                      │ • Sem retry                │ • notificarReativacao()    │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Estatísticas         │ VeiculoService (15 linhas) │ VeiculoEstatisticasService │
│ • Count por status   │ • obterEstatisticasFreita()│ • obterEstatisticas()      │
│ • Total              │ • Typo no nome             │ • obterPorStatus()         │
│ • Relatórios         │ • Retorna String JSON      │ • construirRelatorio()     │
│                      │ • Query ineficiente        │ (Retorna DTO)              │
├──────────────────────┼────────────────────────────────────────────────────────────┤
│ Sincronização        │ VeiculoService (10 linhas) │ VeiculoSincronizacaoSvc    │
│ Externa              │ • sincronizarComSist...()  │ • sincronizar()            │
│ • Integração         │ • Método vazio             │ • configurarRetry()        │
│ • Retry              │ • TODOs não implementados  │ • configCircuitBreaker()   │
│ • Circuit Breaker    │                            │ • handleError()            │
└──────────────────────┴────────────────────────────────────────────────────────────┘
```

---

## 3. Matriz de Testes Unitários (83 Testes)

### VeiculoServiceTest (15 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoServiceTest.java (15 testes)                     │
├─────────────────────────────────────────────────────────┤
│ CRUD Operations                                         │
│ ├─ test_listarTodos_Success()                          │
│ ├─ test_listarTodos_Empty()                            │
│ ├─ test_listarTodos_WithPagination()                   │
│ ├─ test_buscarPorId_Success()                          │
│ ├─ test_buscarPorId_NotFound()                         │
│ ├─ test_buscarPorPlaca_Success()                       │
│ ├─ test_buscarPorPlaca_NotFound()                      │
│ ├─ test_atualizarVeiculo_Success()                     │
│ └─ test_atualizarVeiculo_NotFound()                    │
│ Inativação                                              │
│ ├─ test_desativarVeiculo_Success()                     │
│ ├─ test_desativarVeiculo_NotFound()                    │
│ └─ test_reativarVeiculo_Success()                      │
│ Orquestração                                            │
│ ├─ test_registrarVeiculo_Success()                     │
│ └─ test_registrarVeiculo_ValidadorException()          │
└─────────────────────────────────────────────────────────┘
```

### VeiculoValidadorServiceTest (18 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoValidadorServiceTest.java (18 testes)            │
├─────────────────────────────────────────────────────────┤
│ Validação de Placa                                      │
│ ├─ test_validarPlaca_Valid()                           │
│ ├─ test_validarPlaca_NullThrowsException()             │
│ ├─ test_validarPlaca_EmptyThrowsException()            │
│ ├─ test_validarPlaca_WrongLengthThrowsException()      │
│ ├─ test_validarPlaca_InvalidFormatThrowsException()    │
│ └─ test_validarPlaca_DuplicateThrowsException()        │
│ Validação de Modelo                                     │
│ ├─ test_validarModelo_Valid()                          │
│ ├─ test_validarModelo_NullThrowsException()            │
│ └─ test_validarModelo_EmptyThrowsException()           │
│ Validação de Ano                                        │
│ ├─ test_validarAno_Valid()                             │
│ ├─ test_validarAno_MinBoundary()                       │
│ ├─ test_validarAno_MaxBoundary()                       │
│ └─ test_validarAno_OutOfRangeThrowsException()         │
│ Validação de Status                                     │
│ ├─ test_validarStatus_AllValidStates()                 │
│ └─ test_validarStatus_InvalidThrowsException()         │
└─────────────────────────────────────────────────────────┘
```

### VeiculoQuilometragemServiceTest (12 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoQuilometragemServiceTest.java (12 testes)        │
├─────────────────────────────────────────────────────────┤
│ Atualização de Quilometragem                            │
│ ├─ test_atualizarQuilometragem_IncreaseSuccess()      │
│ ├─ test_atualizarQuilometragem_DecreaseThrowsEx()     │
│ ├─ test_atualizarQuilometragem_NegativeThrowsEx()     │
│ ├─ test_atualizarQuilometragem_ZeroSuccess()          │
│ └─ test_atualizarQuilometragem_VeiculoNotFound()      │
│ Validação de Redução                                    │
│ ├─ test_validarReducao_ReductionThrowsEx()            │
│ ├─ test_validarReducao_SameValueOk()                  │
│ └─ test_validarReducao_IncrementOk()                  │
│ Histórico                                               │
│ ├─ test_registrarHistorico_Success()                  │
│ ├─ test_registrarHistorico_TimestampCorrect()         │
│ └─ test_registrarHistorico_AllFieldsPresent()         │
└─────────────────────────────────────────────────────────┘
```

### VeiculoManutencaoServiceTest (14 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoManutencaoServiceTest.java (14 testes)           │
├─────────────────────────────────────────────────────────┤
│ Agendamento                                              │
│ ├─ test_agendarManutencao_Success()                    │
│ ├─ test_agendarManutencao_VeiculoNotFound()            │
│ ├─ test_agendarManutencao_AlreadyScheduled()           │
│ └─ test_agendarManutencao_InvalidDate()                │
│ Cálculo de Custo                                         │
│ ├─ test_calcularCusto_Basic()                          │
│ ├─ test_calcularCusto_HighMileage()                    │
│ ├─ test_calcularCusto_ZeroMileage()                    │
│ └─ test_calcularCusto_ConfigurableValues()             │
│ Verificação                                              │
│ ├─ test_verificarNecessidade_AboveThreshold()          │
│ ├─ test_verificarNecessidade_BelowThreshold()          │
│ └─ test_verificarNecessidade_ExactlyOnThreshold()      │
│ Configuração                                             │
│ └─ test_loadConfiguration_FromProperties()             │
└─────────────────────────────────────────────────────────┘
```

### VeiculoNotificacaoServiceTest (8 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoNotificacaoServiceTest.java (8 testes)           │
├─────────────────────────────────────────────────────────┤
│ Notificação de Novo Veículo                             │
│ ├─ test_notificarNovoVeiculo_Success()                 │
│ └─ test_notificarNovoVeiculo_ClientFailure()           │
│ Notificação de Manutenção                               │
│ ├─ test_notificarManutencao_Success()                  │
│ └─ test_notificarManutencao_ClientFailure()            │
│ Notificação de Desativação                              │
│ ├─ test_notificarDesativacao_Success()                 │
│ └─ test_notificarDesativacao_ClientFailure()           │
│ Notificação de Reativação                               │
│ └─ test_notificarReativacao_Success()                  │
└─────────────────────────────────────────────────────────┘
```

### VeiculoEstatisticasServiceTest (10 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoEstatisticasServiceTest.java (10 testes)         │
├─────────────────────────────────────────────────────────┤
│ Estatísticas Gerais                                      │
│ ├─ test_obterEstatisticas_WithData()                   │
│ ├─ test_obterEstatisticas_EmptyDatabase()              │
│ ├─ test_obterEstatisticas_ReturnsDTO()                 │
│ └─ test_obterEstatisticas_CountsCorrect()              │
│ Por Status                                               │
│ ├─ test_obterPorStatus_Ativo()                         │
│ ├─ test_obterPorStatus_Inativo()                       │
│ ├─ test_obterPorStatus_Manutencao()                    │
│ ├─ test_obterPorStatus_InvalidStatus()                 │
│ ├─ test_obterPorStatus_Empty()                         │
│ └─ test_obterPorStatus_Paginated()                     │
└─────────────────────────────────────────────────────────┘
```

### VeiculoSincronizacaoServiceTest (6 testes)

```
┌─────────────────────────────────────────────────────────┐
│ VeiculoSincronizacaoServiceTest.java (6 testes)         │
├─────────────────────────────────────────────────────────┤
│ Sincronização                                            │
│ ├─ test_sincronizar_Success()                          │
│ ├─ test_sincronizar_ExternalServiceFailure()           │
│ ├─ test_sincronizar_RetryLogic()                       │
│ └─ test_sincronizar_CircuitBreakerTrips()              │
│ Configuração                                             │
│ ├─ test_configurarRetry_RespectsSettings()             │
│ └─ test_configCircuitBreaker_RespectsThresholds()      │
└─────────────────────────────────────────────────────────┘
```

**TOTAL: 83 testes → ~92% cobertura esperada**

---

## 4. Mapa de Exceções

```
VeiculoException (Base Abstrata)
│
├─ VeiculoNaoEncontradoException
│  └─ Quando: buscarPorId(), buscarPorPlaca() retornam empty
│  └─ HTTP: 404 Not Found
│  └─ Mensagem: "Veículo com ID {id} não encontrado"
│
├─ VeiculoDuplicadoException
│  └─ Quando: registrarVeiculo() com placa duplicada
│  └─ HTTP: 409 Conflict
│  └─ Mensagem: "Veículo com placa {placa} já existe"
│
├─ PlacaInvalidaException
│  └─ Quando: Placa não tem 7 caracteres ou formato inválido
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Placa inválida: deve ter 7 caracteres"
│
├─ ModeloInvalidoException
│  └─ Quando: Modelo vazio ou nulo
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Modelo é obrigatório"
│
├─ AnoFabricacaoInvalidoException
│  └─ Quando: Ano < 1900 ou > 2100
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Ano de fabricação deve estar entre 1900 e 2100"
│
├─ QuilometragemInvalidaException
│  └─ Quando: Quilometragem < 0 ou redução detectada
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Quilometragem inválida: {valor}"
│
├─ StatusInvalidoException
│  └─ Quando: obterVeiculosPorStatus() com status não reconhecido
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Status '{status}' não reconhecido. Valores válidos: ATIVO, INATIVO, MANUTENCAO"
│
├─ CampoObrigatorioException
│  └─ Quando: Campo obrigatório está vazio/nulo
│  └─ HTTP: 400 Bad Request
│  └─ Mensagem: "Campo {campo} é obrigatório"
│
├─ VeiculoJaEmManutencaoException
│  └─ Quando: agendarManutencao() em veículo já em manutenção
│  └─ HTTP: 409 Conflict
│  └─ Mensagem: "Veículo {placa} já está em manutenção"
│
└─ NotificacaoFalhaException
   └─ Quando: Falha ao enviar notificação para serviço externo
   └─ HTTP: 503 Service Unavailable
   └─ Mensagem: "Falha ao enviar notificação: {detalhes}"
```

---

## 5. Enums Criados

### VeiculoStatus.java

```java
public enum VeiculoStatus {
    ATIVO("Veículo ativo e disponível") {
        @Override
        public boolean podeSerDesativado() { return true; }
    },
    INATIVO("Veículo desativado") {
        @Override
        public boolean podeSerDesativado() { return false; }
    },
    MANUTENCAO("Veículo em manutenção") {
        @Override
        public boolean podeSerDesativado() { return false; }
    };
    
    private final String descricao;
    
    VeiculoStatus(String descricao) {
        this.descricao = descricao;
    }
    
    public String getDescricao() { return descricao; }
    public abstract boolean podeSerDesativado();
    public static VeiculoStatus fromString(String status) { ... }
}
```

### TipoNotificacao.java

```java
public enum TipoNotificacao {
    NOVO_VEICULO("Notificação: novo veículo cadastrado"),
    ALERTA_MANUTENCAO("Alerta: manutenção necessária"),
    MANUTENCAO_AGENDADA("Confirmação: manutenção agendada"),
    VEICULO_DESATIVADO("Notificação: veículo desativado"),
    VEICULO_REATIVADO("Notificação: veículo reativado");
    
    private final String descricao;
    
    TipoNotificacao(String descricao) {
        this.descricao = descricao;
    }
    
    public String getDescricao() { return descricao; }
}
```

---

## 6. Configurações (application.properties)

```properties
# ==========================================
# Veiculo Service Configuration
# ==========================================

# Validação
veiculo.placa.tamanho=7
veiculo.ano-fabricacao.minimo=1900
veiculo.ano-fabricacao.maximo=2100

# Quilometragem
veiculo.quilometragem.limite-manutencao=50000
veiculo.quilometragem.intervalo=10000

# Manutenção Preventiva
veiculo.manutencao.intervalo-meses=3
veiculo.manutencao.custo-base=500.0
veiculo.manutencao.custo-por-km=0.05

# Notificações
notificacao.email-gestor=gestor@rotalog.com
notificacao.retry-attempts=3
notificacao.retry-delay-ms=1000

# Sincronização Externa
sincronizacao.enabled=false
sincronizacao.url=https://sistema-externo.com/api
sincronizacao.timeout-ms=5000
sincronizacao.retry-attempts=3
sincronizacao.circuit-breaker-threshold=5
```

---

## 7. Fluxo de Desenvolvimento

### Sprint 1: Preparação (22-30h)
```
├─ Branch: feature/veiculo-refactor-prep
├─ Tarefas:
│  ├─ Criar 10 exceções em package com/rotalog/exception/
│  ├─ Criar 2 enums em package com/rotalog/domain/
│  ├─ Criar VeiculoProperties com @ConfigurationProperties
│  ├─ Atualizar application.properties
│  ├─ Criar VeiculoServiceTest.java skeleton
│  └─ Documentar em ADR
├─ Review: Tech Lead
├─ Merge: feature → main
└─ Status: ✅ PRONTO PARA FASE 2
```

### Sprint 2-3: Refatoração Core (87-115h)
```
├─ Branch: feature/veiculo-refactor-core
├─ Tarefas:
│  ├─ Refatorar VeiculoService (core CRUD)
│  ├─ Criar VeiculoValidadorService
│  ├─ Criar VeiculoQuilometragemService
│  ├─ Converter @Autowired → Constructor injection
│  ├─ Escrever 33 testes (coverage: 65%)
│  ├─ Remover System.out.println()
│  ├─ Remover Javadoc redundante
│  └─ Atualizar VeiculoController
├─ Review: 2 Tech Leads + 1 QA
├─ Coverage: 65%
├─ Merge: feature → main
└─ Status: ✅ PRONTO PARA FASE 3
```

### Sprint 4-5: Refatoração Complementar (98-132h)
```
├─ Branch: feature/veiculo-refactor-complete
├─ Tarefas:
│  ├─ Criar VeiculoManutencaoService
│  ├─ Criar VeiculoNotificacaoService
│  ├─ Criar VeiculoEstatisticasService
│  ├─ Criar VeiculoSincronizacaoService
│  ├─ Implementar factory/builder patterns
│  ├─ Escrever 50 testes adicionais (coverage: 92%)
│  ├─ Executar code coverage analysis
│  └─ Testes de integração (opcional)
├─ Review: Tech Lead
├─ Coverage: 92%
├─ Merge: feature → main
└─ Status: ✅ PRONTO PARA FASE 4
```

### Sprint 6: Finalização (40-60h)
```
├─ Branch: feature/veiculo-refactor-final
├─ Tarefas:
│  ├─ Code review completo
│  ├─ Atualizar Javadoc/README
│  ├─ Performance tests
│  ├─ Security review
│  └─ Deploy em staging
├─ QA: Smoke tests completos
├─ Review: CTO/Arquiteto
├─ Merge: feature → main
└─ Status: ✅ PRODUCTION READY
```

---

## 8. Checklist de Implementação Detalhado

### FASE 1: Preparação
```
Exceções:
  □ VeiculoException (abstract base class)
  □ VeiculoNaoEncontradoException
  □ VeiculoDuplicadoException
  □ PlacaInvalidaException
  □ ModeloInvalidoException
  □ AnoFabricacaoInvalidoException
  □ QuilometragemInvalidaException
  □ StatusInvalidoException
  □ CampoObrigatorioException
  □ VeiculoJaEmManutencaoException
  □ NotificacaoFalhaException (10/10)

Enums:
  □ VeiculoStatus
  □ TipoNotificacao (2/2)

Configuração:
  □ VeiculoProperties class
  □ application.properties atualizado (com todas as props)

Testes:
  □ Base test infrastructure criada
  □ Annotations configuradas (@ExtendWith, etc)
  □ Fixtures preparadas
```

### FASE 2: Refatoração Core
```
Refatoração:
  □ VeiculoService refatorado (120 L) - CRUD + Orquestração
  □ VeiculoValidadorService criado (80 L) - Todas validações
  □ VeiculoQuilometragemService criado (60 L) - Quilometragem
  □ Constructor injection implementado (3/3 classes)
  □ System.out.println() removido
  □ Javadoc redundante removido

Testes:
  □ VeiculoServiceTest (15 testes)
  □ VeiculoValidadorServiceTest (18 testes)
  □ VeiculoQuilometragemServiceTest (12 testes)
  □ Coverage: 65%
  □ Todos os testes PASSING (33/33)

Atualização:
  □ VeiculoController refatorado
  □ DTOs criados/atualizados
```

### FASE 3: Refatoração Complementar
```
Services:
  □ VeiculoManutencaoService criado (100 L)
  □ VeiculoNotificacaoService criado (50 L)
  □ VeiculoEstatisticasService criado (40 L)
  □ VeiculoSincronizacaoService criado (60 L)

Testes:
  □ VeiculoManutencaoServiceTest (14 testes)
  □ VeiculoNotificacaoServiceTest (8 testes)
  □ VeiculoEstatisticasServiceTest (10 testes)
  □ VeiculoSincronizacaoServiceTest (6 testes)
  □ Coverage: 92%
  □ Todos os testes PASSING (50/50)

Otimizações:
  □ Queries otimizadas (count() vs findAll().size())
  □ Paginação implementada
  □ Método obterEstatisticasFreita() renomeado
```

### FASE 4: Finalização
```
Qualidade:
  □ Code review completo (sem comments pendentes)
  □ SonarQube checks: PASSING
  □ Checkstyle: PASSING (zero warnings)
  □ Coverage: >= 90%

Documentação:
  □ Javadoc atualizado (sem redundância)
  □ README com arquitetura
  □ ADR completo
  □ Exemplos de uso

Testes:
  □ Unit tests: 100% passing
  □ Integration tests: 100% passing
  □ Performance tests: Acceptable

Deploy:
  □ Merge para main
  □ Build em CI/CD: SUCCESS
  □ Deploy staging: SUCCESS
  □ Smoke tests: PASSING
  □ TECHNICAL_DEBT_MAP.md atualizado
```

---

## 9. Métricas de Sucesso - Dashboard

```
ANTES vs DEPOIS:

Linhas por Classe:
  ANTES: ████████████████████████ (340 L)
  DEPOIS: ████ (120 L) ✅ -65%

Complexidade Ciclomática:
  ANTES: █████████████ (Alto)
  DEPOIS: █████ (Baixo) ✅ -60%

Métodos por Classe:
  ANTES: ███████ (15)
  DEPOIS: ██ (5-7) ✅ -60%

Cobertura de Testes:
  ANTES: (0%)
  DEPOIS: ███████████████████ (92%) ✅ +92%

Exceções Específicas:
  ANTES: (0)
  DEPOIS: ██████████ (10) ✅ +10

Code Smells:
  ANTES: ██████████ (10+)
  DEPOIS: ██ (2) ✅ -80%

Maintainability Index:
  ANTES: 35 (Baixo)
  DEPOIS: 85 (Alto) ✅ +143%
```

---

## 10. Riscos de Implementação

| Risco | Mitigation | Contingency |
|-------|-----------|------------|
| Quebra testes existentes | 90%+ cobertura nova | Rollback rápido |
| Performance degradation | Performance tests | Otimização após análise |
| Conflitos merge | Feature branch pequena | Frequent rebase |
| Equipe não familiarizada | Pair programming | Workshop antes |

---

**Documento:** Guia de Implementação - ADR-0001  
**Criado:** 2026-06-18  
**Versão:** 1.0  
**Status:** PLANEJAMENTO
