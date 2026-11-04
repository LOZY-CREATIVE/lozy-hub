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

### Vínculo demanda ↔ projeto

- `demanda.projetoId` liga uma demanda a um `S.projetos` específico (produção de cliente). Sem isso, duas produções do mesmo cliente "vazavam" demandas uma pra outra (tudo era inferido por `cliente===cliente`). `demandasDoProjeto(p)` é a função central: usa `projetoId` quando setado; demandas antigas sem o campo (migradas antes dele existir) só aparecem no projeto do cliente quando esse cliente tem **um único** projeto — em caso de ambiguidade, ficam sem projeto até alguém linkar manualmente (não assume, não deixa vazar). Use sempre `demandasDoProjeto()`/`demProjetoOpts()`, nunca volte a filtrar demandas de um projeto por `cliente===`.
- **Seletor de "Demanda vinculada" com busca** (`demSelHtml`/`demSelFiltra`/`demSelEscolhe`): substitui o `<select>` nativo com todas as demandas (ficava inviável conforme a base cresce) por um campo de texto + lista flutuante — sem busca mostra as 10 mais recentes. Usado em Tarefas, Financeiro e no Card Drawer (tasks/fin). Nos formulários do modal único o valor só é lido no submit via o input oculto (mesmo `id` do antigo `<select>` — os handlers de submit não mudam); no Card Drawer (commit imediato) passe `cb:'cdSetDemanda'`. **Não existe pro seletor "Demanda mãe"** (openDem `m6` / Card Drawer) — ficou de fora de propósito, a lista de candidatas a mãe já é menor (exclui quem já é filha).

### Proposta V2 — seções liga/desliga

- `p.secoes` (normalizado por `propSecoes()`) decide o que aparece na proposta do cliente: `cliente, partida, escopo, mensal, crono, cond, notas` + `valoresItens` (mostrar preço por item ou só o total). `PROP_PRESETS` traz dois pontos de partida — `essencial` (só valor) e `completa`.
- **Propostas salvas antes disso não têm `secoes`** — `propostaHTML2` cai no padrão "tudo ligado" (`mensal` seguindo o antigo `p.recorrente`), então nada muda retroativamente. Não remova esse fallback.
- **A numeração das seções é derivada**, não fixa: `propostaHTML2` monta um array `blocos` com o que sobrou ligado e numera na ordem. Ao adicionar uma seção nova, entre nesse array — números hardcoded voltariam a sair furados ("01, 04, 06").
- `PROP_TEXTOS_PADRAO` cobre título/descrição de cada seção **e** os rótulos internos (`inv_*`, `rot_*`, `rodape_*`). Texto novo visível ao cliente deve entrar aí, não hardcoded no template.
- Perfis salvos (`S.perfis`) guardam `textos` **e** `secoes`.

### Animação de entrada da proposta (armadilha séria)

- O `.reveal` da proposta gerada é **progressive enhancement**: o conteúdo nasce visível no CSS, e o script só adiciona `js-anim` no `<html>` — que aí sim esconde tudo pra revelar via `IntersectionObserver` — **quando `document.visibilityState==='visible'`**.
- Motivo: por muito tempo `.reveal` nascia com `opacity:0` e dependia do observer pra voltar. Em qualquer contexto onde o observer não roda (aba em segundo plano, iframe de preview, painel de render, conversor de PDF) o cliente recebia **a proposta em branco, só com o rodapé** — o rodapé era o único elemento fora do `.reveal`. Em aba oculta os timers também são estrangulados, então `setTimeout` como rede de segurança não resolve.
- Nunca inverta isso de volta. Qualquer bloco novo da proposta deve ser legível sem JS.
- Relacionado: **não use `overflow-x` no `body`** do template. Em `body`, `overflow-x:hidden` força `overflow-y:auto`, o body vira contêiner de rolagem e o observer para de disparar mesmo em aba visível.

### Marca embutida na proposta

- `PROP_LOGO_URL` e `PROP_SIGIL_URL` são o logo e o símbolo Lozy em WebP base64 (~39 KB somados), usados **como máscara CSS** (`.mark` e `.sigil`) — a cor vem de `var(--accent)`, então um único desenho serve wine e terracotta. Não troque por `<img>` colorido: dobraria o payload e quebraria o vínculo com a cor de destaque.
- Ficam embutidos porque a proposta gerada é um **arquivo único** que vai por e-mail/WhatsApp — referenciar arquivo externo quebraria na mão do cliente.
- Os dois são `display:none` por padrão e só aparecem dentro de um `@supports` de máscara: sem suporte, some — melhor que um retângulo sólido no lugar do logo.
- Fonte dos assets: `~/Desktop/lozy_creative/business/BRAND/LOZY_NEW/{LOGO,SYMBOL}/*.png` (2000px e 8334px). Para regerar: recortar pelo bbox do alpha, redimensionar, `cwebp -lossless`, base64.

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
- **`dataVencimento(l)`**: separado de `mesRef` de propósito — `mesRef` decide o MÊS (competência), `dataVencimento` decide o DIA em que a cobrança fica exigível, e é isso (não `l.data` cru) que `prazoStatus()` usa pra marcar "atrasada" em despesas de crédito (Visão geral, Contas, Centro Estratégico). Sem essa distinção, uma compra parcelada aparecia "atrasada" contando a partir da data da compra, mesmo com a fatura ainda aberta. Usa `CONTAS[].venc` (Nubank vence dia 16, BTG dia 5 do mês seguinte) — a função decide sozinha se o vencimento cai no mesmo mês do fechamento ou no seguinte, comparando `venc` com `fech`.
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

