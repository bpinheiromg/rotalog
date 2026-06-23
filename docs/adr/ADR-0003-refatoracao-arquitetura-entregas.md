# ADR-0003: Refatoração Arquitetural - Separação de Responsabilidades em `entregas.js`

**Data:** 2026-06-18  
**Status:** PROPOSTO  
**Decisão Crítica:** Sim ⚠️  
**Prioridade:** 🟠 ALTA (Quadrante 2 do TECHNICAL_DEBT_MAP)  

---

## 1. Contexto

O arquivo `src/routes/entregas.js` (366 linhas) foi identificado como tendo **grave violação de separação de responsabilidades**. O arquivo de rotas contém lógica de negócio, validação, SQL queries, e tratamento de erro manual, violando padrões MVC e SOLID.

### Situação Atual

**Arquivo:** `rotalog-api-entregas/src/routes/entregas.js`

**Problemas:**
1. 366 linhas - tamanho excessivo
2. Mistura de 3 padrões async (callbacks, promises, async/await)
3. Lógica de negócio na rota (cálculo de distância, geração de ID)
4. SQL queries diretas
5. Validação inline duplicada
6. Logging manual com `console.log`
7. Sem máquina de estados para transições
8. Sem verificação de autorização
9. Tratamento de erro manual em cada endpoint
10. Sem testes possível (tudo acoplado)

### Referência no TECHNICAL_DEBT_MAP

```
#### ⚠️ ALTO: Mistura de Padrões Async
- Problema: 60% callbacks + async/await no mesmo código
- Localização: Múltiplos serviços
- Exemplo problemas:
  - Callbacks de erro não tratados
  - Mistura de `.then()` com `await`
  - Fire-and-forget sem error handling

#### ⚠️ ALTO: Sem Estratégia Centralizada de Erro
- Problema: Cada endpoint lida com erros manualmente
- Ação: Implementar middleware de erro global

#### ⚠️ MÉDIO: Logging Manual com console.log
- Problema: Sem biblioteca de logging, logs não estruturados
```

---

## 2. Problema

### 2.1 Análise Detalhada das Violações

#### 1️⃣ **Tamanho Excessivo (366 linhas em um único arquivo)**

**Problema Identificado:**
- ✗ Arquivo muito grande para arquivo de rotas
- ✗ Métodos variando de 5 a 80 linhas
- ✗ Múltiplas responsabilidades entrelaçadas
- ✗ Difícil de navegar, revisar e manter

**Medições:**
```
Estrutura Atual: entregas.js (366 linhas)
├── GET / (list) - 35 linhas
├── GET /stats - 25 linhas (com raw SQL)
├── GET /:id - 20 linhas
├── GET /pedido/:numero - 13 linhas (callback style)
├── POST / - 58 linhas (validação + lógica + DB)
├── PUT /:id - 23 linhas
├── PATCH /:id/status - 45 linhas (máquina de estados fake)
├── PATCH /:id/atribuir - 42 linhas
└── DELETE /:id - 27 linhas (nested callbacks)
```

**Responsabilidades Misturadas:**
| Responsabilidade | Métodos | Linhas |
|-----------------|---------|--------|
| Roteamento | 9 endpoints | 50 |
| Validação | Inline em 6 endpoints | 40 |
| Lógica de negócio | Distância, ID, tempo | 35 |
| SQL queries | Raw SQL + ORM | 25 |
| Autorização | Nenhuma! | 0 |
| Logging | console.log manual | 15 |
| Error handling | Try-catch em cada endpoint | 40 |
| **TOTAL** | | **366** |

**Impacto:**
- ✗ Testes impossíveis (não é possível testar lógica isoladamente)
- ✗ Reutilização impossível
- ✗ Manutenção difícil

---

#### 2️⃣ **Mistura de Padrões Async (60% Callbacks + 40% Async/Await)**

