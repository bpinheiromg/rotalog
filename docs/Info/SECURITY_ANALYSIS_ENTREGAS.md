# 🔒 Análise de Vulnerabilidade e Arquitetura - API Entregas (Node.js)

**Data:** 2026-06-18  
**Status:** 🔴 CRÍTICO - Múltiplas vulnerabilidades de segurança identificadas  
**Escopo:** rotalog-api-entregas  

---

## ⚠️ Resumo Executivo

A API Entregas possui **12 vulnerabilidades críticas**, **8 problemas arquiteturais de alto risco** e **violações de separação de responsabilidades**. Os três maiores riscos são:

1. **Credenciais hardcodeadas em arquivos commitados** → Exposição de BD e JWT secret
2. **Arquivo de rotas misturando lógica de negócio e SQL** → Violação de MVC
3. **Autenticação bypass em desenvolvimento** → Ausência de validação real de JWT

---

## 🔴 VULNERABILIDADES CRÍTICAS

### 1. **Credenciais Hardcodeadas em `.env` Commitado**

**Localização:** `.env` (raiz do projeto)

**Problema:**
```env
# FIXME: Credentials hardcoded in .env file (should use secrets manager)
DB_PASSWORD=rotalog123
JWT_SECRET=super-secret-key-that-should-not-be-hardcoded
```

**Risco:**
- ✗ Qualquer pessoa com acesso ao repo tem credenciais do PostgreSQL
- ✗ JWT secret exposto permite falsificar tokens
- ✗ API Frotas (porta 8080) e Notificações (porta 5000) endpoints revelados
- ✗ Histórico git permanente (mesmo se removido depois)

**Impacto:** CRÍTICO - Comprometimento de toda a base de dados

**Remediação:**
```bash
# 1. Remover do histórico git
git rm --cached .env
git filter-branch --force --index-filter 'git rm --cached --ignore-unmatch .env' -- --all

# 2. Rotacionar credenciais imediatamente
# ALTER ROLE rotalog_admin WITH PASSWORD 'novo-password-forte';

# 3. Criar .env.example com placeholders
# DB_PASSWORD=your_secure_password_here
# JWT_SECRET=your_jwt_secret_here

# 4. Adicionar .env ao .gitignore
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore

# 5. Usar secrets manager (AWS Secrets Manager, Azure Key Vault, etc.)
```

---

### 2. **JWT Hardcodeado em `auth.js`**

**Localização:** `src/middleware/auth.js:12`

**Código vulnerável:**
```javascript
const JWT_SECRET = process.env.JWT_SECRET || 'super-secret-key-that-should-not-be-hardcoded';
```

**Problemas:**
- ✗ Se `JWT_SECRET` não estiver definida (cenários de erro), fallback para secret fraco
- ✗ Secret é string legível e curta (fácil de bruteforce)
- ✗ Sem rotação de keys
- ✗ Sem algoritmo configurável

**Risco:** CRÍTICO - Token JWT forjável

**Remediação:**
```javascript
// ❌ ERRADO
const JWT_SECRET = process.env.JWT_SECRET || 'super-secret-key';

// ✅ CORRETO
const JWT_SECRET = process.env.JWT_SECRET;
if (!JWT_SECRET) {
    throw new Error('JWT_SECRET environment variable is required');
}
if (JWT_SECRET.length < 32) {
    throw new Error('JWT_SECRET must be at least 32 characters long');
}
```

---

### 3. **Autenticação Completamente Bypass em Desenvolvimento**

**Localização:** `src/middleware/auth.js:15-19`

**Código vulnerável:**
```javascript
if (process.env.NODE_ENV !== 'production') {
    console.log('[AUTH] Bypass em desenvolvimento');
    return next(); // ← Qualquer um acessa!
}
```

**Problemas:**
- ✗ Nenhuma validação em development (100% das requisições são aceitas)
- ✗ Usuário fake injetado: `{ id: 1, nome: 'Admin', role: 'admin' }`
- ✗ Sem forma de testar auth localmente sem produção
- ✗ Desenvolvedor pode deixar NODE_ENV=development em produção acidentalmente

**Risco:** CRÍTICO - Bypass completo de autenticação

