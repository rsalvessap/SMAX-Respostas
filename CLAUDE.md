# CLAUDE.md

Orientações para trabalhar neste repositório.

## 1. O que é este projeto

Dois **userscripts Tampermonkey** que automatizam o SMAX (OpenText/Micro Focus Service Management Automation X) do TJSP, em `https://suporte.tjsp.jus.br/saw/*`, mais uma ponte para o eProc (`https://eproc1g.tjsp.jus.br/eproc/controlador.php*`).

```
SMAX/SMAX Respostas - TJSP.user.js        # versão "geral" (usuários)  — ~8.900 linhas
SMAX/SMAX Respostas ADM - TJSP.user.js    # versão "ADM" (administrador) — ~10.400 linhas
README.md                                  # manual do usuário final (versão geral)
```

**O ADM é um superset da geral.** Os dois arquivos são cópias quase idênticas, mantidas em paralelo — não há build, bundler, dependências nem testes. O diff entre eles é de ~1.750 linhas adicionadas / ~250 removidas e concentra-se em: editor de equipes, publicação de config no GitHub, Script Picker redesenhado, popup do solicitante, botão Extrair e destaque de solicitantes.

> Ao alterar algo que vale para os dois, **replique a mudança nos dois arquivos** e faça o bump de versão de ambos (o padrão dos commits é `... (ADM v1.X / Regular v1.Y)`).

### Versionamento

Cada arquivo tem **duas** strings de versão que precisam andar juntas:

| Local | ADM | Geral |
|---|---|---|
| `// @version` (cabeçalho) | `1.33` | `1.11` |
| `SMAX_TOOLKIT_VERSION` (linha ~37) | `1.31` ⚠️ | `1.11` |

⚠️ Hoje o ADM está dessincronizado (header 1.33, constante 1.31). O `README.md` também declara "Versao atual: 1.7" enquanto a geral está em 1.11. Distribuição é via `@downloadURL`/`@updateURL` apontando para `raw` do branch `master` — **push no master publica para todos os usuários**.

## 2. Arquitetura

Arquivo único, IIFE global. Guardas de entrada logo no topo:

```js
if (window.top && window.top !== window.self) return;              // sem iframes
if (window.location.hostname !== 'suporte.tjsp.jus.br') return;    // só no SMAX
```

Por isso o **eProc Bridge fica em um segundo IIFE**, depois do fechamento do principal (ADM ~10.375), atrás de `if (hostname === 'eproc1g.tjsp.jus.br')`.

Módulos em ordem no arquivo (linhas do ADM; na geral o offset é ~-100 no início e cresce até ~-1.500 no fim):

| # | Módulo | ADM | Camada |
|---|---|---|---|
| 1 | `PrefStore` | 47 | storage |
| 2 | `PersonalStore` | 131 | storage |
| 3 | `SignatureManager` | 176 | domínio |
| 4 | `ThemeManager` | 236 | UI |
| 5 | `ActivityLog` | 266 | auditoria |
| 6 | Styles (`GM_addStyle`) | 528 | UI |
| 7 | `Utils` | 1057 | infra |
| 8 | `ApiClient` | 1538 | rede |
| 9 | `TeamsConfig` | 1722 | domínio |
| 10 | `DataRepository` | 1880 | cache |
| 11 | `Network` (patch XHR/fetch) | 2741 | rede |
| 12 | `Api` (escritas reais) | 2829 | rede |
| 13 | `AttachmentService` + `AttachmentPreviewer` | 3178 | domínio |
| 14 | `SettingsPanel` | 3666 | UI |
| 15 | Mapas de status compartilhados | 5472 | domínio |
| 16 | `ResponseHUD` | 5565 | UI (maior módulo, ~4.400 linhas) |
| 17 | `Templates` | 9949 | domínio |
| 18 | `SharedConfig` | 10174 | sincronização |
| 19 | `boot()` | 10357 | — |
| 20 | eProc Bridge (IIFE separado) | 10375 | integração |

`boot()` ordem: `ThemeManager.init()` → `SettingsPanel.init()` → `ResponseHUD.init()` → `Templates.init()` → `SharedConfig.init()` → `DataRepository.refreshQueueFromApi()` + `ensureSupportGroups()` (ambos best-effort).

### Chaves de armazenamento