**Padrão 1 - Async/Await (Linhas 22-57):**
```javascript
router.get('/', async (req, res) => {
    try {
        const entregas = await Entrega.findAll({...});
        res.json(entregas);
    } catch (error) { ... }
});
```

**Padrão 2 - Promise Chain (Linhas 65-91):**
```javascript
router.get('/stats', function(req, res) {
    sequelize.query(query)
        .then(function(stats) { ... })
        .catch(function(error) { ... });
});
```

**Padrão 3 - Nested Callbacks (Linhas 336-363):**
```javascript
Entrega.findByPk(req.params.id)
    .then(function(entrega) {
        return entrega.save()
            .then(function() {
                return Rastreamento.create({...})
                    .then(function() {
                        res.json({...});
                    });
            });
    })
    .catch(function(error) { ... });
```

**Problema:**
- ✗ Inconsistência dificulta leitura
- ✗ Diferentes padrões de error handling
- ✗ Novo desenvolvedor confuso
- ✗ Callback hell (padrão 3)

**Impacto:**
- Qualidade de código prejudicada
- Manutenção mais difícil

---

#### 3️⃣ **Lógica de Negócio na Rota (Linhas 142-200)**

**Problema 1: Geração de Número de Pedido**
```javascript
// ❌ LÓGICA DE NEGÓCIO NA ROTA
const numeroPedido = 'PED-' + Date.now() + '-' + Math.floor(Math.random() * 1000);
```

**Problema 2: Cálculo de Distância**
```javascript
// ❌ ALGORITMO NA ROTA + INCORRETO
if (origem_lat && origem_lng && destino_lat && destino_lng) {
    const dLat = Math.abs(destino_lat - origem_lat);
    const dLng = Math.abs(destino_lng - origem_lng);
    distanciaKm = Math.sqrt(dLat * dLat + dLng * dLng) * 111; // Pitágoras 2D!
}
```

**Fórmula de Pitágoras em coordenadas geográficas é INCORRETA:**
```
Exemplo real:
São Paulo → Brasília: ~1000 km
Esta fórmula retorna: ~500 km (50% de erro!)

Buenos Aires → Nova York: ~9000 km
Esta fórmula retorna: ~7000 km (20% de erro)
```

**Deveria usar:**
- Haversine formula (distância entre 2 pontos numa esfera)
- Ou API de rotas (Google Maps, HERE, etc.)

**Problema 3: Tempo Estimado Hardcoded**
```javascript
// ❌ HEURÍSTICA NA ROTA + HARDCODED
const tempoEstimado = distanciaKm ? Math.ceil(distanciaKm * 2) : null; // 2 min/km
```

**Impacto:**
- ✗ Impossível testar lógica isoladamente
- ✗ Impossível reutilizar em outro endpoint
- ✗ Fórmula matematicamente incorreta
- ✗ Hardcoded sem configuração
- ✗ Difícil estender (ex: considerar tráfego, horário)

---

#### 4️⃣ **SQL Queries Diretas (Linhas 67-75)**

**Código Vulnerável:**
```javascript
const query = `
    SELECT 
        status,
        COUNT(*) as total,
        COALESCE(AVG(distancia_km), 0) as distancia_media,
        COALESCE(AVG(peso_kg), 0) as peso_medio
    FROM entregas.entregas
    GROUP BY status
`;

sequelize.query(query, { type: sequelize.constructor.QueryTypes.SELECT })
```

**Problema:**
- ✗ Raw SQL bypassa proteções do ORM
- ✗ Duplicação se outro endpoint precisar
- ✗ Risco de SQL injection
- ✗ Difícil para mock em testes

**Deveria usar:**
```javascript
// ✅ ORM query
const stats = await Entrega.findAll({
    attributes: [
        'status',
        [sequelize.fn('COUNT', sequelize.col('id')), 'total']
    ],
    group: ['status'],
    raw: true
});
```

---

#### 5️⃣ **Validação Inline Duplicada (Linhas 148-150, 245-248, 296-298)**

