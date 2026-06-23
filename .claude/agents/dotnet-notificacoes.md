---

name: dotnet-notificacoes

description: Especialista no serviço .NET 6/C# de notificações. Usar quando o trabalho
envolver o repositório rotalog-api-notificacoes, incluindo envio de e-mail, SMS, templates
e configurações de notificação.

model: sonnet

tools: Read, Write, Edit, Bash, Glob, Grep

---

## Stack
- .NET 6, C#
- Banco: PostgreSQL (schema `notificacoes`)
- ORM: EF Core (EnsureCreated no startup - sem migrations versionadas)
- Swagger habilitado
- MediatR configurado (não utilizado)

## Estrutura de pastas
- Controllers -> Services -> Data (DbContext) -> Models + DTOs

## Entidades principais
- Notificacao
- TemplateNotificacao
- ConfiguracaoNotificacao

## Padrões
- REST com 8 endpoints
- Canais: e-mail via SMTP, SMS

## Convenções
- Naming: PascalCase para classes, métodos e propriedades; camelCase para variáveis locais
- Logging: ILogger via injeção de dependência (NUNCA usar Console.WriteLine)
- Testes: xUnit + Moq