| Chave | Store | Conteúdo |
|---|---|---|
| `smax_prefs` | GM | config da equipe (equipes, assinaturas de equipe, ack, URL shared, token GitHub, identidade) |
| `smax_personal_prefs` | GM | `myDestaque`, `themeMode`, `personalSignatures` |
| `smax_activity_log` | GM | log de auditoria (máx. 5.000 entradas, poda de 150) |
| `smax_solutions_v2` / `smax_discussions_v2` | GM | scripts locais de solução / discussão |
| `smax_resp_filters` | GM | último estado de filtros do HUD |
| `smax_resp_filter_presets_v1` | GM | presets salvos de filtro |
| `smax_shared_cache` | GM | cache do SharedConfig (TTL 1h) |
| `smax_gerenciador_equipe_id` | GM | equipe no Gerenciador de Chamados (Supabase) — **escrita por outro script** |
| `smaxTenantId` | session/localStorage | tenant resolvido |
| `eproc_smax_bridge_proc` | sessionStorage (eProc) | processo pendente de consulta |

### Serviços externos

- **API SMAX** (mesma origem, `credentials: 'include'`): `ems/bulk`, `ems/Request`, `ems/Person`, `ems/PersonGroup`, `ems/Attachment`, `ems/RequestCausesRequest`, `entity-page/...`, `frs/file-list/...`. Tenant `213963628`.
- **Supabase** (`rlcbmrjkojopipiwpktf.supabase.co`, anon key embutida): tabela `smax_activity_log` (relatório) e `scripts_customizados` + `equipes` (scripts do Gerenciador de Chamados).
- **GitHub raw**: `shared-config.json` em `rsalvessap/SMAX-TOOLS@master`, lido por todos.
- **GitHub API** (`api.github.com`): **só no ADM** — escrita do `shared-config.json`.

## 3. Inventário de funcionalidades

### 3.1 PrefStore / PersonalStore
Duas stores propositalmente separadas: `PrefStore` guarda o que é **compartilhável com a equipe**; `PersonalStore` guarda o que é **individual** e nunca entra no export. Ambas fazem migração automática de formatos antigos (`ackMessageTemplate` HTML→texto; `myDetratores`→`myDestaque`; `string[]`→`{id,name}[]`).

### 3.2 SignatureManager
Funde assinaturas **de equipe** (`prefs.teamSignaturesRaw`, mapa `teamId → html`) com **pessoais** (`personal.personalSignatures`, `{name, html}[]`) numa lista única. Insere em `contenteditable` (`execCommand('insertHTML')`) ou CKEditor (`setData` concatenado). Evita duplicação comparando os primeiros ~60 caracteres do texto já presente.

### 3.3 ThemeManager
Três temas cíclicos: `dark` → `gray` → `light`. Aplica via `dataset.smaxTheme` no `<html>` e `<body>`; todo o CSS é escrito em cima de tokens `--sp-*`.

### 3.4 ActivityLog
Registra cada ação relevante com `relevantWork` derivado por prioridade: `RESPONDIDO > VINCULO_GLOBAL > TRANSFERIDO > DESIGNADO > STATUS > OUTRO`. Grava local **e** faz POST assíncrono no Supabase (`Prefer: return=minimal,resolution=ignore-duplicates`, timeout 8s). Degradação graciosa: se o POST falhar com HTTP 400 por causa da coluna `ticket_subject`, refaz sem ela. Exporta CSV com BOM UTF-8.

### 3.5 Utils
~27 helpers. Os não-óbvios:

- **`linkifyCNJ` / `openEprocProcess`** — varre o DOM (tree walk que pula `<a>`, `<script>`, `<style>`), detecta CNJ formatado (`4000439-14.2026.8.26.0201`) ou 20 dígitos crus, cria spans clicáveis com `data-smax-proc` e um único listener delegado em `document` (capture). Ao clicar, abre aba do eProc e envia `postMessage` **três vezes** (800ms/2s/4s) para cobrir o tempo de carga.
- **`normalizeContentEditableHtml`** — o SMAX trunca campos rich-text grandes (converte para link server-side). Esta função achata `<div>`/`<p>` em `<br>`, remove markup Office (`o:p`, `mso-*`) e mantém só formatação inline + imagens.
- **`sanitizeRichText`** — whitelist de tags, remove `on*`, `style` inline e URIs `javascript:`.
- **`pushSolutionHtml`** — injeta no editor com retry (10× a cada 250ms) porque o CKEditor do SMAX instancia tarde.
- **`parseDigitRanges` / `digitsToRangeString`** — `"1,2,5-10"` ↔ `[1,2,5..10]`, usado na distribuição de chamados por dígito.