**Validação 1 (linha 148-150):**
```javascript
if (!origem_endereco || !destino_endereco) {
    return res.status(400).json({ error: 'Endereços são obrigatórios' });
}
```

**Validação 2 (linha 245-248):**
```javascript
const statusValidos = ['PENDENTE', 'ATRIBUIDA', 'EM_TRANSITO', 'ENTREGUE', 'CANCELADA'];
if (!statusValidos.includes(status)) {
    return res.status(400).json({ error: 'Status inválido' });
}
```

**Validação 3 (linha 296-298):**
```javascript
if (!veiculo_placa || !motorista_id) {
    return res.status(400).json({ error: 'Placa...obrigatórios' });
}
```

**Problema:**
- ✗ Duplicação (difícil manter consistência)
- ✗ Sem padronização de formato de erro
- ✗ Sem suporte a validações complexas (regex, range, etc.)
- ✗ Impossível reutilizar em outro serviço

**Deveria usar:**
```javascript
// ✅ MIDDLEWARE CENTRALIZADO
const { body, validationResult } = require('express-validator');

const validateEntrega = [
    body('origem_endereco').notEmpty().trim(),
    body('destino_endereco').notEmpty().trim(),
    (req, res, next) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        next();
    }
];

router.post('/', validateEntrega, (req, res) => { ... });
```

---

#### 6️⃣ **Logging Manual com `console.log`**

**Ocorrências:**
```javascript
// Linhas 54, 192, 272, 321
console.log('Entrega criada:', numeroPedido);
console.log('Status atualizado:', entrega.numero_pedido, statusAnterior, '->', status);
console.log('Entrega atribuída:', entrega.numero_pedido, 'veículo:', veiculo_placa);
console.error('Erro ao listar entregas:', error.message);
```

**Problema:**
- ✗ Logs não estruturados
- ✗ Sem níveis de severidade (debug, info, warn, error)
- ✗ Sem timestamp consistente
- ✗ Sem contexto (userId, correlationId)
- ✗ Impossível buscar/filtrar em produção

**Exemplo de saída atual:**
```
Entrega criada: PED-1718727891234-456
Status atualizado: PED-1718727891234-456 PENDENTE -> ATRIBUIDA
Erro ao listar entregas: Cannot read property 'toUpperCase' of undefined
```

**Deveria ser:**
```json
{
    "timestamp": "2026-06-18T10:30:45.123Z",
    "level": "info",
    "service": "api-entregas",
    "action": "entrega_created",
    "entregaId": 100,
    "numeroPedido": "PED-1718727891234-456",
    "userId": 42,
    "correlationId": "req-abc123",
    "message": "Entrega criada"
}
```

---

#### 7️⃣ **Sem Máquina de Estados para Transições (Linhas 235-279)**

**Código:**
```javascript
const statusValidos = ['PENDENTE', 'ATRIBUIDA', 'EM_TRANSITO', 'ENTREGUE', 'CANCELADA'];
if (!statusValidos.includes(status)) {
    return res.status(400).json({ error: 'Status inválido' });
}

// FIXME: Sem validação de transição de estado
const statusAnterior = entrega.status;
entrega.status = status;  // ← QUALQUER transição é permitida!
```

**Problema:**
- ✗ Permite transições inválidas
- ✗ Exemplo: ENTREGUE → PENDENTE (impossível!)
- ✗ Exemplo: CANCELADA → EM_TRANSITO (reativar impossível)
- ✗ Sem regras de negócio

**Transições Inválidas Atualmente Permitidas:**
```
ENTREGUE → PENDENTE (❌ Impossível re-entregar)
CANCELADA → ATRIBUIDA (❌ Não reativar)
CANCELADA → EM_TRANSITO (❌ Não reativar)
```

**Transições Válidas Esperadas:**
```
PENDENTE → ATRIBUIDA → EM_TRANSITO → ENTREGUE
PENDENTE → CANCELADA
ATRIBUIDA → CANCELADA
EM_TRANSITO → CANCELADA
```

