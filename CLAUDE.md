# Lozy Hub

Sistema de gestão da **Lozy Creative** (agência de produção audiovisual/conteúdo, São Paulo). PWA single-file, sem build step, sem framework.

## Stack e arquitetura

- **Um único arquivo**: `index.html` (~4.600 linhas) contém HTML + CSS (inline `<style>`) + toda a lógica em ~14 tags `<script>` sequenciais que compartilham o mesmo escopo global. Não há módulos ES, bundler, nem `package.json`.
- Dependência externa única: **Chart.js** via CDN (`<script src="https://cdn.jsdelivr.net/npm/chart.js...">`), usado na aba Análises do Financeiro.
- `manifest.json` + `sw.js` fazem o app funcionar como PWA instalável. O service worker só é registrado quando servido via `http(s)://` — não roda em `file://`.
- Idioma do domínio: **tudo em português** — nomes de entidades, campos, status, comentários. Mantenha esse padrão em qualquer código novo (não traduza para inglês).
- Estilo de código: denso, terso, "minifier-adjacent" mas escrito à mão (arrow functions, template literals para todo HTML, poucos espaços). Não reformate/"prettifique" — infla diffs sem ganho.

## Rodando localmente

Sem build. Basta servir o diretório estático:
```bash
python3 -m http.server 8080
```
(Abrir como `file://` também funciona, exceto o service worker.)

## Estado e persistência

- Estado global único: objeto `S` (definido perto da linha 804), contendo todas as entidades (`leads, contatos, followups, demandas, tasks, projetos, pint, fin, eventos, propostas, cbase, parc, assin, trash, dash, pricing, orcamentos, cols, sort, prefs, tab, ...`).
- **`tasks`** = Tarefas · **`pint`** = Projetos Internos · **`cbase`** = base de Clientes · **`fin`** = livro-razão único (receitas E despesas, discriminado por `.tipo`) · **`parc`** = parcelamentos · **`assin`** = assinaturas.
- Persistência: `localStorage` (ou `window.storage` quando hospedado dentro do claude.ai) sob a chave `'lozy-hub-v2'`. `save()` é **debounced em 250ms** — nunca assuma que persistiu de imediato. `KEYS` define quais chaves de `S` são salvas (estado transiente como `active`, `q`, `f`, `wiz` fica de fora).
- **Sem reatividade/framework**: qualquer mutação de estado precisa terminar em `save();render();` manualmente. Esquecer uma das duas chamadas é bug silencioso (UI velha ou dado perdido) — é o footgun mais comum ao editar este arquivo.
- Seed de dados históricos migrados do Notion (`SEED`/`SEED2`/`SEED3`) carrega automaticamente se `demandas` e `fin` estiverem vazios. Há também blocos de import "one-off" datados (`IMPORT_20260802`, `SEGUNDAS_20260802`) — dados reais de reconciliação contábil, não fixtures/exemplo para copiar.

## Módulos (`NAVGROUPS` / `SECTIONS`)

**Operação**: dashboard, comercial (CRM/kanban de leads), clientes, demandas, tarefas, pint (projetos internos), projetos (produção de cliente), agenda, financeiro.
**Ferramentas**: propostas, calculadora (motor de precificação), contratos, central (Centro Estratégico — analytics read-only), lixeira, ajustes.

- `render()` despacha por `S.active` para a função `render<Secao>` correspondente. Seções com `ready:false` em `SECTIONS` caem no placeholder "em breve".
- **`contratos` ainda não tem render function** — é só placeholder. Para implementar: adicionar `contratos:renderContratos` no dispatch de `render()` e virar `ready:true` em `SECTIONS`.
- **Propostas tem dois sistemas paralelos coexistindo** no mesmo array: wizard V1 antigo (`startWizard`/`renderWizard`/`propHTML`) e V2 novo (`startPropBlank`/`renderPropWizard`/`propostaHTML2`), distinguidos só por `p.versao===2`. Trate como dois sistemas até consolidar, não como um só.

## Padrões de UI compartilhados

- **Modal único** (`#overlay`/`#modal`): `openModal(title, bodyHtml, onSubmit, hideSubmit)` / `closeModal()`.
- **Card Drawer** (`openCard(kind,id)`): reusa o mesmo modal mas com abas (Detalhes/Tarefas/Observações/Checklist/Tempo por status) para demandas, tasks, pint e fin.
- **Kanban drag-and-drop**: engine própria via Pointer Events (não HTML5 DnD) — objeto global `DG` + `dgDown/dgMove/dgEnd/dgDrop`. Suporta 4 tipos: `demandas, tasks, leads, pint`.
- **Toast + undo**: `UNDO = {u:[], r:[], max:20}`, snapshot só das chaves de `S` afetadas (não o estado inteiro). `Cmd/Ctrl+Z` global. Toda ação relevante chama `pushUndo(label, keys)` **antes** de mutar.
- **Lixeira (soft-delete)**: `softDel(kind,id)` é a "única porta de saída" de qualquer registro — nunca deleta direto, sempre arquiva em `S.trash` com purge automática após 30 dias (`TRASH_DIAS`). `trashDrop`/`trashEmpty` são as únicas ações realmente irreversíveis do app (atrás de `confirm()`).
- Todo texto de usuário interpolado em HTML passa por `esc()` (prevenção de XSS) — mantenha essa disciplina em qualquer template novo.
- Handlers são majoritariamente `onclick="..."` inline referenciando funções globais — então (a) novas funções de UI devem ficar no escopo global, e (b) cuidado com aspas ao interpolar strings de usuário em atributos onclick.