### 3.6 ApiClient
Resolve o `tenantId` de URL, hash, cookie ou `sessionStorage`. Monta `/rest/{tenant}/ems/...`, injeta XSRF-TOKEN, aplica timeout via `AbortController`.

### 3.7 TeamsConfig — roteamento de chamados
`suggestTeam(ticket)` decide a equipe com esta **precedência**:

1. GSE (`ExpertGroup`) presente em `team.gseRules[]` → match direto.
2. Matchers regex, por escopo: `scope:'location'` testa só `RegisteredForLocation`; `scope:'text'` testa assunto + descrição; sem escopo testa o blob concatenado (`gseName | locationName | descriptionText | subjectText | descriptionHtml`, tudo em maiúsculas).
3. Fallback por containment do `team.id` no nome da GSE.
4. Equipe com `isDefault: true` (ou a primeira da lista).

`suggestWorker(team, ticketId)` pega os últimos dígitos do ID e procura o worker cujo range cobre aquele número; se o worker estiver `isAbsent`, tenta o par de dígitos seguinte. As equipes vindas do SharedConfig são marcadas `_shared: true` e **nunca** são persistidas em `prefs.teamsConfigRaw`.

### 3.8 DataRepository
Caches em memória com eviction LRU: `triageCache` (400 chamados, poda 150), `peopleCache` (600 pessoas, poda 200), `supportGroupMap`. Sem TTL — vivem na sessão. Expõe `ingest*Payload()` (usados pelo Network patch) e `ensure*` (carregamento sob demanda, paginado).

### 3.9 Network patch
Monkey-patch de `XMLHttpRequest.prototype.open/send` e `window.fetch`. Quando a **UI nativa do SMAX** busca `ems/Request`, `ems/Request/{id}`, `ems/Person`, `ems/PersonGroup` ou `ems/RequestCausesRequest`, o script intercepta a resposta e alimenta os caches. Isso evita refetch e mantém o HUD em sincronia com a tela nativa sem polling.

### 3.10 Api — escritas reais
**Todas as escritas passam por `prefs.enableRealWrites`**; se `false`, retornam `{skipped: true}`.

| Operação | Endpoint | Payload |
|---|---|---|
| `postUpdateRequest(props)` | `ems/bulk` | `{entities:[{entity_type:'Request', properties}], operation:'UPDATE'}` |
| `postCreateRequestCausesRequest(pai, filho)` | `ems/bulk` | `relationships` + `operation:'CREATE'` |
| `postDiscussion(...)` | `ems/bulk` | UPDATE em `Request.Comments` (JSON string) |
| `postAddFollower` / `postRemoveFollower` | `ems/bulk` | relacionamento `FollowedByUsers`, CREATE/DELETE |

O payload de discussão foi **obtido por engenharia reversa do tráfego da UI nativa** (ver comentário perto de ADM:3102). Pontos críticos:

- O campo `Comments` é o **histórico inteiro** serializado, não só o comentário novo — é preciso concatenar os existentes.
- `CommentId` é gerado (36 hex aleatórios).
- Imagens base64 acima de 50 KB são recomprimidas via Canvas (máx 1200px, quality 0.7).
- Se o JSON total passar de ~60 KB, base64 de comentários **antigos** é substituído por um GIF de 1px para caber.
- `postAddFollower` tenta duas ordens de endpoint porque o modelo do SMAX varia.

### 3.11 AttachmentService / AttachmentPreviewer
Descobre anexos por várias rotas, em cascata: `entity-page/initializationDataByLayout/Request/{id}` → `ems/Attachment?filter=ParentEntity.Id=...` → `ems/Attachment/{id}` (campo `file_list`) → `frs/file-list/{id}`. Cada anexo carrega um array `downloadCandidates` testado em ordem até um funcionar. Preview: modal com navegação ←/→ e ESC para imagens; PDF abre em nova aba; demais tipos disparam download. Cache + dedupe de requisições em voo.

### 3.12 SettingsPanel
Sete seções (`SECTIONS`, ADM:3676): **Geral**, **Equipes**, **Especialistas**, **Destaque**, **Scripts**, **Respostas**, **Assinaturas**.