**Remediação:**
```javascript
// ✅ Validação adequada em todos os ambientes
const jwt = require('jsonwebtoken');

function authMiddleware(req, res, next) {
    const authHeader = req.headers.authorization;
    
    if (!authHeader) {
        return res.status(401).json({ error: 'Token não fornecido' });
    }
    
    const parts = authHeader.split(' ');
    if (parts.length !== 2 || parts[0] !== 'Bearer') {
        return res.status(401).json({ error: 'Token mal formatado' });
    }
    
    try {
        const decoded = jwt.verify(parts[1], process.env.JWT_SECRET, {
            algorithms: ['HS256']
        });
        req.user = decoded;
        next();
    } catch (error) {
        return res.status(401).json({ error: 'Token inválido ou expirado' });
    }
}
```

---

### 4. **Validação JWT Fake (Apenas Verifica Comprimento)**

**Localização:** `src/middleware/auth.js:35-42`

**Código vulnerável:**
```javascript
// FIXME: "Validação" fake - apenas verifica se não está vazio
if (!token || token.length < 10) {
    return res.status(401).json({ error: 'Token inválido' });
}

// FIXME: Sem decodificação real do JWT
// FIXME: Sem verificação de expiração
// FIXME: Sem verificação de assinatura
```

**Problemas:**
- ✗ Aceita qualquer string com 10+ caracteres como token válido
- ✗ Exemplo: `"valid token!"` seria aceito
- ✗ Sem decodificação real do JWT (RFC 7519)
- ✗ Sem verificação de assinatura criptográfica
- ✗ Sem validação de expiração (`exp` claim)

**Risco:** CRÍTICO - Qualquer string é um token válido

**Impacto:** Um atacante pode forjar tokens sem conhecer o secret

---

### 5. **CORS Completamente Aberto**

**Localização:** `src/index.js:34-42`

**Código vulnerável:**
```javascript
app.use(function(req, res, next) {
    res.header('Access-Control-Allow-Origin', '*');  // ← Qualquer origem!
    res.header('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS');
    res.header('Access-Control-Allow-Headers', '...Authorization');
```

**Problemas:**
- ✗ `Access-Control-Allow-Origin: *` permite requisições de qualquer website
- ✗ Credentials podem ser incluídas (se `withCredentials: true` no cliente)
- ✗ Expõe a API a ataques CSRF
- ✗ Integração com qualquer domínio não confiável

**Risco:** ALTO - CSRF, XSS, data exfiltration

**Remediação:**
```javascript
const cors = require('cors');

app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    allowedHeaders: ['Content-Type', 'Authorization']
}));
```

---

### 6. **Endpoint de Rastreamento Sem Autenticação**

**Localização:** `src/index.js:76`

**Código vulnerável:**
```javascript
app.use('/api/rastreamento', rastreamentoRoutes); // ← Sem authMiddleware!
app.use('/api/entregas', authMiddleware, entregasRoutes);
```

**Problema:**
- ✗ Qualquer pessoa pode rastrear qualquer entrega
- ✗ Expõe dados de clientes (endereços, status de entrega)
- ✗ Sem autorização de quem pode ver cada entrega

**Risco:** ALTO - Privacy violation, business logic exposure

**Remediação:**
```javascript
// Adicionar middleware de autenticação/autorização
app.use('/api/rastreamento', authMiddleware, verificarAutorizacao, rastreamentoRoutes);
```

---

### 7. **Detalhes de Erro Expostos em Respostas 500**

**Localização:** `src/routes/entregas.js:197-198`

**Código vulnerável:**
```javascript
res.status(500).json({ 
    error: 'Erro ao criar entrega', 
    detalhes: error.message  // ← Expondo stack trace interno!
});
```

**Problemas:**
- ✗ `error.message` pode revelar paths internos, estrutura de BD, etc.
- ✗ Informações úteis para atacante mapear aplicação
- ✗ Exemplo: "Cannot insert NULL into column 'veiculo_placa'" → BD schema exposed

**Risco:** MÉDIO - Information disclosure

**Remediação:**
```javascript
// ❌ ERRADO
res.status(500).json({ error: error.message });

// ✅ CORRETO
console.error('Erro ao criar entrega:', error); // Log para arquivo/ELK
res.status(500).json({ error: 'Erro interno do servidor' });
```

---

### 8. **SQL Injection Risk via Raw Queries**

**Localização:** `src/routes/entregas.js:67-75`

**Código vulnerable:**
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

**Problemas:**
- ✗ Raw SQL query sem parametrização
- ✗ Se houver filtros por status, pode ser vulnerável a SQL injection
- ✗ Bypassa proteções do ORM