## Financeiro — regras de negócio (não óbvias)

- **`mesRef(l)`**: decide o mês de competência de um lançamento. Receita usa `l.competencia` (não a data). Despesa em **crédito** usa o dia de fechamento da fatura por conta (`CONTAS`: Nubank fecha dia 9, BTG dia 27) — se a compra é depois do fechamento, cai no mês seguinte.
- **`liq(r)`**: valor líquido de imposto de uma receita = `valor * (1 - aliquota/100)`. Toda receita exibida no app passa por `liq()`, nunca `.valor` cru. Despesas não têm ajuste de imposto.
- **Demanda mãe/filha**: `maeId` liga uma "filha" (entrega) à "mãe" (captação). Custo/horas ficam só na mãe; receita de edição em cada filha — evita duplicar totais. Qualquer rollup financeiro sobre `demandas` precisa respeitar isso. `softDel`/`trashRestore` preservam esse vínculo via `rel.filhas`.
- **Prazo**: dois campos ativos, `prazo` (original) e `prazoAjuste` (revisado). Sempre altere status de demanda via `demTransicao()` (nunca `.status=` direto) — é isso que carimba `entregueV1Em`/`entregueEm`/`statusHistory` corretamente, só na primeira transição.
- Aba **Lançamentos** do Financeiro mostra só passado/realizado (`data<=hoje`); aba **Futuros** (nova, do último commit) separa parcelas/assinaturas ainda não vencidas.

## Sync com Supabase (opcional)

- Config fica em `S.prefs` (`sb_url, sb_key, sb_token, sb_refresh, sb_email`), persistida no mesmo blob local, mas **explicitamente removida** (`sanitizePrefs()`) antes de ir para a tabela remota `lozy_settings`.
- Estratégia: diff por registro contra um snapshot do último sync (`lozy-sync-snap`, chave separada), "mais recente vence" via `updated_at`. Auto-sync a cada 2 min com a aba visível, mais em `visibilitychange`/`online`.
- Tabelas remotas esperadas (inferidas do código, **não há `.sql` no repo**): `lozy_records (id, kind, data jsonb, updated_at, deleted)` e `lozy_settings (key, value)`. O app referencia um `supabase-schema.sql` que o usuário deveria rodar no SQL Editor — **esse arquivo não existe no repositório**. Se pedirem para consertar o setup do Supabase, será preciso recriar esse schema a partir de `SYNC_KINDS` e das chamadas em `sbFetch`.
- Sem conexão configurada, o app funciona 100% local (localStorage).

## Integração com IA (chat flutuante)

- Botão `✦` monta um painel de chat (`mountAI()`) que envia contexto resumido (`aiSnapshot()`: leads/demandas/tasks limitados, totais financeiros do mês, eventos da semana) para `POST https://api.anthropic.com/v1/messages`.
- **Não há API key no client** (sem header `x-api-key`) — depende do app rodar dentro do claude.ai, que intercepta/injeta auth no fetch. Fora desse host (ex: GitHub Pages), a chamada falha e a IA degrada silenciosamente para um toast de erro — isso é esperado, não é bug.
- ⚠️ Se algum dia precisar "consertar" a IA para funcionar self-hosted, **não** hardcode uma API key no client — isso seria uma regressão real de segurança (exposição de credencial). A solução correta é um backend/proxy.
- A resposta do modelo é JSON estrito `{"reply":"...","actions":[]}`; `actions[]` (addTask/addDemanda/addLead/addEvento/addReceita/addDespesa) são aplicadas direto em `S` sem confirmação — a IA tem escrita direta no estado.

## Coisas frágeis / a saber antes de mexer

- `uid()` atual é seguro (contador monotônico + sufixo aleatório), mas existe uma ferramenta de auditoria (`auditIds`) porque versões antigas colidiam em bulk-imports — não assuma que todo ID em dados migrados é único.
- Anexos de imagem em observações (Card Drawer) viram base64 dentro do próprio `S`/localStorage, sem limite rígido — pode estourar a quota (~5-10MB) e falhar o `save()` silenciosamente (só loga no console, sem toast visível).
- CSS tem um bloco duplicado (~linhas 110–400 repetem ~320–420 quase idênticas) — inofensivo mas é debt de limpeza, não uma feature.
- Convenções de nomenclatura de função a seguir ao adicionar código: `render<Secao>` para módulos, `open<Entidade>`/`del<Entidade>` para o padrão modal-CRUD, `<modulo>Transicao` para trocas de status que carimbam histórico, prefixos curtos (`com*, fin*, ce*, prop*, calc*, dg*, sb*`) por área.

## O que NÃO fazer

- Não introduzir framework/bundler/build step sem alinhar antes — é uma decisão de arquitetura, não um detalhe de implementação.
- Não "prettificar"/reformatar blocos de código só por estilo — o arquivo é grande e já denso de propósito.
- Não mudar `.status` de demanda/task/pint diretamente — sempre pelas funções `*Transicao()`.
- Não deletar registros direto dos arrays — sempre via `softDel()`.