- **Geral** — identidade (`myPersonId`/`myPersonName`), URL do SharedConfig, mensagem de recebimento, export/import de config, publicação no Git.
- **Equipes** — no ADM, editor completo; na geral, lista somente-leitura.
- **Especialistas** — agregação read-only dos workers de todas as equipes.
- **Destaque** — solicitantes marcados com ⭐. Busca híbrida: substring nos caches locais (imediata) + busca na API por prefixo (debounce 400ms, com guarda de sequência para descartar respostas obsoletas).
- **Scripts** — CRUD de templates de solução/discussão, importação do Gerenciador (Supabase) e do SharedConfig, import/export JSON, "Do clipboard".
- **Respostas** — abre o ResponseHUD, configura os botões de encaminhamento rápido (`forwardingButtonsRaw`) e gera o Relatório de Atividades.
- **Assinaturas** — CRUD das assinaturas pessoais com preview ao vivo.

### 3.13 Mapas de status
`REQUEST_STATUS_LABELS` (~24 entradas, ex. `RequestStatusInProgress` → "Em Andamento") e `STATUS_SCCD_LABELS` (60+ entradas TJSP, ex. `AguardandoCliente_c`, `EmAtendimento_c`). `humanReadableStatus()` consulta o primeiro e cai no segundo.

### 3.14 ResponseHUD
O módulo central. **Painel esquerdo:**

- Filtros combináveis por **equipe**, **Status**, **Status Operacional**, **Especialista** e **texto livre** (busca em ID, descrição, solicitante, localização). Regra: `Set` vazio = filtro desligado, tudo passa. Os filtros são AND entre si, OR dentro de cada um.
- **Presets** em `smax_resp_filter_presets_v1`, mais o preset fixo `__rejected__` ("🔴 Rejeitados": status "Aguardando Atendimento" + `RequestStatusReady` + meu usuário). Presets com `useMyAssignee` injetam `prefs.myPersonId`.
- **Ordenação** por `id | createTime | status | assignee`, asc/desc.
- **Scroll virtual**: item fixo de 66px, overscan de 8, spacers acima/abaixo, re-render via `requestAnimationFrame`.
- **Seleção em lote** via `Set selectedTicketIds` (sem limite explícito).
- Badges: **VIP**, **Global pai** ("🌐 Global (N filhos)"), **Global filho** ("⬆ Global #id"), **⭐ Destaque**.

**Painel direito** — chips de ação, cada um acumulando estado pendente antes do envio:

| Chip | Efeito |
|---|---|
| GSE | altera `AssignedToGroup`; variante "com encaminhamento" posta discussão com texto configurável |
| Especialista | altera `ExpertAssignee` |
| Status | altera `Status` (por chamado, em `pendingStatusByTicket`) |
| Status Operacional | altera `StatusSCCDSMAX_c` (em `pendingStatusSCCDByTicket`) |
| Seguidor | adiciona/remove seguidores |
| Seguir | auto-adiciona como seguidor |
| Recebimento | agenda discussão pública com `ackMessageTemplate` |
| Escalar | move o chamado de Validação para Atendimento |

**Editor de solução**: negrito/itálico/sublinhado/tachado, listas, indent, `<hr>`, link/unlink, tamanho de fonte, cor de texto e fundo, autoformat, seletor de assinatura, seletor de scripts e três códigos de finalização (`CompletionCodeFulfilled`, `...FulfilledByLiveSupport`, `...IncidentResolved`).

**Discussões**: lista com autor, data, badge PUBLIC/INTERNAL; "⤢ Expandir" (modal com prev/next e zoom de fonte) e "↺ Replicar" (copia para o editor). Nova discussão tem editor próprio + seletor de destinatário (Agent/User/Vendor/ExternalServiceDesk/Stakeholder) e de objetivo (StatusUpdate/FollowUp/Resolution/...). Inserção otimista no `triageCache` após o POST.

**Fluxo de ENVIO** — alvo é `selectedTicketIds` (lote) ou `activeTicketId` (único). Lote passa por modal de confirmação com tabela por chamado. Ordem por chamado:

1. `postUpdateRequest` com solução + completion code + GSE + especialista + status + status operacional
2. seguidores (add/remove)
3. escalação (`PhaseId: 'Validation'`)
4. discussão de recebimento
5. vínculo global (`RequestCausesRequest`)

### 3.15 Templates
Modelo: `{ title, html, commentTo?, purposeCode?, _shared? }`. Persistência em GM storage com migração automática vinda de `localStorage` e do formato antigo `{content}` → `{html}`. `loadAll()` funde locais + compartilhados; compartilhados são read-only.

