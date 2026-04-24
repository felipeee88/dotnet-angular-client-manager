# dotnet-angular-client-manager

CRM simples para cadastrar **Clientes**, seus **Contatos** e **Oportunidades comerciais**. API REST em **.NET 8** seguindo **DDD + CQRS + Clean Architecture + SOLID**, consumida por um frontend **Angular** com **Angular Material** e **Reactive Forms**.

## Stack

### Backend

- **.NET 8** · **ASP.NET Core** WebApi
- **MediatR** (CQRS) · **AutoMapper** · **FluentValidation**
- **EF Core 8** · **SQL Server** (provider principal) com suporte a **PostgreSQL**
- **JWT Bearer** para autenticação
- **xUnit** · **NSubstitute** · **Bogus** · **FluentAssertions** · **Testcontainers**
- **Serilog** para logging estruturado

### Frontend

- **Angular** (última versão LTS, standalone components)
- **Angular Material** — biblioteca de componentes
- **Reactive Forms** — formulários tipados
- **RxJS** · **TypeScript strict**
- **Jasmine/Karma** para testes unitários

## Pré-requisitos

- **.NET 8 SDK**
- **Node.js 20+** e **npm** (ou pnpm) para o frontend
- **Docker Desktop** — recomendado para SQL Server/PostgreSQL em dev e obrigatório para os testes Integration/Functional (Testcontainers)
- **Angular CLI** (`npm i -g @angular/cli`)
- **dotnet-ef** tool:

  ```bash
  dotnet tool install --global dotnet-ef
  ```

## Estrutura do repositório

```
dotnet-angular-client-manager/
├── guideline/
│   ├── backend.md       (convenções da API — arquitetura, padrões, testes)
│   └── frontend.md      (convenções do Angular — estrutura, UI, forms)
├── api/                 (solution .NET 8)
│   ├── src/
│   │   ├── ClientManager.Domain/
│   │   ├── ClientManager.Application/
│   │   ├── ClientManager.ORM/
│   │   ├── ClientManager.WebApi/
│   │   ├── ClientManager.IoC/
│   │   └── ClientManager.Common/
│   └── tests/
│       ├── ClientManager.Unit/
│       ├── ClientManager.Integration/
│       └── ClientManager.Functional/
└── front/               (app Angular)
    └── src/app/
        ├── core/        (serviços singletons, interceptors, guards)
        ├── shared/      (componentes, pipes e diretivas reutilizáveis)
        ├── layout/      (shell: header, sidenav, footer)
        └── features/
            ├── clients/
            ├── contacts/
            └── opportunities/
```

Guidelines detalhados em [guideline/backend.md](guideline/backend.md) e [guideline/frontend.md](guideline/frontend.md).

## Executando o backend

A partir da raiz do repositório:

```bash
# 1. Subir SQL Server (ou Postgres) local via docker-compose
docker-compose up -d clientmanager.database

# 2. Aplicar migrations
dotnet ef database update \
  --project api/src/ClientManager.ORM \
  --startup-project api/src/ClientManager.WebApi

# 3. Rodar a API
dotnet run --project api/src/ClientManager.WebApi --launch-profile http
```

Endpoints assim que a API subir:

- Health: http://localhost:5000/health
- Swagger UI: http://localhost:5000/swagger

### Conexão com o banco

O `docker-compose.yml` publica SQL Server na porta **1433** (ou Postgres na **5432**). As credenciais de dev e a connection string ficam em [api/src/ClientManager.WebApi/appsettings.json](api/src/ClientManager.WebApi/appsettings.json) (e seu equivalente `appsettings.Development.json`).

> Para trocar de SQL Server para PostgreSQL, ajuste o provider em `ClientManager.IoC` e a `DefaultConnection` em `appsettings.json`. Os mappings EF Core são agnósticos quanto ao provider, exceto pelo concurrency token (`RowVersion` no SQL Server, `xmin` no PostgreSQL).

## Executando o frontend

```bash
cd front
npm install
npm run start
```

App disponível em http://localhost:4200. A URL da API é configurada em `front/src/environments/environment.ts`.

## Autenticação

