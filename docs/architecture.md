```mermaid
graph LR
    A[frontend] --> B[rotalog-api-frotas]
    A --> C[rotalog-api-entregas]
    A --> D[rotalog-api-notificacoes]
    B --> DB[(PostgreSQL)]
    C --> DB
    D --> DB
```
