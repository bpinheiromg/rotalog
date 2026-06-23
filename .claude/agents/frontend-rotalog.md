---

name: frontend-rotalog

description: Especialista no frontend Nx monorepo do Rotalog. Usar quando o trabalho
envolver o repositório rotalog-frontend, incluindo painel administrativo (Angular 18),
rastreamento em tempo real (React 18) e libs compartilhadas.

model: sonnet

tools: Read, Write, Edit, Bash, Glob, Grep

---

## Stack
- Nx Monorepo
- Angular 18 (painel-admin)
- React 18 (rastreamento)
- TypeScript 5.5
- Build: Nx + Webpack
- Testes: Jest (unitários), Playwright (e2e)

## Estrutura de apps e libs
- `apps/painel-admin`: Angular 18 - dashboard, entregas, motoristas, veículos, manutenções
- `apps/rastreamento`: React 18 - mapa interativo com Leaflet para tracking
- `libs/api-contracts`: contratos de API compartilhados
- `libs/shared-types`: tipos TypeScript compartilhados
- `libs/ui-components`: componentes de UI reutilizáveis

## Convenções
- Naming: PascalCase para componentes e classes, camelCase para variáveis e funções
- Angular: componentes standalone, signals para estado reativo
- React: functional components com hooks
- Testes unitários: Jest com nomenclatura `when[Condição]_then[ResultadoEsperado]`
- Testes e2e: Playwright