## Financeiro vindo do Fin Pessoal (Ajustes → "Financeiro vindo do Fin Pessoal")

- O dado cru financeiro da Lozy passa a nascer no **Fin Pessoal** (importação de extrato/faturas). O Hub importa um *pacote Lozy* (JSON gerado lá) por `hubLerPacote`/`hubAplicarPacote`: cada lançamento mantém o **mesmo `id`**, `fonte:'fin-pessoal'`. Reimportar atualiza só os campos crus (`FP_CRU`: valor, data, categoria, pago, `desc`…) e **preserva** o que é do Hub (`demandaId`, `cliente` já preenchido, alíquota, observações criadas aqui — as do Fin Pessoal levam `origem:'fin-pessoal'`). Lançamentos digitados no Hub (sem `fonte`) nunca são tocados; os que sumiram do pacote vão para a Lixeira (opção marcada por padrão).
- `mesRef()` e `dataVencimento()` usam `mesRefFP`/`vencFP` quando `fonte==='fin-pessoal'` (os ciclos reais de fechamento do cartão vivem no Fin Pessoal; `CONTAS` do Hub não os conhece). `parcInfo` considera `lancadasAntes`. `autoLancarPendentes` **pula** parcelamentos/assinaturas com `fonte:'fin-pessoal'` (as parcelas já chegam lançadas).
- `hubZerarFinanceiro()` é a única exceção consciente à regra "nunca deletar direto": pedido explícito do usuário, baixa o backup completo antes, dupla confirmação e `pushUndo`. Marca `S.prefs.finFonte='fin-pessoal'` — com isso `load()`/`applySeed()` **não** recarregam o financeiro da carga do Notion (`finDoFP()`); sem a flag, zerar `parc`/`assin` seria desfeito no próximo carregamento.
- Privacidade: o arquivo só tem o que é Lozy (lista fixa de campos). Não há Supabase nesse caminho ainda; quando houver, o desenho previsto é dois projetos separados (pessoal × equipe) — nunca o mesmo projeto, porque a RLS atual dá acesso total a qualquer usuário logado.

### Financeiro do Hub — paridade com o Fin Pessoal (leitura e vínculo)

- **Ordenação** (Contas, Por cliente, Recorrentes): handlers inline NÃO enxergam variáveis locais do render; o padrão `onchange="S.sort.x={...srt,…}"` dava ReferenceError silencioso. Use `ordCampo(chave,padrao,valor)`/`ordInverter(chave,padrao)`.
- **Recorrentes**: parcelamentos e assinaturas abrem em "em andamento" (`S.f.finRec.status`/`sa` indefinidos valem `andamento`; opções `quitado|todos` e `encerradas|todas` — não use `''` para "todos"). Busca por nome, ordenar por valor da parcela/total/data da compra. "Encerrada" = `ativo:false`. Parcelamentos com `fonte:'fin-pessoal'` não entram na auditoria de datas (`parcAudit`), e `projecaoFixos` soma `lancadasAntes` (senão projetaria parcelas já pagas).
- **Lançamentos**: coluna/filtro de **subcategoria**, coluna **Demanda** (`demTituloFin`), filtro **Vínculo** (sem demanda / sem cliente / sem os dois / com demanda) e **ligar em lote** (`S.f.finSel`, `finSelTodos`, `finLigarLote`): põe `demandaId` e/ou `cliente` nos selecionados, sem tocar nos campos crus. Desfazível.
- **Por cliente**: agrupar por Cliente | Projeto | Demanda (`S.f.finCliG`; projeto = projeto da demanda ligada, `projDeLanc`). **Análises** ganhou as dimensões Demanda, Projeto e Subcategoria.
- **Visão geral personalizável** (`S.prefs.visao={ordem,off}`, `visaoMontar`/`visaoPers`): blocos `kpis, onde, venc, receber, nota`; "A receber" (`recHubCardHtml`) lista receitas `pago:false`. Previsão criada só no Hub não casa com o extrato — o ideal é cadastrá-la no Fin Pessoal como Lozy. Textos de ajuda usam `dica(id,html,titulo)` (recolhidos por padrão).
- O que NÃO foi trazido de propósito: Classificar, Importar e a estimativa da fatura aberta (dado bancário/pessoal mora no Fin Pessoal).

