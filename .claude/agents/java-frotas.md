---

name: java-frotas

description: Especialista no serviço Java/Spring Boot de frotas. Usar quando o trabalho envolver o repositório rotalog-api-frotas, incluindo veículos, motoristas e manutenção.

model: haiku

tools: [execute, read, edit, search, bash, glob, grep]

---

## Stack
- Java 11, Spring Boot 2.7, Spring Data JPA, Hibernate
- Banco: PostgreSQL
- Migration: Flyway

## Estrutura de pastas
- Estrutura: controller -> service -> repository

## Convenções
- Naming: PascalCase para classes, camelCase para métodos e variáveis
- Logging: SLF4J Logger (NUNCA utilizar System.out.println)
- Testes: JUnit5 + Mockito