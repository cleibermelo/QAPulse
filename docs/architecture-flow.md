# Fluxo da aplicação: UI, Supabase, SonarCloud e GitHub Actions

## Visão geral

```mermaid
flowchart LR
    subgraph GH[GitHub]
        CRON[Agendamento cron<br/>06:00 UTC]
        JOB[Job sonar-sync<br/>Node no Actions]
        SEC[(GitHub Secrets)]
        PAGES[GitHub Pages<br/>ficheiros estáticos]
        DEPLOY[Workflow pages.yml<br/>build + deploy]
    end

    SONAR[SonarCloud API]
    SUPA[(Supabase<br/>Auth + Postgres + API)]
    UI[Navegador<br/>Angular + supabase-js]

    CRON --> JOB
    SEC -. token e chave secreta .-> JOB
    JOB -->|1. consulta histórico| SONAR
    JOB -->|2. upsert com chave secreta| SUPA

    DEPLOY -->|publica| PAGES
    PAGES -->|carrega a aplicação| UI
    UI -->|3. login + JWT, leitura e escrita com RLS| SUPA
```

## Os dois ciclos independentes

### A. Alimentação dos dados (diária, sem utilizador)

```mermaid
sequenceDiagram
    participant C as Cron (GitHub)
    participant J as Job (Actions)
    participant S as SonarCloud
    participant D as Supabase

    C->>J: dispara às 06:00 UTC
    J->>J: lê secrets (token Sonar, chave Supabase)
    J->>S: projects/search (org + filtro)
    S-->>J: lista de projetos
    loop para cada projeto
        J->>S: branches e measures/search_history
        S-->>J: histórico de cobertura
        J->>J: une master e main, último valor por mês
    end
    J->>D: upsert em sonar_coverage_monthly (lotes de 500)
    D-->>J: confirmação
```

### B. Utilização da aplicação (quando alguém abre a página)

```mermaid
sequenceDiagram
    participant U as Utilizador
    participant P as GitHub Pages
    participant B as Navegador (Angular)
    participant D as Supabase

    U->>P: abre a URL
    P-->>B: HTML, JS e CSS estáticos
    B->>D: login (e-mail e senha)
    D-->>B: JWT da sessão
    B->>D: lê dados (anonKey + JWT)
    D->>D: aplica RLS (papel authenticated)
    D-->>B: dados (equipas, aplicações, cobertura Sonar)
    B-->>U: dashboards e página SonarCloud
```

## Quem tem acesso a quê

| Componente | Fala com | Credencial |
|---|---|---|
| Navegador | Supabase | `anonKey` pública + JWT do login |
| Navegador | SonarCloud | Nunca fala |
| Job (Actions) | SonarCloud | `SONAR_TOKEN` (GitHub Secret) |
| Job (Actions) | Supabase | `SUPABASE_SERVICE_ROLE_KEY` (GitHub Secret) |
| Supabase | SonarCloud | Nunca fala |
| Pages | Ninguém | Só serve ficheiros estáticos |

## Publicação da aplicação

Um push no `main` dispara o workflow `pages.yml`: instala dependências, faz o build do Angular e publica os ficheiros em GitHub Pages. Isso não toca em dados e não usa os secrets do Sonar ou do Supabase.

## Pontos-chave

- **Não há servidor próprio.** O Supabase faz de backend de dados e autenticação, e o GitHub Actions faz de "servidor" do job agendado.
- **A página SonarCloud só mostra o que o job já gravou.** Se o job falhar, a página mostra os dados da última execução bem-sucedida.
- **Os dois ciclos só se encontram na tabela** `sonar_coverage_monthly`: o job escreve, a UI lê.
- **O job grava sempre.** Ele envia o histórico completo e o Supabase faz upsert por `(project_key, month)`: insere o que é novo e atualiza o que já existe.
- **A segurança está no banco (RLS + JWT),** não na interface. A `anonKey` pública no repositório não dá acesso a dados sem login.