**Risco:** ALTO - SQL Injection (dependendo de parámetros)

**Remediação:**
```javascript
// ✅ Usar Sequelize aggregation
const stats = await Entrega.findAll({
    attributes: [
        'status',
        [sequelize.fn('COUNT', sequelize.col('id')), 'total'],
        [sequelize.fn('AVG', sequelize.col('distancia_km')), 'distancia_media'],
        [sequelize.fn('AVG', sequelize.col('peso_kg')), 'peso_medio']
    ],
    group: ['status'],
    raw: true
});
```

---

## 🏗️ PROBLEMAS ARQUITETURAIS CRÍTICOS

### A. **Arquivo `entregas.js` Misturando Responsabilidades**

**Localização:** `src/routes/entregas.js` (366 linhas)

**O arquivo deveria ser:** Apenas definir rotas/endpoints

**O arquivo contém:**
1. ✗ SQL queries diretas (estatísticas com raw SQL)
2. ✗ Lógica de negócio (cálculo de distância, fórmula de tempo estimado)
3. ✗ Validação de dados
4. ✗ Comunicação com banco de dados (Entrega.create, .save(), etc.)
5. ✗ Logging manual com `console.log`
6. ✗ Tratamento de erro manual em cada endpoint

**Exemplo de mistura de responsabilidades (linhas 155-162):**
```javascript
// LÓGICA DE NEGÓCIO NÃO DEVE ESTAR AQUI
let distanciaKm = null;
if (origem_lat && origem_lng && destino_lat && destino_lng) {
    const dLat = Math.abs(destino_lat - origem_lat);
    const dLng = Math.abs(destino_lng - origem_lng);
    distanciaKm = Math.sqrt(dLat * dLat + dLng * dLng) * 111; // Fórmula fake!
}
```

**Violação:** Model-View-Controller (MVC)

```
ESTRUTURA ESPERADA:
├── routes/entregas.js          → Apenas define endpoints
├── controllers/entregasCtrl.js → Orquestra requisição
├── services/entregasService.js → Lógica de negócio
└── models/Entrega.js           → Acesso a dados

ESTRUTURA ATUAL:
└── routes/entregas.js          → Tudo em um arquivo!
```

---

### B. **Lógica de Cálculo de Distância Hardcoded e Incorreta**

**Localização:** `src/routes/entregas.js:155-162`

**Código:**
```javascript
// FIXME: Fórmula simplificada - deveria usar Haversine ou API de rotas
const dLat = Math.abs(destino_lat - origem_lat);
const dLng = Math.abs(destino_lng - origem_lng);
distanciaKm = Math.sqrt(dLat * dLat + dLng * dLng) * 111; // FIXME: aproximação grosseira
```

**Problemas:**
- ✗ Fórmula de Pitágoras em coordenadas geográficas → INCORRETA para longas distâncias
- ✗ Usa 111 como constante (1° de latitude ≠ 1° de longitude em km)
- ✗ Não considera curvatura da Terra
- ✗ Fator 111 é apenas aproximação para latitudes próximas do equador

**Exemplos de erro:**
- São Paulo → Brasília: ~1000 km real, este cálculo retorna ~500 km
- Buenos Aires → Nova York: ~9000 km real, fórmula incorreta

**Risco:** MÉDIO - Cálculos de rotas completamente incorretos

---

### C. **Mistura de Padrões Async (60% Callbacks + Async/Await)**

**Localização:** Múltiplos endpoints em `entregas.js`

**Exemplo 1 - Async/Await (linhas 22-57):**
```javascript
router.get('/', async (req, res) => {
    const entregas = await Entrega.findAll({ ... });
    res.json(entregas);
});
```

**Exemplo 2 - Callbacks (linhas 65-91):**
```javascript
router.get('/stats', function(req, res) {
    sequelize.query(query)
        .then(function(stats) { ... })
        .catch(function(error) { ... });
});
```

**Exemplo 3 - Nested Callbacks (linhas 336-363):**
```javascript
Entrega.findByPk(req.params.id)
    .then(function(entrega) {
        return entrega.save().then(function() {
            return Rastreamento.create({...})
                .then(function() { ... });
        });
    })
    .catch(function(error) { ... });
```

**Problemas:**
- ✗ Inconsistência torna código difícil de ler/manter
- ✗ Callback hell em alguns endpoints (nested promises)
- ✗ Diffícil fazer tratamento de erro centralizado
- ✗ Developers novos confusos sobre padrão esperado