### 3.16 SharedConfig
Busca `prefs.sharedConfigUrl` (GitHub raw) via `GM_xmlhttpRequest` com cache-busting `?_t=`, TTL de 1 hora, cache local em `smax_shared_cache` como fallback. Schema:

```jsonc
{
  "_version": 1, "_updatedAt": "2026-09-11", "_description": "...",
  "teams": [ { "id", "name", "priority", "isDefault", "gseRules": [], "matchers": [], "workers": [] } ],
  "teamSignatures": { "<teamId>": "<html>" },
  "scripts": { "sol": [], "disc": [] },
  "nameGroups": {}, "ausentes": [],
  "enableRealWrites": true, "defaultGlobalChangeId": "", "ackMessageTemplate": ""
}
```

Distribui: equipes (`TeamsConfig.setSharedTeams`), scripts (`Templates.setSharedScripts`), assinaturas de equipe, ausentes, `nameGroups`, `defaultGlobalChangeId` e `ackMessageTemplate`.

### 3.17 eProc Bridge
IIFE separado que roda só em `eproc1g.tjsp.jus.br`. Recebe `{type: 'SMAX_CONSULTAR_PROCESSO'}` por `postMessage` (origin restrito a `https://suporte.tjsp.jus.br`), normaliza o CNJ, e preenche `#txtNumProcessoPesquisaRapida` usando o **setter nativo** de `HTMLInputElement.prototype.value` (para o framework do eProc detectar a mudança), dispara `input`/`change`/`Enter` e submete o form. Fallback para formulários internos (`#txtNumProcesso`, `input[name*="processo"]`, botão com "pesquisar|consultar|buscar|localizar"). Guarda o número em `sessionStorage` para retomar após redirect.

## 4. Exclusivo do ADM

Verificado por diff completo entre os dois arquivos. Nada abaixo existe na versão geral.

| # | Funcionalidade | ADM | Como funciona |
|---|---|---|---|
| 1 | **Editor de Equipes** (`renderTeamEditor`, `wireTeamEvents`) | 3775, 3893 | Criar/editar/remover equipes: nome, GSEs, palavras-chave por Local de Registro e por Assunto/Descrição, membros, assinatura da equipe. Na geral a aba é read-only com o aviso "Gerenciado pelo ADM". |
| 2 | **Busca de GSE no editor** | ~4030 | Autocomplete sobre `DataRepository.getSupportGroupsSnapshot()`, dispara `ensureSupportGroups()` no focus. |
| 3 | **`DataRepository.searchPeopleRemote`** | 2658 | Busca global de pessoas no SMAX. O SMAX **não suporta `LIKE`/`%` em `Person`** — usa range de prefixo (`Name >= 'X' and Name < 'Y'`), mínimo 3 letras, debounce 500ms. |
| 4 | **Export/Import de configuração** (`CONFIG_KEYS`, `buildConfigJSON`, `applyConfigJSON`) | 4191–4265 | Serializa 7 chaves compartilháveis. `ausentes` é **derivado** dos flags `isAbsent` dos workers (fonte única da verdade), não armazenado direto. Exclui de propósito: `myPersonId`, `myPersonName`, `githubToken`, `forwardingButtonsRaw`. |
| 5 | **Publicar para a equipe (Git)** (`publishConfigToGit`) | 4268 | Faz o parse de `sharedConfigUrl` (`raw.githubusercontent.com/{owner}/{repo}/{branch}/{path}`), GET no `contents` da API para pegar o SHA, faz merge preservando campos de outros scripts, incrementa `_version`, e PUT em base64 com mensagem `chore: atualiza shared-config SMAX Toolkit v{n}`. Autentica com PAT em `prefs.githubToken`. Requer `@connect api.github.com`. |
| 6 | **URL do SharedConfig editável** + botão Salvar | ~4380 | Na geral a URL é fixa e só exibida. |
| 7 | **Mensagem de Recebimento editável** | ~4392 | Textarea + salvar + restaurar padrão. Na geral é exibição read-only. |
| 8 | **Botão "📥 Extrair"** | 8734, 9392 | Copia o chamado inteiro para o clipboard em **dois formatos simultâneos** (`ClipboardItem` com `text/html` + `text/plain`), pensado para colar em uma IA. Busca a lista de anexos por `AttachmentService.fetchList()`, baixa cada imagem com `credentials:'include'` e converte em `data:` base64 — tanto as `<img>` inline da descrição/discussões quanto os anexos. **O ChatGPT descarta imagens vindas do flavor `text/html`**, então cada imagem também é numerada com um marcador `[IMAGEM N]` no texto e **baixada como arquivo** `chamado-{id}-img{N}.{ext}` para anexar manualmente no chat. Inclui todas as discussões, inclusive as `systemGenerated` (bate com o painel). Fallback para texto puro se a extração rica falhar. |
| 9 | **Script Picker reformulado** | 785–890 (CSS), 6952–7330 | Três views (**grid** / **rows** / **tiles**), chips de equipe coloridos (`SMAX_SP_TEAM_COLORS`), dropdown multi-select de **Assuntos** (extraídos do prefixo do nome antes de `. `, ` — `, ` - `, ` – `), count pill "N de Total", painel de preview estruturado e hover card com delay de 180ms. A geral ainda usa a lista simples de duas colunas. |
| 10 | **`_team` nos scripts** | 6888–6940 | Resolve o nome da equipe do script via PostgREST resource embedding (`equipes(nome)`); fallback carrega `equipes` ou `gerenciador_equipes` e resolve os IDs manualmente. Filtra valores que ainda são UUID. |
| 11 | **Popup de dados do solicitante** | 8687, 9017 | Nome do solicitante vira link. Ao clicar, busca `ems/Person/{id}` com layout estendido e mostra cargo, matrícula, e-mail, telefone, celular, localização, organização e VIP. Cada linha é click-to-copy. Cai para o `peopleCache` se a API falhar. |
| 12 | **Destaque (⭐) funcional** | 5838, 682–683 (CSS) | Marca a linha na lista e o detalhe. Match por ID **ou** nome exato (sem match parcial — intencional). Na geral o código está marcado como "implementação pendente". |
| 13 | **Busca de pessoas no Destaque por prefixo** | 4848 | ADM: local imediato + API por prefixo com debounce e guarda de sequência. Geral: pagina `ems/Person` inteiro (200/página) para um cache em memória. |
| 14 | **Diagnóstico de payload em `postDiscussion`** | 3079 | `console.info` com tamanhos de `bodyHtml`, nº de comentários existentes e tamanho do JSON, para caçar truncamento do SMAX. |
| 15 | **Filtro padrão "Em Atendimento"** | 8950 | Na geral o padrão é "Aguardando Atendimento". |

