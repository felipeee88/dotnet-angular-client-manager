# Guideline — Backend (API .NET 8)

Este documento define as convenções de arquitetura, código e testes para a API do **dotnet-angular-client-manager**. Serve como referência ao implementar as entidades `Client`, `Contact` e `Opportunity`.

## Objetivo do sistema

CRM simples para cadastrar **Clientes**, seus **Contatos** e **Oportunidades comerciais** associadas. A API expõe endpoints REST autenticados por JWT e é consumida pelo frontend Angular.

## Stack

- **.NET 8** · **ASP.NET Core** WebApi
- **MediatR** (CQRS) · **AutoMapper** · **FluentValidation**
- **EF Core 8** · **SQL Server** (provider principal) com suporte opcional a **PostgreSQL**
- **JWT Bearer** para autenticação
- **xUnit** · **NSubstitute** · **Bogus** · **FluentAssertions** · **Testcontainers**

## Metodologias

- **DDD** — entidades e invariantes vivem no Domain, nunca no Controller ou Handler.
- **CQRS com MediatR** — Commands mutam estado; Queries leem. Handlers são finos e orquestram domínio + repositório.
- **Clean Architecture** — dependências apontam para dentro: `WebApi → Application → Domain`. `ORM` depende de `Domain` e é injetado via `IoC`.
- **SOLID** — em especial SRP (handlers com uma responsabilidade) e DIP (repositórios via interface no Domain, implementação em ORM).

## Estrutura de projetos

```
api/
├── ClientManager.sln
├── src/
│   ├── ClientManager.Domain/       (Entities, ValueObjects, Events, Repositories, Exceptions)
│   ├── ClientManager.Application/  (Commands, Queries, Handlers, Validators, DTOs)
│   ├── ClientManager.ORM/          (DbContext, Mappings, Repository impls, Migrations)
│   ├── ClientManager.WebApi/       (Controllers, Request/Response DTOs, Middleware, Program)
│   ├── ClientManager.IoC/          (ModuleInitializers — Application, Infra, WebApi)
│   └── ClientManager.Common/       (JWT, ValidationBehavior, Logging, Health checks)
└── tests/
    ├── ClientManager.Unit/         (aggregates, handlers, validators — sem infra)
    ├── ClientManager.Integration/  (repositórios contra DB efêmero via Testcontainers)
    └── ClientManager.Functional/   (WebApplicationFactory<Program> + TestAuthHandler)
```

Regras de dependência:

- **Domain** não depende de framework além de MediatR (para `INotification`).
- **Application** depende apenas de Domain + AutoMapper + FluentValidation + MediatR.
- **ORM** depende de Domain e wires EF Core.
- **IoC** compõe tudo e é consumido pela **WebApi**.
- **Common** hospeda utilitários transversais.

## Domínio

Agregados e entidades previstos:

- **Client** (raiz de agregado) — dados cadastrais do cliente, possui coleção de `Contact` e de `Opportunity`.
- **Contact** (entidade do agregado Client) — pessoa de contato vinculada a um cliente (nome, cargo, e-mail, telefone).
- **Opportunity** (entidade do agregado Client) — oportunidade comercial com estágio, valor estimado e data prevista de fechamento.

Invariantes devem ser reforçadas no construtor/métodos do agregado, lançando `DomainException`. Clientes nunca enviam valores derivados (por exemplo, totalizadores ou flags calculadas).

Eventos de domínio (`INotification`) são drenados do agregado e publicados **após** `SaveChangesAsync` bem-sucedido.

## Fluxo de requisição (CQRS)

```
HTTP request
    │
    ▼
Controller ── Mediator.Send(Command) ──▶ ValidationBehavior (FluentValidation)
                                             │
                                             ▼
                                         Handler
                                             │
                                             ▼
                                    Aggregate (regras de domínio)
                                             │
                                             ▼
                                    Repository (EF Core)
                                             │
                                             ▼
                                          Database
                                             │
                     ┌─── SaveChanges OK ─────┘
                     ▼
           drena DomainEvents ── Mediator.Publish ──▶ LoggingHandlers
                     │
                     ▼
              Response (AutoMapper) ──▶ HTTP 201/200
```

## Convenções de API

- Rotas no plural: `/api/Clients`, `/api/Clients/{id}/contacts`, `/api/Clients/{id}/opportunities`.
- Paginação, ordenação e filtros via query string: `_page`, `_size`, `_order`, mais filtros específicos do recurso.
- Respostas envolvidas em um envelope `{ data, success, message? }`.
- Todos os endpoints (exceto `/api/Auth` e `/api/Users` de bootstrap) exigem JWT Bearer.

## Tratamento de erros

Middleware global mapeia exceções conhecidas para um payload consistente `{ type, error, detail }`:

| Exceção | HTTP | `type` |
|---|---|---|
| `FluentValidation.ValidationException` | 400 | `ValidationError` |
| `NotFoundException` | 404 | `ResourceNotFound` |
| `DomainException` | 422 | `DomainError` |
| Outras | 500 | `InternalError` |

Stack traces de exceções não tratadas são logadas (Serilog) e **nunca** retornadas na resposta.

## Autenticação

- JWT Bearer em `Authorization: Bearer {token}`.
- Endpoint `POST /api/Auth` emite o token.
- Endpoint `POST /api/Users` para bootstrap de usuários; usuários autenticados com role adequada podem gerenciar os demais.

## Persistência

- EF Core 8 com mappings por entidade (`IEntityTypeConfiguration<T>`) no projeto ORM.
- Migrations vivem em `src/ClientManager.ORM/Migrations/`.
- Repositórios expõem apenas métodos que o Application precisa — sem `IQueryable` vazando para fora da camada ORM.
- Concurrency token: usar `RowVersion` (`byte[]`) em SQL Server; se usar PostgreSQL, trocar para `xmin` via shadow property.

## Testes

Três projetos, todos em xUnit:

- **Unit** — foco em agregados, validators e handlers. Infra é mockada com NSubstitute. Bogus para dados fake.
- **Integration** — `Repository` contra banco efêmero via Testcontainers (SQL Server ou PostgreSQL conforme provider escolhido). Exige Docker Desktop rodando.
- **Functional** — `WebApplicationFactory<Program>` com banco em container e `TestAuthHandler` que bypassa JWT para que endpoints `[Authorize]` sejam exercidos sem emitir tokens reais.

Asserções com FluentAssertions. Padrão AAA (Arrange/Act/Assert). Nome dos testes no formato `Given_When_Then` ou `Method_Scenario_Expectation`.

## Padrões de código

- Nullable reference types habilitado em todos os projetos.
- `async/await` em toda a pilha de I/O; sem `.Result` ou `.Wait()`.
- DTOs imutáveis (records) em Application e WebApi.
- Entidades de domínio com setters privados e construtores que validam invariantes.
- Sem lógica de negócio em Controllers — apenas orquestração via MediatR.
