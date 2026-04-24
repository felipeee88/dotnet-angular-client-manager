# Guideline — Frontend (Angular)

Este documento define as convenções de arquitetura, código e UX para o frontend Angular do **dotnet-angular-client-manager**. Serve como referência ao implementar as telas de **Clientes**, **Contatos** e **Oportunidades**.

## Stack

- **Angular** (última versão LTS, standalone components)
- **Angular Material** — biblioteca de componentes
- **Reactive Forms** — formulários sempre reativos, nunca template-driven
- **RxJS** — fluxos assíncronos e HTTP
- **TypeScript** estrito (`strict: true` em `tsconfig.json`)
- **ESLint** + **Prettier** para padronização
- **Jasmine/Karma** (ou Jest) para testes unitários

## Estrutura de pastas

```
front/
└── src/
    └── app/
        ├── core/         (serviços singletons, interceptors, guards, models globais)
        ├── shared/       (componentes, pipes e diretivas reutilizáveis)
        ├── layout/       (shell da aplicação: header, sidenav, footer)
        └── features/
            ├── clients/        (lista, detalhe, formulário de cliente)
            ├── contacts/       (CRUD de contatos do cliente)
            └── opportunities/  (CRUD de oportunidades do cliente)
```

Regras:

- **core/** — importado uma única vez; contém `AuthService`, `HttpInterceptors` (JWT, error handling), `AuthGuard`, modelos/tipos compartilhados por toda a aplicação.
- **shared/** — sem estado; só exporta peças reutilizáveis (ex.: `ConfirmDialogComponent`, pipe de formatação de moeda).
- **layout/** — componentes estruturais da UI principal.
- **features/** — cada feature é autocontida: rotas próprias (`*.routes.ts`), componentes standalone, serviços específicos da feature.

## Autenticação

- `AuthService` gerencia login, logout e persistência do JWT (localStorage ou sessionStorage).
- `AuthInterceptor` injeta `Authorization: Bearer {token}` em todas as requisições HTTP.
- `AuthGuard` (functional guard) protege rotas autenticadas.
- `ErrorInterceptor` trata 401 (redireciona para login) e 403 (mensagem de acesso negado).

## Formulários (Reactive Forms)

- Sempre `FormGroup` + `FormControl` tipados (`FormGroup<ClientFormModel>`).
- Validações no `FormBuilder`; mensagens de erro padronizadas por um componente `FormFieldError`.
- Submissão desabilita o botão enquanto a requisição está em voo.
- Erros 400 (`ValidationError`) do backend são mapeados de volta para os controles correspondentes.

## UI (Angular Material)

- Layout responsivo com `MatSidenav` + `MatToolbar`.
- Listas em `MatTable` com `MatPaginator` e `MatSort`.
- Formulários em `MatFormField` + `matInput` / `MatSelect` / `MatDatepicker`.
- Diálogos de confirmação (excluir, cancelar) em `MatDialog`.
- Feedback rápido via `MatSnackBar` para sucesso/erro.
- Tema Material customizado em `src/styles.scss` — sem sobrescrever classes `.mat-*` direto nos componentes.

## Comunicação HTTP

- Um serviço por recurso (`ClientsService`, `ContactsService`, `OpportunitiesService`) em `features/<feature>/services/`.
- Métodos retornam `Observable<T>` — nada de `Promise`.
- Tipos de request/response definidos em `*.model.ts` ao lado do serviço.
- Respostas seguem o envelope do backend `{ data, success, message? }` — o serviço desembrulha e entrega apenas `data` ao componente.
- `environment.ts` define `apiBaseUrl` (ex.: `http://localhost:5119`).

## Roteamento

- Rotas lazy-loaded por feature via `loadChildren: () => import('./features/clients/clients.routes')`.
- `AuthGuard` aplicado nas rotas que exigem autenticação.
- Breadcrumbs (se usados) derivam do `data` das rotas.

## Padrões de código

- **Standalone components** em toda a aplicação (sem `NgModule` de feature).
- Nomes de arquivo: `kebab-case.component.ts`, `kebab-case.service.ts`, `kebab-case.model.ts`.
- Um componente/serviço por arquivo.
- `OnPush` change detection por padrão.
- `takeUntilDestroyed()` ou signals para gerenciar inscrições — nunca `subscribe` sem cleanup.
- Sem `any` — tipagem explícita sempre; `unknown` quando necessário.

## Testes

- Unit tests com `TestBed` para componentes e serviços críticos.
- Serviços mockados com `jasmine.createSpyObj` ou `provideHttpClientTesting`.
- Cobertura focada em: validação de forms, fluxos de erro, mapeamento de DTOs.

## Acessibilidade e i18n

- Todos os inputs com `<mat-label>`; ícones puramente decorativos com `aria-hidden="true"`.
- Textos voltados ao usuário centralizados (preparar para i18n — Angular `@angular/localize`).
- Contraste mínimo WCAG AA respeitado pelo tema Material escolhido.

## Build e ambientes

- `npm run start` — dev server em `http://localhost:4200`.
- `npm run build` — build de produção em `dist/`.
- `environment.ts` / `environment.prod.ts` com as variáveis de ambiente (URL da API, flags).