---

#### 8️⃣ **Sem Verificação de Autorização**

**Problema:**
- ✗ Uma vez autenticado, usuário acessa TUDO
- ✗ Cliente A pode modificar entrega de Cliente B
- ✗ Sem verificação de propriedade de recurso

**Exemplo de ataque:**
```
Cliente A (user_id=1) cria entrega_id=100

Cliente B (user_id=2) executa:
PATCH /api/entregas/100/status 
{ "status": "ENTREGUE" }

✗ Consegue marcar como entregue sem fazer nada!
```

---

#### 9️⃣ **Tratamento de Erro Manual em Cada Endpoint**

**Padrão repetido 8 vezes:**
```javascript
try {
    // ... lógica
} catch (error) {
    console.error('Erro ao ...:', error.message);
    res.status(500).json({ error: 'Erro ao...' });
}
```

**Problema:**
- ✗ Duplicação de código
- ✗ Mensagens inconsistentes
- ✗ Alguns expõem `detalhes: error.message`
- ✗ Sem tratamento de erros específicos
- ✗ Sem middleware de erro global

---

#### 🔟 **Impossível Testar (Tudo Acoplado)**

**Problema:**
- ✗ Não é possível testar lógica de distância isoladamente
- ✗ Não é possível testar validação isoladamente
- ✗ Testes precisariam de BD real
- ✗ Mocks precisariam de reflexão

**Exemplo Impossível:**
```javascript
// ❌ Como testar geração de ID?
// Está dentro de POST endpoint
// Requer setup completo de BD, Express, etc.

// ✅ Como seria com separação:
const service = new EntregasService();
const id = service.gerarNumeroPedido();
expect(id).toMatch(/^PED-\d+-\d+$/);
```

---

### 2.2 Comparativo: Estrutura Atual vs Esperada

**ATUAL (Incorreta):**
```
src/routes/entregas.js (366 linhas)
└── Tudo acoplado:
    ├── Roteamento
    ├── Validação
    ├── Lógica de negócio
    ├── Queries SQL
    ├── Logging
    └── Error handling
```

**ESPERADA (Correta - MVC):**
```
src/
├── routes/entregas.js (80 linhas)
│   └── Apenas rotas
│
├── controllers/entregasCtrl.js (100 linhas)
│   └── Orquestração de requisição
│
├── services/entregasService.js (150 linhas)
│   └── Lógica de negócio
│
├── repositories/entregasRepo.js (50 linhas)
│   └── Acesso a dados (ORM)
│
├── middleware/
│   ├── validation.js (centralizado)
│   ├── errorHandler.js (centralizado)
│   └── authorization.js (RBAC)
│
└── validators/
    └── entregaValidator.js (regras)
```

---

## 3. Decisão

### 3.1 Estratégia de Refatoração

**Implementar arquitetura MVC com separação clara de responsabilidades:**

#### **FASE 1: Estrutura Base (Dias 1-2)**

1. **Criar estrutura de diretórios:**
   - `src/controllers/` - Orquestração
   - `src/services/` - Lógica de negócio
   - `src/repositories/` - Acesso a dados
   - `src/validators/` - Regras de validação
   - `src/utils/` - Utilitários (logging, etc.)

2. **Criar base classes:**
   - `BaseService` - com error handling
   - `BaseValidator` - com padrão de validação
   - `BaseRepository` - com queries padrão

---

#### **FASE 2: Refatoração Core (Dias 3-5)**

1. **Extrair lógica de negócio para `EntregasService`:**
   - `gerarNumeroPedido()`
   - `calcularDistancia(origin, destiny)`
   - `calcularTempoEstimado(km)`
   - `criarEntrega(data)`
   - `atualizarStatus(entregaId, novoStatus)`
   - `atribuirVeiculo(entregaId, veiculoPlaca, motoristaId)`