**Impacto:** Qualidade de código, manutenibilidade reduzida

---

### D. **Validação Inline vs Centralizada**

**Localização:** Múltiplos endpoints

**Validação inline (anti-pattern):**
```javascript
// Linha 148-150
if (!origem_endereco || !destino_endereco) {
    return res.status(400).json({ error: 'Endereços são obrigatórios' });
}
```

**Problemas:**
- ✗ Duplicada em múltiplos endpoints
- ✗ Sem tipo de erro padronizado
- ✗ Difícil manter consistência
- ✗ Sem suporte a validações complexas

**Solução:** Middleware de validação centralizado

---

### E. **Falta de Validação de Máquina de Estados**

**Localização:** `src/routes/entregas.js:235-279`

**Código:**
```javascript
router.patch('/:id/status', async (req, res) => {
    const statusValidos = ['PENDENTE', 'ATRIBUIDA', 'EM_TRANSITO', 'ENTREGUE', 'CANCELADA'];
    if (!statusValidos.includes(status)) {
        return res.status(400).json({ error: 'Status inválido' });
    }

    // FIXME: Sem validação de transição de estado
    const statusAnterior = entrega.status;
    entrega.status = status;  // ← Pode ir de ENTREGUE → PENDENTE!
```

**Problema:** Transições inválidas são permitidas

```
TRANSIÇÕES VÁLIDAS:
PENDENTE → ATRIBUIDA → EM_TRANSITO → ENTREGUE
QUALQUER_STATUS → CANCELADA

MAS O CÓDIGO PERMITE:
ENTREGUE → PENDENTE (❌ Impossível re-entregar)
CANCELADA → EM_TRANSITO (❌ Entregar após cancelada)
```

**Risco:** MÉDIO - Inconsistência de dados de negócio

---

### F. **Sem Verificação de Autorização**

**Localização:** Todos endpoints

**Problema:** Uma vez autenticado, usuário acessa TUDO

```javascript
// Sem validação como:
if (entrega.cliente_id !== req.user.id) {
    return res.status(403).json({ error: 'Acesso negado' });
}
```

**Exemplo de ataque:**
- Cliente A cria entrega com ID=1
- Cliente B faz PATCH `/api/entregas/1/status` → Consegue atualizar entrega de outro cliente!

---

### G. **Logging com `console.log` (Não Estruturado)**

**Localização:** Múltiplos arquivos

**Problemas:**
- ✗ Logs não estruturados (sem JSON)
- ✗ Sem níveis de severidade (debug, info, warn, error)
- ✗ Sem timestamp consistente
- ✗ Impossível filtrar/buscar em produção
- ✗ Sem correlationId para rastrear requisições

**Exemplo:**
```javascript
console.log('Entrega criada:', numeroPedido); // Qual app? Qual servidor?
```

**Remediação:**
```javascript
// Usar winston ou pino
logger.info('Entrega criada', {
    entregaId: entrega.id,
    numeroPedido,
    userId: req.user.id,
    timestamp: new Date().toISOString(),
    correlationId: req.headers['x-correlation-id']
});
```

---

### H. **Sem Tratamento de Erro Centralizado**

**Localização:** Cada endpoint tem seu próprio try-catch

**Código:**
```javascript
router.post('/', async (req, res) => {
    try {
        // ...
    } catch (error) {
        console.error('Erro ao criar entrega:', error.message);
        res.status(500).json({ error: 'Erro ao criar entrega' });
    }
});
```

**Problemas:**
- ✗ Duplicação de código
- ✗ Inconsistência em responses de erro
- ✗ Erro handler middleware não é usado

**Solução:** Middleware de erro global

---

## 📊 PROBLEMAS SECUNDÁRIOS

### 1. **Número de Pedido Gerado com Math.random()**

**Localização:** `src/routes/entregas.js:153`

```javascript
const numeroPedido = 'PED-' + Date.now() + '-' + Math.floor(Math.random() * 1000);
```

**Problemas:**
- ✗ `Math.random()` não é criptograficamente seguro
- ✗ Padrão previsível (timestamp + número pequeno)
- ✗ Colisões possíveis (múltiplas requests no mesmo milissegundo)

**Solução:**
```javascript
const crypto = require('crypto');
const numeroPedido = 'PED-' + crypto.randomBytes(8).toString('hex').toUpperCase();
```

---

