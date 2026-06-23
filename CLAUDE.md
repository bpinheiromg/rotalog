# Rotalog

## Visão Geral

Sistema de gestão de frotas e entregas. Gerencia veículos, motoristas, entregas e notificações. Composto por 3 APIs de backend e 1 frontend (monorepo NX).

## Repositórios

### rotalog-api-frotas (Java/Spring Boot 2.7)

- **Responsabilidade**: Pedidos, roteamento, tracking
- **Banco**: PostgreSQL
- **Migrations**: Flyway
- **Estrutura**: controller → service → repository
- **Entidades**: JPA em `model/`
- **Configuração**: Maven (pom.xml)
- **Java**: 11
- **Spring Boot**: 2.7.14
- **Build**: `mvn clean install` / `mvn spring-boot:run`

### rotalog-api-entregas (Node.js/Express)

- **Responsabilidade**: Gestão de entregas, despacho, notificações
- **Banco**: PostgreSQL
- **ORM**: Sequelize
- **Estrutura**: routes → services → models
- **Configuração**: npm
- **Scripts**: `npm start`, `npm run dev` (nodemon)
- **Migrations**: SQL scripts em `src/config/`
- **Seed**: Dados iniciais em `src/config/seed.sql`

### rotalog-frontend (Monorepo NX - Angular + React)

- **Tipo**: Nx workspace com múltiplas aplicações
- **Frontend 1**: Angular 18
- **Frontend 2**: React 18
- **Ferramenta Build**: Nx 19.8.4
- **Estrutura**: `packages/*` (workspaces)
- **Testes**: Jest com Playwright E2E
- **Linting**: ESLint + Prettier
- **Bibliotecas**: Leaflet, React-Leaflet para mapas
- **Comandos Nx**: `nx run`, `nx run-many`, `nx affected`

## Padrões Gerais

### Estrutura de Projetos

**Backend Java:**
```
src/main/java/com/rotalog/
├── controller/
├── service/
├── repository/
└── model/
```

**Backend Node.js:**
```
src/
├── routes/
├── services/
├── models/
├── middleware/
└── config/
```

### Banco de Dados

- Todas as APIs usam **PostgreSQL**
- Credenciais padrão: usuário `rotalog_admin`
- Migrations automáticas (Flyway para Java, SQL scripts para Node)

### Comandos Comuns

**API Frotas:**
```bash
cd rotalog-api-frotas
mvn clean install
mvn spring-boot:run
```

**API Entregas:**
```bash
cd rotalog-api-entregas
npm install
npm run dev      # desenvolvimento
npm start        # produção
npm run db:migrate  # aplicar migrations
npm run db:seed     # dados iniciais
```

**Frontend:**
```bash
cd rotalog-frontend
npm install
nx run <app>:serve     # servir aplicação
nx run-many --target=serve  # todas as apps
nx affected --target=test    # testes em arquivos modificados
```

## Notas

- Todos os repositórios estão em um **monorepo dentro de um workspace**
- As APIs de backend são independentes e podem rodar em paralelo
- O frontend é organizado como monorepo NX para compartilhamento de código