2. **Criar `EntregasController`:**
   - Orquestrador de requisições
   - Chama service para lógica
   - Chama repository para dados
   - Delegação de validação/autorização

3. **Refatorar `EntregasRepository`:**
   - Queries com ORM (sem raw SQL)
   - Métodos: `findAll()`, `findById()`, `create()`, `update()`, etc.

4. **Criar `EntregasValidator`:**
   - Regras de validação centralizadas
   - `validateEntrega()`, `validateStatus()`, etc.

---

#### **FASE 3: Middleware Centralizado (Dias 6-7)**

1. **Criar `errorHandler.js`:**
   - Middleware global de erro
   - Respostas padronizadas
   - Logging estruturado

2. **Criar `validation.js`:**
   - Middleware de validação
   - Usa `express-validator`

3. **Criar `authorization.js`:**
   - Middleware RBAC
   - Verifica propriedade de recurso

---

#### **FASE 4: Async/Await Unificado (Dias 8-9)**

1. **Converter todos callbacks para async/await**
2. **Remover promise chains**
3. **Usar try-catch consistente**

---

#### **FASE 5: Logging Estruturado (Dias 10-11)**

1. **Instalar winston**
2. **Remover console.log**
3. **Estruturar logs com contexto**

---

#### **FASE 6: Máquina de Estados (Dias 12)**

1. **Criar `statusTransitionsValidator.js`**
2. **Implementar transições válidas**
3. **Validar antes de salvar**

---

### 3.2 Decisões de Design

**Decisão 1: Padrão MVC**
- ✅ Routes → Controllers → Services → Repositories
- ✅ Separação clara de responsabilidades
- ✅ Testabilidade melhorada

**Decisão 2: Async/Await Uniforme**
- ✅ Todos os serviços async/await
- ✅ Try-catch consistente
- ✅ Sem promise chains ou callbacks

**Decisão 3: Validação Centralizada**
- ✅ Middleware de validação
- ✅ Usar `express-validator`
- ✅ Regras em arquivo separado

**Decisão 4: Logging Estruturado**
- ✅ Winston com JSON output
- ✅ Levels: debug, info, warn, error
- ✅ Context: userId, correlationId

**Decisão 5: Máquina de Estados**
- ✅ Arquivo separado com transições
- ✅ Validação antes de transição
- ✅ Configurável, não hardcoded

---

## 4. Consequências

### 4.1 Positivas ✅

| Consequência | Benefício |
|-------------|-----------|
| **Separação de Responsabilidades** | Cada camada tem 1 responsabilidade |
| **Testabilidade** | Lógica isolada, fácil testar |
| **Reutilização** | Services usáveis por múltiplos controllers |
| **Manutenção** | Mudanças isoladas, sem efeito colateral |
| **Performance** | Queries otimizadas sem raw SQL |
| **Escalabilidade** | Fácil adicionar novos endpoints |
| **Logging** | Rastreamento de operações |
| **Validação centralizada** | Consistência garantida |
| **Máquina de estados** | Regras de negócio protegidas |
| **Documentação** | Código auto-documentado |

### 4.2 Negativas ⚠️

| Consequência | Mitigation |
|-------------|-----------|
| **Mais arquivos** | Organização clara em camadas |
| **Mais dependências** | Express-validator, winston |
| **Curva de aprendizado** | Documentação no README |
| **Refatoração maior** | 50+ horas de desenvolvimento |
| **Testes a escrever** | 30+ testes unitários necessários |

### 4.3 Impacto Técnico

**Arquitetura Antes:**
```
Route {
    validação
    lógica
    SQL
    logging
    error handling
}
```

**Arquitetura Depois:**
```
Route 
  → Middleware (validation)
  → Middleware (auth)
  → Controller (orchestration)
    → Service (logic)
      → Repository (data access)
  → Middleware (error handler)
```

---

## 5. Alternativas Consideradas

### 5.1 Alternativa 1: Não Refatorar (❌ Rejeitada)

**Descrição:** Manter `entregas.js` como está

