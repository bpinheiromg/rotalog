# 📋 Análise Detalhada: `src/routes/entregas.js`

**Arquivo:** `rotalog-api-entregas/src/routes/entregas.js`  
**Tamanho:** 366 linhas  
**Status:** 🔴 CRÍTICO - Violação grave de separação de responsabilidades  

---

## Problema Principal

**Um arquivo de rotas NÃO deve conter:**
- ✗ Lógica de negócio (cálculos, algoritmos)
- ✗ Consultas SQL diretas
- ✗ Validação de dados complexa
- ✗ Logging manual
- ✗ Tratamento de erro manual

**Este arquivo contém TODOS os itens acima.**

---

## 📍 Mapa de Problemas por Linha

### 1. **Mistura de Padrões Async (Linhas 22-91)**

#### ❌ Padrão 1: Async/Await (Linhas 22-57)
```javascript
router.get('/', async (req, res) => {
    try {
        const entregas = await Entrega.findAll({...});
        res.json(entregas);
    } catch (error) {
        console.error('Erro ao listar entregas:', error.message);
        res.status(500).json({ error: 'Erro interno do servidor' });
    }
});
```

#### ❌ Padrão 2: Promise Chain (Linhas 65-91)
```javascript
router.get('/stats', function(req, res) {
    sequelize.query(query)
        .then(function(stats) { ... })
        .catch(function(error) { ... });
});
```

#### ❌ Padrão 3: Nested Promises (Linhas 336-363)
```javascript
Entrega.findByPk(req.params.id)
    .then(function(entrega) {
        return entrega.save()
            .then(function() {
                return Rastreamento.create({...})
                    .then(function() { ... });
            });
    })
    .catch(function(error) { ... });
```

**Problema:** Mesmo arquivo usa 3 estilos diferentes!

**Impacto:**
- Novo desenvolvedor confuso sobre qual padrão usar
- Difícil revisar código
- Tratamento de erro inconsistente
- Impossível automatizar linting

---

### 2. **Lógica de Negócio Misturada (Linhas 142-200)**

#### Exemplo 1: Geração de Número de Pedido
```javascript
// ❌ LÓGICA DE NEGÓCIO NA ROTA
const numeroPedido = 'PED-' + Date.now() + '-' + Math.floor(Math.random() * 1000);
```

**Deveria estar em:** `src/services/entregasService.js`

#### Exemplo 2: Cálculo de Distância
```javascript
// ❌ ALGORITMO NA ROTA
if (origem_lat && origem_lng && destino_lat && destino_lng) {
    const dLat = Math.abs(destino_lat - origem_lat);
    const dLng = Math.abs(destino_lng - origem_lng);
    distanciaKm = Math.sqrt(dLat * dLat + dLng * dLng) * 111;
}
```

**Problemas:**
- ✗ Fórmula matematicamente incorreta
- ✗ Hardcoded na rota
- ✗ Impossível testar isoladamente
- ✗ Impossível reutilizar em outro endpoint

#### Exemplo 3: Tempo Estimado
```javascript
// ❌ HEURÍSTICA NA ROTA
const tempoEstimado = distanciaKm ? Math.ceil(distanciaKm * 2) : null;
```

**Deveria estar em:** Serviço com lógica testável

---

### 3. **SQL Queries Diretas (Linhas 67-75)**