Todo endpoint protegido exige **JWT Bearer**. Fluxo de bootstrap:

### 1. Criar um usuário

```bash
curl -X POST http://localhost:5000/api/Users \
  -H "Content-Type: application/json" \
  -d '{
    "username":"admin",
    "password":"Admin@123",
    "email":"admin@clientmanager.test",
    "phone":"+5511999999999",
    "status":1,
    "role":3
  }'
```

- `status` e `role` são enums numéricos (`1` = Active, `3` = Admin).
- Política de senha: ≥ 8 caracteres com maiúscula, minúscula, dígito e caractere especial.

### 2. Autenticar

```bash
curl -X POST http://localhost:5000/api/Auth \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@clientmanager.test","password":"Admin@123"}'
```

Use o `data.token` retornado no header `Authorization: Bearer {token}` (ou no botão **Authorize** do Swagger).

## Domínio

### Agregado `Client`

Raiz de agregado que carrega seus **contatos** e **oportunidades**. Invariantes são reforçadas no próprio agregado, lançando `DomainException` quando violadas. Clientes da API nunca enviam valores derivados.

```
Client
├── Contact[]       (pessoas de contato — nome, cargo, email, telefone)
└── Opportunity[]   (oportunidades comerciais — estágio, valor, data prevista)
```

### Estágios de Oportunidade

| Estágio | Descrição |
|---|---|
| `Prospecting` | Prospecção inicial |
| `Qualification` | Qualificação do lead |
| `Proposal` | Proposta enviada |
| `Negotiation` | Em negociação |
| `Won` | Fechada com sucesso |
| `Lost` | Perdida |

Transições inválidas (ex.: `Won` → `Prospecting`) são rejeitadas pelo agregado com `DomainException`.

## API — endpoints principais

### Clientes

| Método + Rota | Status |
|---|---|
| `POST /api/Clients` | 201 / 400 / 422 |
| `GET /api/Clients` | 200 |
| `GET /api/Clients/{id}` | 200 / 404 |
| `PUT /api/Clients/{id}` | 200 / 400 / 404 / 422 |
| `DELETE /api/Clients/{id}` | 204 / 404 |

### Contatos (aninhados no cliente)

| Método + Rota | Status |
|---|---|
| `POST /api/Clients/{id}/contacts` | 201 / 400 / 404 / 422 |
| `PUT /api/Clients/{id}/contacts/{contactId}` | 200 / 400 / 404 / 422 |
| `DELETE /api/Clients/{id}/contacts/{contactId}` | 204 / 404 |

### Oportunidades

| Método + Rota | Status |
|---|---|
| `POST /api/Clients/{id}/opportunities` | 201 / 400 / 404 / 422 |
| `PUT /api/Clients/{id}/opportunities/{opportunityId}` | 200 / 400 / 404 / 422 |
| `PATCH /api/Clients/{id}/opportunities/{opportunityId}/stage` | 200 / 404 / 422 |
| `DELETE /api/Clients/{id}/opportunities/{opportunityId}` | 204 / 404 |

### Exemplo — criar cliente com contatos e oportunidade

```bash
curl -X POST http://localhost:5000/api/Clients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "legalName": "Acme Ltda",
    "tradeName": "Acme",
    "document": "12.345.678/0001-90",
    "email": "contato@acme.com",
    "phone": "+551133334444",
    "contacts": [
      { "name": "João Silva", "role": "Diretor de Compras", "email": "joao@acme.com", "phone": "+5511988887777" }
    ],
    "opportunities": [
      { "title": "Renovação anual", "stage": "Proposal", "estimatedValue": 50000.00, "expectedCloseDate": "2026-06-30" }
    ]
  }'
```

### Listagem com paginação/filtros

```bash
curl "http://localhost:5000/api/Clients?_page=1&_size=10&_order=legalName%20asc&legalName=acme" \
  -H "Authorization: Bearer $TOKEN"
```

Query params comuns: `_page`, `_size`, `_order`, filtros específicos do recurso (`legalName`, `document`, `stage`, `_minExpectedCloseDate`, `_maxExpectedCloseDate`, etc.).

## Tratamento de erros