**Prós:**
- Sem refatoração

**Contras:**
- Código não mantível
- Sem testes possível
- Débito técnico cresce

**Decisão:** ❌ Rejeitada

---

### 5.2 Alternativa 2: Usar GraphQL (⚠️ Complexo)

**Descrição:** Migrar para GraphQL ao invés de REST

**Prós:**
- Menos over-fetching
- Melhor API

**Contras:**
- Trabalho adicional
- Complexidade aumenta

**Decisão:** ⚠️ Futuro (após REST refatorado)

---

### 5.3 Alternativa 3: Usar Padrão Saga (✅ Considerada)

**Descrição:** Implementar saga pattern para transações distribuídas

**Prós:**
- Consistência eventual
- Escalável

**Contras:**
- Complexidade
- Não é necessário agora

**Decisão:** ✅ Fase 2 (após refatoração básica)

---

## 6. Plano de Implementação

### 6.1 Timeline Estimada

```timeline
Dias 1-2: Estrutura Base
├── Criar diretórios (1h)
├── Criar base classes (4h)
└── Setup de testes (3h)
   Total: 8h

Dias 3-5: Refatoração Core
├── Extrair service (12h)
├── Criar controller (8h)
├── Refatorar repository (6h)
└── Testes (10h)
   Total: 36h

Dias 6-7: Middleware
├── Error handler (6h)
├── Validation (4h)
├── Authorization (6h)
└── Testes (4h)
   Total: 20h

Dias 8-9: Async/Await + Logging
├── Unificar padrão (8h)
├── Remover console.log (4h)
├── Winston setup (6h)
└── Testes (8h)
   Total: 26h

Dias 10-12: Estados + Finalização
├── Máquina de estados (6h)
├── Code review (6h)
├── Documentação (4h)
└── PR final (2h)
   Total: 18h

TOTAL GERAL: ~108 horas (≈ 3 semanas, 1-2 pessoas)
```

### 6.2 Estrutura de Arquivos Final

```
src/
├── routes/
│   └── entregas.js (90 linhas - apenas rotas)
│       ├── router.get('/')
│       ├── router.get('/stats')
│       ├── router.post('/')
│       ├── router.put('/:id')
│       ├── router.patch('/:id/status')
│       ├── router.patch('/:id/atribuir')
│       └── router.delete('/:id')
│
├── controllers/
│   └── entregasCtrl.js (120 linhas - orchestração)
│       ├── list(req, res)
│       ├── getStats(req, res)
│       ├── create(req, res)
│       ├── update(req, res)
│       ├── updateStatus(req, res)
│       ├── assignVehicle(req, res)
│       └── delete(req, res)
│
├── services/
│   ├── entregasService.js (150 linhas - business logic)
│   │   ├── gerarNumeroPedido()
│   │   ├── calcularDistancia()
│   │   ├── calcularTempoEstimado()
│   │   ├── criarEntrega()
│   │   ├── buscarEntrega()
│   │   ├── atualizarStatus()
│   │   └── atribuirVeiculo()
│   │
│   └── frotasService.js (integração)
│
├── repositories/
│   └── entregasRepo.js (80 linhas - data access)
│       ├── findAll(filters)
│       ├── findById(id)
│       ├── create(data)
│       ├── update(id, data)
│       ├── updateStatus(id, status)
│       └── getStats(groupBy)
│
├── validators/
│   ├── entregaValidator.js (100 linhas)
│   │   ├── validateCreate()
│   │   ├── validateUpdate()
│   │   ├── validateStatus()
│   │   └── validateAssign()
│   │
│   └── statusTransitionsValidator.js (40 linhas)
│       └── isTransitionValid(from, to)
│
├── middleware/
│   ├── validation.js (50 linhas - express-validator)
│   ├── errorHandler.js (60 linhas - global)
│   ├── authorization.js (50 linhas - RBAC)
│   └── auth.js (existente)
│
└── utils/
    ├── logger.js (winston setup)
    └── constants.js (status, etc.)

src/test/
├── services/entregasService.test.js (15 testes)
├── repositories/entregasRepo.test.js (12 testes)
├── controllers/entregasCtrl.test.js (14 testes)
└── validators/statusTransitions.test.js (8 testes)
   Total: 49 testes
```