```javascript
// ❌ RAW SQL NA ROTA
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

**Problemas:**
- ✗ Raw SQL exposto
- ✗ Se houver filtros por query params, risco de SQL injection
- ✗ Duplicação se outro endpoint precisar
- ✗ Sem validação ou sanitização

**Deveria ser:**
```javascript
// ✅ ORM QUERIES EM SERVIÇO
const stats = await Entrega.aggregate(/* opciones */, {
    attributes: [[sequelize.fn('COUNT', ...), 'total']],
    group: ['status']
});
```

---

### 4. **Validação Inline (Linhas 147-150, 245-248, 296-298)**

#### Validação 1: Campos obrigatórios
```javascript
if (!origem_endereco || !destino_endereco) {
    return res.status(400).json({ error: 'Endereços são obrigatórios' });
}
```

#### Validação 2: Status válido
```javascript
const statusValidos = ['PENDENTE', 'ATRIBUIDA', 'EM_TRANSITO', 'ENTREGUE', 'CANCELADA'];
if (!statusValidos.includes(status)) {
    return res.status(400).json({ error: 'Status inválido' });
}
```

#### Validação 3: Campos obrigatórios
```javascript
if (!veiculo_placa || !motorista_id) {
    return res.status(400).json({ error: 'Placa...obrigatórios' });
}
```

**Problemas:**
- ✗ Duplicado em múltiplos endpoints
- ✗ Sem padronização de mensagem de erro
- ✗ Sem suporte a validações complexas
- ✗ Impossível reutilizar

**Solução:** Middleware de validação

```javascript
// ✅ MIDDLEWARE DE VALIDAÇÃO CENTRALIZADO
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

### 5. **Logging Manual com `console.log` (Múltiplas linhas)**

```javascript
// ❌ LOGGING NÃO ESTRUTURADO
console.log('Entrega criada:', numeroPedido);
console.log('Status atualizado:', entrega.numero_pedido, statusAnterior, '->', status);
console.log('Entrega atribuída:', entrega.numero_pedido, 'veículo:', veiculo_placa);
console.error('Erro ao listar entregas:', error.message);
```

**Problemas:**
- ✗ Sem JSON estruturado
- ✗ Sem níveis de severidade (info, warn, error)
- ✗ Sem correlationId para rastrear requisições
- ✗ Sem contexto de usuário
- ✗ Impossível buscar logs em ELK/CloudWatch

**Deveria ser:**
```javascript
// ✅ LOGGING ESTRUTURADO
logger.info('Entrega criada', {
    entregaId: entrega.id,
    numeroPedido,
    userId: req.user.id,
    timestamp: new Date().toISOString(),
    correlationId: req.headers['x-correlation-id']
});
```

---

### 6. **Sem Validação de Transição de Estado (Linhas 235-279)**

```javascript
// ❌ MÁQUINA DE ESTADO SEM LÓGICA
const statusValidos = ['PENDENTE', 'ATRIBUIDA', 'EM_TRANSITO', 'ENTREGUE', 'CANCELADA'];
if (!statusValidos.includes(status)) {
    return res.status(400).json({ error: 'Status inválido' });
}

// FIXME: Sem validação de transição de estado
const statusAnterior = entrega.status;
entrega.status = status;  // ← QUALQUER transição é permitida!
```

**Transições que deveriam ser BLOQUEADAS:**
- ENTREGUE → PENDENTE (impossível re-entregar)
- CANCELADA → ATRIBUIDA (não reativar)
- CANCELADA → EM_TRANSITO (não reativar)
- ENTREGUE → EM_TRANSITO (já entregue)

**Transições que devem ser permitidas:**
```
PENDENTE → ATRIBUIDA → EM_TRANSITO → ENTREGUE
PENDENTE → CANCELADA
ATRIBUIDA → CANCELADA
EM_TRANSITO → CANCELADA
```

**Solução:**
```javascript
// ✅ MÁQUINA DE ESTADO CENTRALIZADA
const statusTransitions = {
    'PENDENTE': ['ATRIBUIDA', 'CANCELADA'],
    'ATRIBUIDA': ['EM_TRANSITO', 'CANCELADA'],
    'EM_TRANSITO': ['ENTREGUE', 'CANCELADA'],
    'ENTREGUE': [],  // Nenhuma transição
    'CANCELADA': []  // Nenhuma transição
};

const statusAnterior = entrega.status;
if (!statusTransitions[statusAnterior]?.includes(status)) {
    return res.status(400).json({ 
        error: `Transição inválida: ${statusAnterior} → ${status}`,
        transicsoes_validas: statusTransitions[statusAnterior]
    });
}
```

---

### 7. **Sem Verificação de Autorização (Todos endpoints)**