Um middleware global mapeia exceções conhecidas para um payload consistente `{ type, error, detail }`:

| Exceção | HTTP | `type` |
|---|---|---|
| `FluentValidation.ValidationException` | 400 | `ValidationError` |
| `NotFoundException` | 404 | `ResourceNotFound` |
| `DomainException` | 422 | `DomainError` |
| Demais `Exception` | 500 | `InternalError` |

Stack traces de exceções não tratadas são logadas via Serilog e **nunca** retornadas na resposta.

## Arquitetura

### Camadas (Clean Architecture)

```
   WebApi  ─────────▶  IoC  ─────────▶  Application  ─────────▶  Domain
     │                                       │                        ▲
     │                                       └──▶  ORM  ──────────────┘
     └────────────────▶  ORM (via IoC module)
```

- **Domain** — entidades, Value Objects, eventos de domínio, interfaces de repositório. Sem dependências de framework além de MediatR (para `INotification`).
- **Application** — Commands, Queries, Handlers, Validators. Depende só de Domain + AutoMapper + FluentValidation + MediatR.
- **ORM** — EF Core, mappings, migrations, implementações de repositório.
- **IoC** — compõe os módulos e é consumido pela WebApi.
- **Common** — utilitários transversais (JWT, `ValidationBehavior`, health checks).
- **WebApi** — Controllers finos que apenas orquestram via `IMediator`.

### Fluxo de requisição (POST /api/Clients)

```
HTTP request
    │
    ▼
ClientsController ── Mediator.Send(CreateClientCommand) ──▶ ValidationBehavior (FluentValidation)
                                                                │
                                                                ▼
                                                        CreateClientHandler
                                                                │
                                                                ▼
                                                       Client aggregate (regras)
                                                                │
                                                                ▼
                                                       ClientRepository (EF Core)
                                                                │
                                                                ▼
                                                             Database
                                                                │
                          ┌──────────────── SaveChanges OK ──────┘
                          ▼
               drena DomainEvents ── Mediator.Publish ──▶ LoggingHandlers (Serilog)
                          │
                          ▼
                  ClientResponse (AutoMapper) ──▶ HTTP 201
```

Eventos de domínio são publicados **somente após** `SaveChangesAsync` bem-sucedido.

## Testes

```bash
cd api
dotnet test
```

Três projetos:

- **ClientManager.Unit** — agregados, handlers e validators. Infra mockada com NSubstitute; dados fake via Bogus. Sem Docker.
- **ClientManager.Integration** — repositórios contra banco efêmero via Testcontainers. Exige Docker Desktop.
- **ClientManager.Functional** — `WebApplicationFactory<Program>` + banco em container + `TestAuthHandler` que bypassa JWT para exercer endpoints `[Authorize]` sem emitir tokens reais.

Frontend:

```bash
cd front
npm run test
```

## Decisões / notas

- **Clean Architecture** — Domain é o núcleo; dependências só apontam para dentro. Application não conhece EF Core nem ASP.NET.
- **CQRS** — Commands e Queries são segregados mesmo quando compartilhariam um handler, para deixar intenção explícita.
- **Concurrency token** — `RowVersion` (`byte[]`) em SQL Server; trocar para `xmin` (shadow property) se migrar para PostgreSQL. Um `RowVersion` literal não se autoatualiza no Postgres.
- **Agregados enxutos** — `Contact` e `Opportunity` pertencem ao agregado `Client`; toda mutação passa pelo root para proteger invariantes.
- **Delete** — hard delete administrativo (sem evento). Mudanças de estado relevantes ao negócio (ex.: `Opportunity` marcada como `Won` ou `Lost`) emitem eventos de domínio dedicados.
- **Frontend standalone** — nada de `NgModule` por feature; componentes, diretivas e pipes são standalone. Rotas lazy-loaded por feature.
- **Envelope de resposta** — toda resposta segue `{ data, success, message? }`; o frontend desembrulha no serviço antes de entregar ao componente.

## Guidelines

Referências completas durante o desenvolvimento:

- Backend — [guideline/backend.md](guideline/backend.md)
- Frontend — [guideline/frontend.md](guideline/frontend.md)