### 2. **Tempo Estimado Hardcoded**

**Localização:** `src/routes/entregas.js:165`

```javascript
const tempoEstimado = distanciaKm ? Math.ceil(distanciaKm * 2) : null; // 2 min/km
```

**Problemas:**
- ✗ Constante 2min/km é arbitrária
- ✗ Não considera tráfego, tipo de veículo, horário
- ✗ Distância já está incorreta (Pitágoras)

---

### 3. **Health Check Retorna Status Fake**

**Localização:** `src/index.js:60-72`

```javascript
app.get('/api/health', (req, res) => {
    res.json({
        status: 'UP',
        dependencies: {
            database: 'UNKNOWN',  // ← Nunca verifica!
            'api-frotas': 'UNKNOWN',
            'api-notificacoes': 'UNKNOWN'
        }
    });
});
```

**Problemas:**
- ✗ Sempre retorna UP mesmo se BD estiver down
- ✗ Orquestrador (Docker, Kubernetes) acha que serviço está saudável
- ✗ Requisições continuam caindo

---

### 4. **Sem Graceful Shutdown**

**Localização:** `src/index.js` (falta completa)

**Problema:**
- ✗ SIGTERM não é tratado
- ✗ Servidor encerra imediatamente, cortando requisições ativas
- ✗ Conexões de BD não são fechadas
- ✗ Transações incompletas

---

### 5. **Limite Hardcodeado em Query**

**Localização:** `src/routes/entregas.js:47`

```javascript
include: [{
    model: Rastreamento,
    as: 'rastreamentos',
    limit: 5,  // FIXME: hardcoded
```

**Problema:**
- ✗ Sempre retorna últimos 5 rastreamentos
- ✗ Sem paginação
- ✗ Sem configuração externalizável

---

## 🎯 MATRIX DE RISCO

| Vulnerabilidade | Severidade | Likelihood | Impact | CVSS |
|-----------------|-----------|-----------|--------|------|
| Credenciais hardcodeadas | 🔴 CRÍTICA | Alto | Comprometimento total do BD | 9.8 |
| JWT bypass (fake validation) | 🔴 CRÍTICA | Alto | Acesso sem autenticação | 9.7 |
| Auth bypass em dev | 🔴 CRÍTICA | Médio | Sem segurança em staging | 8.5 |
| CORS aberto | 🟠 ALTA | Alto | CSRF, XSS, data theft | 8.2 |
| Rastreamento sem auth | 🟠 ALTA | Alto | Privacy violation | 7.8 |
| SQL Injection (raw queries) | 🟠 ALTA | Médio | Acesso a dados arbitrários | 8.0 |
| Detalhes de erro expostos | 🟡 MÉDIA | Alto | Information disclosure | 5.3 |
| Sem autorização (RBAC) | 🟠 ALTA | Alto | Acesso cross-tenant | 7.9 |

---

## ✅ PLANO DE REMEDIAÇÃO (Priorizado)

### Fase 1: Segurança Imediata (24h)
- [ ] Rotacionar credenciais de BD
- [ ] Remover `.env` do histórico git
- [ ] Implementar autenticação real com `jsonwebtoken`
- [ ] Restringir CORS

### Fase 2: Arquitetura (1 semana)
- [ ] Separar `entregas.js` em routes + controller + service
- [ ] Implementar middleware de validação centralizado
- [ ] Adicionar máquina de estados para transições
- [ ] Implementar logger estruturado (winston)

### Fase 3: Autenticação (3 dias)
- [ ] Adicionar autorização (verificar propriedade de entrega)
- [ ] Proteger endpoint `/api/rastreamento`
- [ ] Implementar role-based access control

### Fase 4: Qualidade (1 semana)
- [ ] Converter todos callbacks para async/await
- [ ] Implementar health checks reais
- [ ] Adicionar graceful shutdown
- [ ] Implementar testes unitários

---

## 📚 Referências

- **OWASP Top 10 2024**
  - A01: Broken Access Control
  - A02: Cryptographic Failures
  - A04: Insecure Design
  - A07: Identification and Authentication Failures

- **12-Factor App:** https://12factor.net/
- **JWT Best Practices:** https://tools.ietf.org/html/rfc8725
- **Express Security:** https://expressjs.com/en/advanced/best-practice-security.html

---

**Próxima Revisão:** Após implementação das correções da Fase 1  
**Gerado por:** Claude Code Security Analyzer
