# Painel de Inscrições – Plano de Implementação

Guia para criar o painel administrativo que irá listar, filtrar e detalhar os leads coletados pelo formulário do Workshop Inaugural da VOZUP. O objetivo é hospedar esse painel na Vercel e ler diretamente os dados já armazenados no Postgres da Hostinger (`inscricoes.inscricoes`).

## Visão Geral
- Painel em **Next.js 14+ (App Router)** com **TypeScript** e **Tailwind CSS**.
- Camada de dados usando `pg.Pool` reutilizando a mesma base do formulário com um **usuário somente leitura**.
- Autenticação usando **biblioteca/provedor dedicado** (ex.: Auth.js, Clerk, Better Auth) com sessão HttpOnly, RBAC e opcionalmente MFA.
- Páginas protegidas exibem tabela paginada com ordenação, filtros por nome/telefone e modal com o JSON completo.
- API interna (`GET /api/inscricoes`) para retornar os dados paginados, reaproveitando a lógica do dashboard.
- Ver [SECURITY.md](SECURITY.md) para a baseline recomendada de segurança e deploy.

## Como Iniciar (estrutura sugerida)
1. `npx create-next-app@latest painel-inscricoes --ts --app --tailwind --eslint`.
2. Remover boilerplate e configurar Tailwind conforme padrão do time.
3. Criar as pastas e arquivos:
   - `app/(dashboard)/layout.tsx` e `page.tsx` (server components) com tabela e cabeçalho.
   - `app/login/page.tsx` e `app/api/login/route.ts` para autenticação.
   - `app/api/inscricoes/route.ts` (GET) com paginação e filtros.
   - `components/InscricoesTable.tsx`, `components/InscricaoDetails.tsx`.
   - `lib/db.ts`, `lib/listInscricoes.ts`, `lib/parsePayload.ts`, `lib/auth.ts`.
   - `types/inscricao.ts` com tipagem opcional do payload JSONB.

## Banco de Dados
- Schema já existente: `inscricoes`.
- Tabela: `inscricoes.inscricoes` com colunas `id`, `payload JSONB`, `criado_em TIMESTAMPTZ`.
- Criar função utilitária `ensureSchema` apenas para ambientes de teste (não rodar em produção se já existe).
- Opcional: usuário readonly
  ```sql
  CREATE USER inscricoes_readonly WITH PASSWORD 'troque-a-senha';
  GRANT USAGE ON SCHEMA inscricoes TO inscricoes_readonly;
  GRANT SELECT ON ALL TABLES IN SCHEMA inscricoes TO inscricoes_readonly;
  ALTER DEFAULT PRIVILEGES IN SCHEMA inscricoes GRANT SELECT ON TABLES TO inscricoes_readonly;
  ```

## Variáveis de Ambiente
Criar `.env.local`:
```
DATABASE_URL=postgres://<readonly-user>:<readonly-password>@<host>:<port>/<database>?sslmode=require
AUTH_SECRET=<gere-um-segredo-longo-e-aleatorio>
AUTH_TRUST_HOST=true
```
> **Importante:** nunca commitar segredos reais em `README`, `.env`, prints ou issues. Se algum segredo real já foi exposto, faça a rotação imediatamente.
>
> **Nota:** o Aiven exige SSL. Sempre prefira um usuário readonly dedicado ao dashboard.
Adicionar `.env.example` com placeholders.

## Fluxo da Aplicação
- `/login`: fluxo de autenticação do provedor escolhido (magic link, SSO ou credencial forte com MFA).
- Middleware (`middleware.ts`) e verificações server-side impedem acesso sem sessão válida e role autorizada.
- `/` (dashboard): server component carrega dados via `listInscricoes({ page, pageSize, orderBy, q })`.
- Botão “Ver detalhes” abre modal (client component) renderizando o JSON formatado.
- Badge no topo mostra total de inscrições (consulta rápida `COUNT(*)`).

## API Interna
- `GET /api/inscricoes` aceita `page`, `pageSize`, `orderBy`, `direction`, `q` (busca parcial em nome/telefone).
- Reutiliza a função `listInscricoes` e retorna `{ data, pagination, total }`.
- Valida a sessão do dashboard no servidor e aplica autorização por role antes de consultar o banco.

## Tratamento de Erros e Segurança
- Envolver consultas em try/catch, logar no servidor e retornar mensagens amigáveis ao cliente.
- Proteger rota e página com sessão HttpOnly `Secure`, `SameSite=Lax` ou `Strict`, expiração curta e renovação controlada.
- Aplicar rate limit em `/login` e `/api/inscricoes`.
- Usar CSP e headers defensivos.
- Nunca expor `DATABASE_URL` ao cliente nem usar variáveis `NEXT_PUBLIC_*` para segredos.
- Evitar vazar stack traces em produção (usar `console.error` apenas no server).

## Testes
- Configurar `jest.config.js` compatível com Next + TS.
- Criar `lib/__tests__/parsePayload.test.ts` para validar normalização do payload.
- Usar `supertest` em `app/api/inscricoes/route.test.ts` para garantir paginação/filtros.
- Scripts no `package.json`:
  ```json
  {
    "scripts": {
      "dev": "next dev",
      "lint": "next lint",
      "test": "jest"
    }
  }
  ```

## Deploy na Vercel
1. Conectar repositório `painel-inscricoes` à Vercel.
2. Definir variáveis (`DATABASE_URL`, `AUTH_SECRET`, `VERCEL_ENV`) como **Sensitive Environment Variables**.
3. Ativar **Deployment Protection** e, se o dashboard precisar ficar privado em produção, usar plano/escopo que proteja todos os deployments.
4. Configurar regras de **WAF / Rate Limiting** para `/login` e `/api/inscricoes`.
5. Usar região padrão (Node.js 18+). Não precisa de Edge Functions.
6. Deploy automático a cada push na branch principal.

### Checks pós-deploy
- `/login` exige token e gera cookie.
- `/` lista inscrições reais e abre modal de detalhes.
- Filtros e ordenação funcionam.
- Logs na Vercel sem erros de SSL ou conexão.
- Rodar `npm run lint`, `npm run test`, `npm run dev` antes de subir alterações críticas.

## Próximos Passos
1. Exportar CSV direto da tabela quando necessário.
2. Adicionar gráficos simples (inscrições por dia) com alguma lib leve (por exemplo, `recharts`).
3. Integração com o n8n para disparar alertas ou criar tarefas após cada nova inscrição.