### Só na versão geral (removido no ADM)

- `SharedConfig.getImportLog()` e o painel "Detalhes da importação" na aba Geral — o ADM só loga no console (`console.group('[SMAX SharedConfig ADM] ...')`).
- Exibição read-only das assinaturas de equipe e da mensagem de recebimento na aba Geral.
- Aviso "Gerenciado pelo ADM" na aba Equipes.

## 5. Trabalhando no código

- **Sem build.** Editar o `.user.js` é editar o artefato final. Para testar, recarregue o script no Tampermonkey e a página do SMAX.
- **Sem testes.** Mudanças de UI precisam ser validadas manualmente contra o SMAX real.
- Estilos ficam em dois lugares: o bloco `GM_addStyle` (tokens `--sp-*`, classes `.smax-*`) e **inline styles em template literals** por todo o código. Ao mexer em layout, procure os dois — vários bugs recentes ("inline styles residuais", "cards vazios") vieram dessa duplicação.
- Sempre escape conteúdo dinâmico com `Utils.escapeHtml()` e HTML vindo do SMAX com `Utils.sanitizeRichText()`.
- Mensagens de commit: `tipo: descrição curta (ADM v1.X / Regular v1.Y)`, em português.

### Armadilhas conhecidas

- `Person` no SMAX não aceita `LIKE`/`%` — use range de prefixo (`>=` / `<`).
- O campo `Comments` precisa ser reenviado com o histórico completo; enviar só o comentário novo apaga os anteriores.
- O SMAX trunca rich-text grande (converte em link server-side). Daí a compressão de imagens e o downgrade de base64 antigo para GIF de 1px.
- O CKEditor da página nativa instancia tarde — `pushSolutionHtml` já faz retry, não assuma disponibilidade imediata.
- Equipes com `_shared: true` nunca podem ser gravadas em `prefs.teamsConfigRaw`, senão viram cópia local divergente.
- A anon key do Supabase e o tenant ID estão no código por design (a key já é pública no bundle do app). **O PAT do GitHub não** — ele fica só em `GM_setValue`, nunca commite um.
