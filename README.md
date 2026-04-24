# dotnet-angular-client-manager

CRM simples para cadastrar **Clientes**, seus **Contatos** e **Oportunidades comerciais**. API REST em **.NET 8** seguindo **DDD + CQRS + Clean Architecture + SOLID**, consumida por um frontend **Angular** com **Angular Material** e **Reactive Forms**.

## Objetivo

Centralizar o ciclo de relacionamento comercial: cadastrar clientes (PJ/PF), manter seus contatos e acompanhar oportunidades do pipeline (prospecção → negociação → ganho/perda). Resolve o problema de times comerciais pequenos que gerenciam base de clientes e pipeline em planilhas dispersas, sem histórico consistente nem controle de estágio.

## Stack

- **Backend:** .NET 8, ASP.NET Core WebApi, MediatR (CQRS), AutoMapper, FluentValidation, Serilog, JWT Bearer
- **Frontend:** Angular (LTS, standalone components), Angular Material, Reactive Forms, RxJS, TypeScript strict
- **Banco:** SQL Server (principal), PostgreSQL (suportado), EF Core 8
- **Cloud:** Containers Docker (deploy agnóstico — Azure App Service / AWS ECS / GCP Cloud Run)
- **Testes:** xUnit, NSubstitute, Bogus, FluentAssertions, Testcontainers, Jasmine/Karma

## Arquitetura

- **DDD** — agregado `Client` como raiz, `Contact` e `Opportunity` como entidades filhas; invariantes protegidas por `DomainException`.
- **CQRS** — Commands e Queries segregados via MediatR, mesmo quando compartilhariam handler.
- **SOLID** — dependências apontando para o núcleo, interfaces no Domain, implementações na ORM.
- **Clean Architecture** — quatro camadas (Domain, Application, ORM, WebApi) + IoC e Common. Application não conhece EF Core nem ASP.NET.

## Funcionalidades

- **Cadastro** de clientes, contatos e oportunidades (com validação de documento, email e telefone)
- **Edição** de dados cadastrais e transição de estágio de oportunidade (regras de transição no agregado)
- **Exclusão** de registros (hard delete administrativo)
- **Listagem** paginada e ordenável
- **Filtros** por razão social, documento, estágio de oportunidade e janela de data prevista
- **Autenticação** via JWT Bearer com política de senha forte e roles (Admin/User)

## Como executar

```bash
# 1. Subir banco (SQL Server ou Postgres)
docker-compose up -d

# 2. Aplicar migrations
dotnet ef database update \
  --project api/src/ClientManager.ORM \
  --startup-project api/src/ClientManager.WebApi

# 3. Rodar a API
dotnet run --project api/src/ClientManager.WebApi --launch-profile http

# 4. Rodar o frontend
cd front && npm install && npm run start
```

- API: http://localhost:5000 (Swagger em `/swagger`)
- App: http://localhost:4200

## Prints

_Adicionar imagens das telas (lista de clientes, formulário de cadastro, pipeline de oportunidades, login)._

## Aprendizados demonstrados

- **DDD na prática** — agregado com invariantes reforçadas (transições inválidas de estágio rejeitadas no próprio `Client`), não em services anêmicos.
- **CQRS com MediatR** — pipeline behavior de `FluentValidation` interceptando Commands antes do Handler; eventos de domínio publicados **após** `SaveChangesAsync`.
- **Entity Framework Core** — mappings agnósticos de provider, concurrency token (`RowVersion` no SQL Server, `xmin` no Postgres), migrations versionadas.
- **FluentValidation** — validação de entrada separada das invariantes de domínio; a primeira cai em 400, a segunda em 422.
- **JWT** — autenticação stateless + `TestAuthHandler` nos testes funcionais para bypass controlado de `[Authorize]`.
- **Angular Reactive Forms + Material** — formulários tipados, feedback de erro consistente, componentes standalone com rotas lazy-loaded.
- **Testes em três camadas** — unitários (handlers e agregados com mocks), integração (repositórios contra banco efêmero via Testcontainers) e funcionais (`WebApplicationFactory<Program>` contra banco real em container).
- **Middleware global de erros** — mapeamento consistente de exceções para `{ type, error, detail }`, sem vazamento de stack trace.