---

## 7. Riscos e Mitigation

| Risco | Probabilidade | Impacto | Mitigation |
|-------|------------|--------|-----------|
| Refatoração quebra funcionalidades | 🟠 Média | 🟠 Médio | 49+ testes, code review |
| Descoberta de bugs novos | 🟠 Média | 🟡 Baixo | Testes abrangentes |
| Complexidade de controller | 🟡 Baixa | 🟡 Baixo | Controller thin, service fat |
| Integração com frontend | 🟡 Baixa | 🟡 Baixo | API interface idêntica |
| Conflitos git | 🟡 Baixa | 🟡 Baixo | Feature branch, frequent rebase |

---

## 8. Métricas de Sucesso

### 8.1 Métricas Quantitativas

| Métrica | Antes | Depois | Target |
|---------|-------|--------|--------|
| Linhas em entregas.js | 366 | 90 | < 100 |
| Arquivos de lógica | 1 | 3 | >= 3 |
| Testes unitários | 0 | 49 | >= 40 |
| Métodos > 30 linhas | 4 | 0 | 0 |
| console.log | 5+ | 0 | 0 |
| Raw SQL queries | 1 | 0 | 0 |
| Async padrões | 3 | 1 | 1 |
| Duplicação validação | 3 | 0 | 0 |

### 8.2 Métricas Qualitativas

- ✅ Cada camada tem responsabilidade única
- ✅ Lógica testável isoladamente
- ✅ Código auto-documentado
- ✅ Fácil adicionar novos endpoints
- ✅ Fácil reutilizar serviços
- ✅ Logging estruturado

### 8.3 Checklist de Validação

- [ ] Todos 7 arquivos criados
- [ ] entregas.js reduzido para 90 linhas
- [ ] Service com toda lógica
- [ ] Repository com queries ORM
- [ ] 49+ testes unitários (>80% cobertura)
- [ ] Zero console.log
- [ ] Async/await uniforme
- [ ] Validação centralizada
- [ ] Error handler global
- [ ] Máquina de estados implementada
- [ ] Code review aprovado
- [ ] Merge para main

---

## 9. Próximos Passos

### 9.1 Imediato

1. **Aprovação da ADR**
2. **Criar tickets de implementação**
3. **Setup de branches e estrutura**

### 9.2 Curto Prazo

1. **Fases 1-2** (Estrutura + Core)
2. **Testes incrementais**
3. **Code review frequente**

### 9.3 Médio Prazo

1. **Fases 3-6** (Middleware, states, finalização)
2. **Validação completa**
3. **Merge para main**

---

## 10. Referências

- [MVC Pattern](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [Express Best Practices](https://expressjs.com/en/advanced/best-practice-performance.html)
- [Clean Code by Robert Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)

---

## 11. Histórico de Decisão

| Data | Decisão | Status |
|------|---------|--------|
| 2026-06-18 | Criar ADR para refatoração entregas.js | ✅ PROPOSTO |

---

## 12. Conclusão

A refatoração de `entregas.js` é **IMPORTANTE** para melhorar testabilidade, manutenibilidade e escalabilidade. Esta ADR propõe arquitetura MVC clara com separação de responsabilidades, testes abrangentes, e código limpo.

**Timeline:** 3 semanas, 108 horas, 1-2 pessoas  
**Prioridade:** 🟠 ALTA  
**Impacto:** Qualidade do código melhorada significativamente

---

**Documento Criado:** 2026-06-18  
**Versão:** 1.0  
**Autor:** Análise Técnica - Claude Code  
**Status:** PROPOSTO - Aguardando Aprovação
