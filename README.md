# Findash (Fluxo) — personal finance

An integrated personal finance app (UI in Portuguese) with username/password sign-up, account recovery, a persistent database, automatic budgeting, credit cards and goals. It never fills the dashboard with fake data.

[🇧🇷 Leia em português](#-português)

## Running locally

Requirements: Node.js 24 LTS, npm and Git. Node 24 also provides the SQLite used by the integration tests.

```powershell
git clone https://github.com/UxUchoa/Findash.git
cd Findash
npm ci
npm run build
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_neat_nova.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0001_black_blur.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0002_clean_lifeguard.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0003_tiresome_wiccan.sql
npm run dev
```

Open the URL printed by the server, usually http://127.0.0.1:5173. Apply each migration only once per database; the four commands above are for a fresh database.

To test the production artifact locally, run `npm start` after the build and open the URL printed by Wrangler. Development and preview share `.wrangler/state`; don't delete it if you want to keep local data. The production database is separate.

## Verification

```powershell
npm test
npm run typecheck
npm run build
```

The Vitest suite runs pure calculations and HTTP flows against real SQLite in a temporary file, with foreign keys, transactions and database reopening. It doesn't mock calculations, authorization or persistence. The test adapter reproduces D1's prepared-statement/batch contract; runtime differences are also checked on the local Worker/D1 and on deploy.

Scenarios include allocation, goals, deficit, installments and cents, partial payment, recovery/rotation/revocation, account isolation, CSRF, persistence after logout, reserves across months, reversals, recurrence changes, edge dates and rate limiting.

## What's implemented

- E-mail-free sign-up, normalized usernames, Argon2id (19 MiB, 2 iterations, parallelism 1) and per-user salt.
- 160-bit recovery code shown once, with copy and download. Only the SHA-256 of the random code is stored; using it rotates the code and revokes previous sessions.
- Opaque random 256-bit sessions, verifier in the database, 30-day expiry, HttpOnly, SameSite=Strict and Secure-on-HTTPS cookies.
- Persistent rate limiting per user and origin; Origin checks on mutations. No password or code in logs.
- Five-step skippable onboarding with multiple income sources, initial balance and configurable cycle.
- Income/expense CRUD, confirmation/reversal, duplication, search and filters by date, category and status.
- Monthly (24 occurrences) and yearly (3) recurrences, editable per occurrence or going forward, preserving past and confirmed ones.
- Credit cards, purchases, up to 60 installments, manual review of the first statement, valid closing/due dates, partial payment and reversal.
- Goals by deadline or contribution, last contribution capped to what's left, contributions, withdrawals, reversal, history, priorities, pause, completion and archive.
- Virtual reserves explicitly linked to contributions. Contributions consume the oldest reserves, including other months'; reversals restore the links.
- Server-calculated dashboard, obligation coverage by priority, confirmed money separate from projections, formula explanation and deterministic diagnosis.
- Unified history, category limits, 3/6/12-month commitment projection, JSON export and password-confirmed account deletion.
- Mobile/desktop layout, dark theme, labels, visible focus, native dialog with focus isolation and reduced-motion support.

## Architecture and integrity

`app/` holds the page and HTTP endpoint; `components/` the UI; `lib/finance.ts` the deterministic engine; `lib/server.ts` validation/authorization and operations; `lib/security.ts` credentials; `db/schema.ts` and `drizzle/` the schema and migrations.

Stack: React 19, TypeScript, App Router-style routes, Tailwind and hand-written CSS, Lucide, Zod, Drizzle and Cloudflare D1, running on Vinext (a Next.js-compatible runtime on Vite, pinned beta in the lockfile) as a Cloudflare Worker. It doesn't use PostgreSQL/Prisma, Auth.js or Vercel, and shouldn't be deployed to Vercel without adapting the Worker and D1 dependencies.

All amounts are integer cents. The balance derives from the initial balance, confirmed movements and statement payments. Contributions and reserves aren't cash outflows. Purchases are consumption, installments are debt and payments are cash: a payment never creates a new consumption expense.

Every financial query and mutation filters by owner on the server, and the backend validates links before creating them. Related mutations use atomic D1 batches and optimistic per-user revisions with an exclusive token; concurrent changes return 409 with no partial writes. Free-money checks use the current cycle regardless of the month chosen in the browser.

Card policy: a purchase on the closing day goes to the next cycle; a due day on or before the closing day belongs to the month after closing; nonexistent days are clamped to the month's last day; card changes don't alter dates of existing statements. To fix a purchase in a statement that has payments, reverse those payments first.

Projections subtract pending and overdue obligations. Free money today considers the current cycle and, when viewing a future cycle, its commitments too. Overdue or future income is never assumed received. 3/6/12-month projections show each cycle's result, with no accumulated balance or returns.

One main financial account per user in the MVP. Money reported as already saved must be included in the initial balance. Interest is entered by the user in the purchase's final total; there's no automatic charge calculation.

## Configuration

No application secret is required: sessions and recovery use cryptographic randomness and database verifiers. Never put bank credentials or tokens in the frontend.

- Required binding: `DB` (D1), declared in `.openai/hosting.json`.
- `WRANGLER_SEND_METRICS`: optional, off by default.
- `CLOUDFLARE_CF_FETCH_ENABLED`: optional, for preview metadata only.
- Local profile config lives in `.sites-runtime/`, ignored by Git.

## Deployment

The app is published as a private Site (OpenAI Sites hosting, which provisions D1 and applies migrations before publishing the Worker). The project identity is in `.openai/hosting.json`: reuse it instead of creating a second Site. The flow is: run tests and typecheck, build with the Sites `build-site.mjs` helper (keeping Vite's `sites()` integration), sync with `site-workflow.mjs` passing a write credential through stdin, then save and publish the returned commit and verify the publish state. Temporary publish tokens must never be written to the repository.

The hosted Site is private by default, on top of Fluxo's own login. Before opening it to third parties, the privacy notice needs the controller's identity/contact and an infrastructure retention policy.

## Limits and next steps

P0 is implemented within the scope above. Out of scope: automatic weekly/biweekly frequencies, infinite recurrence renewal, multiple accounts/advanced reconciliation, UI backup restore and actual money transfers. Recurrences have an explicit horizon and must be renewed when they end. Editing a contribution is done by reversing it and entering a corrected one, preserving the audit trail.

Pending P1/P2: scenario simulator, accumulated projection with goal evolution, CSV, PWA, notifications, Open Finance and an AI assistant. The privacy notice is not a legal compliance certification. The initial balance date can't be moved after existing movements, so the same money is never counted twice.

---

## 🇧🇷 Português

# Findash (Fluxo) — gestão financeira pessoal

Aplicação integrada em português, com cadastro por usuário/senha, recuperação de conta, banco persistente, orçamento automático, cartões e metas. Não usa dados fictícios para preencher o dashboard.

## Executar localmente

Requisitos: Node.js 24 LTS, npm e Git. O Node 24 também fornece o SQLite usado nos testes de integração.

```powershell
git clone https://github.com/UxUchoa/Findash.git
cd Findash
npm ci
npm run build
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0000_neat_nova.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0001_black_blur.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0002_clean_lifeguard.sql
node --import ./scripts/sites-env.mjs ./node_modules/wrangler/bin/wrangler.js d1 execute DB --local --config dist/server/wrangler.json --persist-to .wrangler/state --file drizzle/0003_tiresome_wiccan.sql
npm run dev
```

Abra a URL impressa pelo servidor, normalmente http://127.0.0.1:5173. Aplique cada migration somente uma vez por banco. Os quatro comandos de migration acima são para um banco novo.

Para testar o artefato de produção localmente, use `npm start` após o build, e abra a URL impressa pelo Wrangler. Desenvolvimento e preview compartilham `.wrangler/state`; não apague essa pasta se quiser preservar os dados locais. O banco de produção é separado.

## Verificação

```powershell
npm test
npm run typecheck
npm run build
```

A suíte Vitest executa cálculos puros e fluxos HTTP contra SQLite real em arquivo temporário, com chaves estrangeiras, transações e reabertura do banco. Não simula cálculos, autorização nem persistência. O adaptador de testes reproduz o contrato de prepared statements/batch do D1; diferenças de runtime são verificadas adicionalmente no Worker/D1 local e no deploy.

Os cenários incluem distribuição, metas, déficit, parcelas e centavos, pagamento parcial, recuperação/rotação/revogação, isolamento entre contas, CSRF, persistência após logout, reservas entre meses, estornos, alterações de recorrência, datas extremas e rate limiting.

## O que está implementado

- Cadastro sem e-mail, usuário normalizado, Argon2id (19 MiB, 2 iterações, paralelismo 1) e salt individual.
- Código de recuperação de 160 bits apresentado uma vez, cópia e download. Apenas SHA-256 do código aleatório é persistido. Uso rotaciona o código e revoga sessões anteriores.
- Sessões opacas aleatórias de 256 bits, verificador no banco, expiração de 30 dias, cookies HttpOnly, SameSite=Strict e Secure em HTTPS.
- Rate limiting persistente por usuário e origem; verificação de Origin em mutações. Nenhuma senha ou código em logs.
- Onboarding em cinco etapas, com possibilidade de pular, múltiplas fontes de renda, saldo inicial e ciclo configurável.
- CRUD de receitas/despesas, confirmação/estorno, duplicação, pesquisa e filtros de data, categoria e situação.
- Recorrências mensais (24 ocorrências) e anuais (3), editáveis por ocorrência ou para o futuro, preservando ocorrências passadas e confirmadas.
- Cartões, compras, até 60 parcelas, revisão manual da primeira fatura, fechamento/vencimento com datas válidas, pagamento parcial e estorno.
- Metas por prazo ou aporte, último aporte limitado ao restante, aportes, retiradas, estorno, histórico, prioridades, pausa, conclusão e arquivo.
- Reservas virtuais com vínculos explícitos aos aportes. Aportes consomem as reservas mais antigas, inclusive de outros meses; estornos restauram os vínculos.
- Dashboard calculado pelo servidor, cobertura das obrigações por prioridade, dinheiro confirmado separado de projeção, explicação da fórmula e diagnóstico determinístico.
- Histórico unificado, limites de categoria, projeção de compromissos de 3/6/12 meses, exportação JSON e exclusão da conta com senha.
- Layout mobile/desktop, tema escuro, labels, foco visível, diálogo nativo com isolamento de foco e respeito a movimento reduzido.

## Arquitetura e integridade

`app/` contém a página e o endpoint HTTP; `components/`, a interface; `lib/finance.ts`, o motor determinístico; `lib/server.ts`, validação/autorização e operações; `lib/security.ts`, credenciais; `db/schema.ts` e `drizzle/`, o esquema e suas migrations.

A aplicação usa React 19, TypeScript, API de rotas no formato App Router, Tailwind/CSS autoral, Lucide, Zod, Drizzle e Cloudflare D1. O runtime de publicação é o Vinext fornecido pelo Sites (atualmente versão beta fixada no lockfile). Essa é uma adaptação da stack sugerida: não utiliza PostgreSQL/Prisma, Auth.js ou Vercel. O build não deve ser publicado diretamente na Vercel sem adaptar as dependências do Worker e do D1.

Todos os valores são centavos inteiros. O saldo é derivado do saldo inicial, movimentos confirmados e pagamentos de fatura. Aportes e reservas não são saídas de caixa. Compras representam consumo, parcelas representam dívida e pagamentos representam caixa: o pagamento não cria nova despesa de consumo.

Toda consulta financeira e mutação filtra o dono no servidor. O backend valida os vínculos antes de criá-los. Mutações relacionadas usam D1 batch atômico e revisão otimista por usuário com token exclusivo; alterações concorrentes retornam 409 sem gravação parcial. As validações de dinheiro livre usam o ciclo atual, independentemente do mês escolhido no navegador.

Política de cartão: compra no dia de fechamento entra no ciclo seguinte. Vencimento menor ou igual ao fechamento pertence ao mês seguinte ao fechamento. Dias inexistentes são limitados ao último dia do mês. Mudanças no cartão não alteram datas de faturas já criadas. Para corrigir compra que participa de fatura com pagamentos, estorne esses pagamentos primeiro.

A projeção desconta obrigações pendentes e vencidas. O dinheiro livre hoje considera o ciclo atual e, ao consultar um ciclo futuro, também seus compromissos. Não assume recebimento de receitas vencidas ou futuras. Projeções de 3/6/12 meses mostram resultado de cada ciclo, sem saldo acumulado ou rentabilidade.

Uma conta financeira principal por usuário no MVP. O dinheiro informado como já guardado precisa estar incluído no saldo inicial. Juros são informados pelo usuário no total final da compra; não há cálculo automático de encargos.

## Configuração e variáveis de ambiente

Não há segredo de aplicação obrigatório: sessões e recuperação usam aleatoriedade criptográfica e verificadores no banco. Não coloque credenciais bancárias ou tokens no frontend.

- Binding obrigatório: `DB` (D1), declarado em `.openai/hosting.json`.
- `WRANGLER_SEND_METRICS`: opcional, desativado por padrão.
- `CLOUDFLARE_CF_FETCH_ENABLED`: opcional para metadados de preview; não é necessário.
- Configuração de perfil local em `.sites-runtime/`, ignorada pelo Git.
- Tokens temporários de publicação devem ser passados ao helper por stdin; nunca gravados no repositório.

## Publicação no Sites

O projeto já tem identidade em `.openai/hosting.json`. Reutilize-a; não crie um segundo Site. O Sites provisiona o banco e aplica as migrations antes de publicar o Worker.

1. Execute testes e typecheck.
2. Execute o build com o helper `build-site.mjs` da instalação do plugin Sites, preservando a integração `sites()` do Vite.
3. Obtenha uma credencial de escrita para o mesmo projeto pelo conector Sites.
4. Use `site-workflow.mjs --project-id <id do manifest>`, passando a credencial por stdin. O helper sincroniza o código e cria um arquivo de publicação com o commit exato.
5. Salve e publique a versão privada usando o conector Sites com o commit e arquivo devolvidos pelo helper.
6. Verifique o estado terminal da publicação antes de divulgar a URL.

O Site é privado por padrão: a hospedagem exige acesso do proprietário via ChatGPT, além do login próprio do Fluxo. O cadastro da aplicação não exige e-mail. O público da hospedagem só deve ser alterado quando solicitado. Para uma oferta aberta a terceiros, também é necessário completar no aviso de privacidade a identidade/contato do responsável e a política de retenção de infraestrutura.

## Limites e próximos passos

P0 implementado no escopo descrito acima. Permanecem fora: frequências semanais/quinzenais automáticas, renovação infinita de recorrências, múltiplas contas/reconciliação avançada, restauração de backup por interface e transferência física de dinheiro. Recorrências têm horizonte explícito e devem ser renovadas ao término. Edição de um aporte é feita pelo estorno do registro e lançamento corrigido, preservando trilha de alterações.

P1/P2 pendentes: simulador de cenários, projeção acumulada com evolução de metas, CSV, PWA, notificações, Open Finance e assistente com IA. O aviso de privacidade não representa certificação de conformidade legal.

A data do saldo inicial não pode ser movida para depois de movimentações existentes: essa validação evita incorporar o mesmo dinheiro novamente ao redefinir a base.