```javascript
// ❌ QUALQUER USUÁRIO AUTENTICADO ACESSA QUALQUER ENTREGA
router.patch('/:id/status', async (req, res) => {
    const entrega = await Entrega.findByPk(req.params.id);
    // Deveria verificar:
    // if (entrega.cliente_id !== req.user.id && req.user.role !== 'admin') { ... }
});
```

**Risco:** Cliente A pode atualizar status de entrega de Cliente B

**Solução:**
```javascript
// ✅ MIDDLEWARE DE AUTORIZAÇÃO
async function verificarPropriedadeEntrega(req, res, next) {
    const entrega = await Entrega.findByPk(req.params.id);
    
    if (!entrega) {
        return res.status(404).json({ error: 'Entrega não encontrada' });
    }
    
    if (entrega.cliente_id !== req.user.id && req.user.role !== 'admin') {
        return res.status(403).json({ error: 'Acesso negado' });
    }
    
    req.entrega = entrega;
    next();
}

router.patch('/:id/status', verificarPropriedadeEntrega, async (req, res) => {
    // Usar req.entrega ao invés de buscar novamente
});
```

---

### 8. **Sem Verificação de Dependências Externas (Linhas 287-328)**

```javascript
// ❌ ATRIBUI VEÍCULO SEM VERIFICAR SE EXISTE
router.patch('/:id/atribuir', async (req, res) => {
    const { veiculo_placa, motorista_id } = req.body;
    
    // FIXME: Sem verificação se veículo existe no api-frotas
    // FIXME: Sem verificação se motorista existe no api-frotas
    // FIXME: Sem verificação de disponibilidade
    
    entrega.veiculo_placa = veiculo_placa;
    entrega.motorista_id = motorista_id;
});
```

**Problemas:**
- ✗ Atribui veículo que não existe
- ✗ Atribui motorista que não existe
- ✗ Sem verificação de disponibilidade (veículo já em uso)

**Deveria comunicar com api-frotas:**
```javascript
// ✅ VERIFICAR DEPENDÊNCIAS
const frotasService = require('../services/frotasService');

router.patch('/:id/atribuir', async (req, res) => {
    const { veiculo_placa, motorista_id } = req.body;
    
    // Verificar se veículo existe
    const veiculo = await frotasService.getVeiculo(veiculo_placa);
    if (!veiculo) {
        return res.status(400).json({ error: 'Veículo não encontrado' });
    }
    
    // Verificar se motorista existe
    const motorista = await frotasService.getMotorista(motorista_id);
    if (!motorista) {
        return res.status(400).json({ error: 'Motorista não encontrado' });
    }
    
    // Verificar disponibilidade
    if (!veiculo.disponivel) {
        return res.status(400).json({ error: 'Veículo não disponível' });
    }
});
```

---

### 9. **Estrutura de Resposta de Erro Inconsistente**

| Endpoint | Código | Response |
|----------|--------|----------|
| GET / | 500 | `{ error: 'Erro interno do servidor' }` |
| GET /stats | 500 | `{ error: 'Erro ao buscar estatísticas' }` |
| POST / | 500 | `{ error: 'Erro ao criar entrega', detalhes: error.message }` |
| PUT / | 500 | `{ error: 'Erro ao atualizar entrega' }` |

**Problemas:**
- ✗ Mensagem inconsistente
- ✗ Alguns expõem `detalhes`
- ✗ Sem HTTP status padronizado
- ✗ Sem error code/reference

**Deveria ser:**
```javascript
// ✅ RESPONSE PADRONIZADO
{
    error: {
        code: 'INTERNAL_SERVER_ERROR',
        message: 'Erro interno do servidor',
        reference: 'req-123456789',  // Para suporte
        timestamp: '2026-06-18T10:30:00Z'
    }
}
```

---

## 🏗️ Estrutura Atual (INCORRETA)

```
src/routes/entregas.js (366 linhas)
├── GET / (list)
├── GET /stats (stats com raw SQL)
├── GET /:id (getById)
├── GET /pedido/:numero (getByOrderNumber)
├── POST / (create com lógica de negócio)
├── PUT /:id (update)
├── PATCH /:id/status (update status com máquina de estados fake)
├── PATCH /:id/atribuir (assign vehicle)
└── DELETE /:id (soft delete)

+ Tudo misturado: validação, logging, SQL, lógica!
```

---

## ✅ Estrutura Esperada (CORRETA)

```
src/
├── routes/
│   └── entregas.js (80 linhas - APENAS rotas)
│       ├── router.get('/')
│       ├── router.get('/stats')
│       ├── router.post('/')
│       └── ... (chama controllers)
│
├── controllers/
│   └── entregasCtrl.js (100 linhas - orquestração)
│       ├── listEntregas(req, res)
│       ├── getStatistics(req, res)
│       ├── createEntrega(req, res)
│       └── ... (chama services)
│
├── services/
│   └── entregasService.js (150 linhas - lógica de negócio)
│       ├── gerarNumeroPedido()
│       ├── calcularDistancia()
│       ├── validarTransicaoStatus()
│       ├── criarEntrega()
│       └── ... (chama repositories)
│
├── repositories/
│   └── entregasRepo.js (50 linhas - acesso a dados)
│       ├── findAll()
│       ├── findById()
│       ├── create()
│       └── ... (queries)
│
├── middleware/
│   ├── validation.js (validação centralizada)
│   ├── auth.js (autenticação)
│   ├── authorization.js (autorização)
│   └── errorHandler.js (erro centralizado)
│
├── services/
│   ├── frotasService.js (comunicação com api-frotas)
│   └── notificacaoService.js
│
└── validators/
    ├── entregaValidator.js (regras de validação)
    └── statusTransitionsValidator.js (máquina de estados)
```

---

## 📋 Checklist de Refatoração

### Fase 1: Separação de Responsabilidades (Semana 1)
- [ ] Criar `src/controllers/entregasCtrl.js`
- [ ] Criar `src/services/entregasService.js`
- [ ] Criar `src/repositories/entregasRepo.js`
- [ ] Mover lógica de negócio para service
- [ ] Simplificar `routes/entregas.js`

### Fase 2: Validação Centralizada (Dias 2-3)
- [ ] Instalar `express-validator`
- [ ] Criar `src/validators/entregaValidator.js`
- [ ] Remover validação inline de rotas
- [ ] Implementar middleware de validação

### Fase 3: Autorização (Dia 4)
- [ ] Criar `src/middleware/authorization.js`
- [ ] Verificar propriedade de entrega
- [ ] Proteger endpoint `/atribuir` (admin only)

### Fase 4: Máquina de Estados (Dia 5)
- [ ] Criar `src/validators/statusTransitionsValidator.js`
- [ ] Implementar transições válidas
- [ ] Testar transições bloqueadas

### Fase 5: Logging Estruturado (Dia 6)
- [ ] Instalar winston
- [ ] Remover `console.log`
- [ ] Adicionar correlationId

### Fase 6: Tratamento de Erro (Dia 7)
- [ ] Criar error handler centralizado
- [ ] Remover try-catch manual
- [ ] Padronizar responses

---

## 📊 Comparativo: Antes vs Depois

### ANTES (Atual)
```
entregas.js: 366 linhas
- Mix de callbacks + async/await
- SQL direto
- Lógica de negócio
- Validação inline
- Logging manual
- Sem autorização
- Sem teste possível
```

### DEPOIS (Refatorado)
```
routes/entregas.js: ~80 linhas (rotas apenas)
controllers/entregasCtrl.js: ~100 linhas (orquestração)
services/entregasService.js: ~150 linhas (lógica testável)
repositories/entregasRepo.js: ~50 linhas (ORM apenas)
middleware/validation.js: ~80 linhas (validação centralizada)
middleware/authorization.js: ~40 linhas (autorização)

Total: ~500 linhas (mas bem organizadas e testáveis)
```

**Benefícios:**
- ✅ Cada arquivo com responsabilidade única
- ✅ Fácil de testar isoladamente
- ✅ Fácil de reutilizar serviços
- ✅ Fácil de manter e estender
- ✅ Novo desenvolvedor entende estrutura

---

**Documentação gerada:** 2026-06-18
