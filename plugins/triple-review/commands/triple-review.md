---
description: Revisão pré-build em paralelo — QA (funcional, fronteira, bloqueios de negócio e falhas silenciosas) · Sênior (design, corretude, convenções e manutenibilidade) · Gerente UX (fluxo de usuário) — com baseline/delta por branch, consolidação determinística e veredito orientado ao que mudou. Uso: /triple-review [descrição do fluxo]
---

# Triple Review

Revisão pré-build em paralelo: **QA** · **Sênior** · **Gerente UX** — consolidada, deduplicada, com **baseline/delta por branch** e **veredito determinístico**.

O ciclo converge porque cada rodada tem **memória**: os achados desta rodada são comparados com o baseline da rodada anterior (na mesma branch) e cada um recebe uma **tag de origem objetiva** — `NOVO` / `REGRESSÃO` / `PRÉ-EXISTENTE` / `PERSISTENTE` / `RESOLVIDO`. Assim, corrigir tudo e rodar de novo **não** ressurge dívida pré-existente como se fosse problema novo.

## Uso

```
/triple-review [descrição do fluxo de usuário]
```

**Exemplos:**
```
/triple-review
/triple-review inspeção — inspetor abre a ficha do equipamento, clica em Reinspecionar, preenche novos dados e salva
/triple-review reserva de material — operador aprova a reserva e registra a entrega à fábrica
```

---

## Passo 1 — Coletar diff e detectar contexto

> **Economia de contexto:** nunca imprima o diff completo no contexto do orquestrador. Aqui use apenas `--stat`/`--name-only`; o diff completo vai direto para um arquivo no Passo 2, e os três agentes o leem de lá (cada um no seu próprio contexto isolado).

**Arquivos não rastreados:** rode `git status --porcelain` e verifique linhas `??`. Arquivo novo sem `git add` **não aparece no diff** e passaria sem revisão. Para os que forem código do projeto (**ignore artefatos gerados**: vendor, node_modules, builds, **e o diretório de baselines `docs/triple-review-tuning/.baselines/` — nunca inclua o baseline na revisão**), rode `git add -N <arquivos>` — marca a intenção sem stagear conteúdo e os faz aparecer no diff. **Guarde a lista como `ARQUIVOS_ADD_N`**: o Passo 2 desfaz a marcação logo após gravar o diff, e a mensagem do Passo 4 + o relatório informam quais arquivos foram incluídos.

**Determinar a base da revisão** — um único diff que cobre **commits locais + worktree** (nunca revise só o worktree quando existem commits não integrados: isso geraria falso "Cobertura: COMPLETA"):

1. `REF="@{u}"` se `git rev-parse --verify -q '@{u}'` funcionar; senão `master` se o branch existir; senão `main`; senão `REF="HEAD"`
2. `BASE=$(git merge-base $REF HEAD)` (com `REF="HEAD"`, `BASE=HEAD`)
3. `CMD_DIFF="git diff $BASE"` e `PATHSPEC="-- . ':(exclude)docs/triple-review-tuning/.baselines/'"`

> O pathspec de exclusão é obrigatório: se o baseline for commitado (usuário que ignorou a recomendação de `.gitignore`), ele passa a ser rastreado e entraria no diff — os 3 agentes revisariam um JSON de dados, gastando tokens e gerando achados espúrios sobre o próprio baseline. A regra "nunca incluir o baseline na revisão" vale para **os dois** caminhos: `git add -N` (não rastreado) e `CMD_DIFF` (rastreado).
>
> ⚠️ **Ordem obrigatória: `{CMD_DIFF} <flags> {PATHSPEC}`.** Todas as flags (`--stat`, `--name-only`, `--unified=3`) vêm **antes** do `--`. Qualquer coisa depois do `--` é interpretada como caminho, não como flag: `git diff BASE -- . ':(exclude)…' --name-only` **não** lista nomes — ele trata `--name-only` como um path inexistente e imprime o diff completo. Nunca embuta o pathspec dentro de `CMD_DIFF`.

Se `{CMD_DIFF} --stat {PATHSPEC}` vier vazio, informe ao usuário e pare:

> "Nenhuma mudança detectada (base da comparação: [REF]). Possíveis causas:
> 1. Working tree limpo e sem commits locais além da base
> 2. Todas as mudanças já foram integradas ao branch base"

**Capturar lista de arquivos** (saída curta — note as flags **antes** do `{PATHSPEC}`):
```bash
{CMD_DIFF} --name-only {PATHSPEC}          # → LISTA_ARQUIVOS_TODOS
{CMD_DIFF} --name-only {PATHSPEC} | wc -l  # → ARQUIVOS_COUNT
```

**Detectar stack** a partir de `LISTA_ARQUIVOS_TODOS`:
- `.php` ou `.blade.php` presentes → `STACK="Laravel 10 / PHP 8.1 / Blade"`
- `.ts`, `.html`, `.scss` em `src/` presentes → `STACK="Ionic/Angular / TypeScript"`
- Ambos → concatene com ` + `
- Nenhum → `STACK="Genérico (nenhum stack reconhecido nos arquivos alterados)"` e avise que os agentes usarão análise genérica

---

## Passo 2 — Gravar DIFF_FILE

Defina `DIFF_FILE` = caminho de arquivo temporário da sessão (ex: `<scratchpad>/triple-review.patch`).

```bash
{CMD_DIFF} {PATHSPEC} > {DIFF_FILE}
awk '/^[+-]/ && !/^(\+\+\+|---)/ {n++} END {print n+0}' {DIFF_FILE}   # → LINHAS_DIFF (awk imprime sempre um único número, inclusive 0)
```

**Se houve `git add -N` no Passo 1, desfaça a marcação agora** (o diff já está gravado no arquivo): `git reset -- {ARQUIVOS_ADD_N}` — passe cada caminho como argumento **separado e quotado** (`git reset -- "a.php" "my view.blade.php"`), nunca a lista interpolada solta: nome com espaço viraria dois pathspecs. O índice do usuário volta ao estado original e nenhum resíduo sobra para um `git commit -a` futuro.

Se `LINHAS_DIFF` > 1000 → `DIFF_GRANDE=true`: inclua nos prompts a instrução de priorização (ver template do Passo 5).

O diff completo nunca entra no contexto do orquestrador — os agentes leem o `DIFF_FILE` diretamente.

---

## Passo 3 — Estabelecer o baseline da rodada (memória entre rodadas)

Este passo dá memória ao ciclo. Sem ele, cada rodada é um sorteio novo de problemas pré-existentes.

**Localização do baseline — estável por branch, persistente entre rodadas:**
```bash
BRANCH=$(git branch --show-current)
[ -z "$BRANCH" ] && BRANCH="DETACHED"          # detached HEAD → nome fixo (senão BASELINE_FILE e o ref ficam inválidos)
# sanitiza p/ nome de arquivo e de ref. printf (não echo): `echo` emite \n final, que o `tr -c`
# converteria em '-', deixando um traço espúrio no nome do ref/arquivo.
# O sufixo de hash torna o nome INJETIVO: sem ele, `feature/foo` e `feature-foo` colidiriam
# no mesmo baseline e no mesmo ref — os achados de uma branch virariam o baseline da outra.
BRANCH_HASH=$(printf '%s' "$BRANCH" | git hash-object --stdin | cut -c1-8)
BRANCH_SAN="$(printf '%s' "$BRANCH" | tr -c 'A-Za-z0-9._-' '-')-$BRANCH_HASH"
BASELINE_DIR="docs/triple-review-tuning/.baselines"
BASELINE_FILE="$BASELINE_DIR/$BRANCH_SAN.json"
mkdir -p "$BASELINE_DIR"
```

> **Por que este caminho e não o scratchpad:** o scratchpad é isolado por sessão — não sobrevive entre invocações distintas de `/triple-review`. O baseline **precisa** persistir de uma rodada para a próxima na mesma branch, então fica versionado por branch em `docs/triple-review-tuning/.baselines/`. Esse diretório é **excluído da própria revisão** (Passo 1) e recomenda-se adicioná-lo ao `.gitignore` do projeto para não poluir commits. Cada branch tem seu próprio baseline — trocar de branch troca de baseline automaticamente.

**Capturar o estado revisado desta rodada (para o `git diff` de delta da próxima):**

> ⚠️ **Não use `git stash create`.** Ele só captura arquivos **rastreados** — os arquivos novos que o Passo 1 trouxe para a revisão via `git add -N` (e o Passo 2 desfez) ficariam **de fora, sem erro**. Na rodada seguinte, `git diff <sha> -- arquivo_novo` veria o arquivo inteiro como adicionado → `TOCADA` → um achado pré-existente seria rotulado `NOVO`. Isso é exatamente a não-convergência que este passo existe para eliminar, e dispararia no caso comum de "a feature criou um arquivo novo".

Capture o estado com um **índice temporário** que inclui os arquivos novos com conteúdo real:
```bash
TMP_INDEX=$(mktemp -u)                                          # índice descartável; não toca o índice do usuário
GIT_INDEX_FILE="$TMP_INDEX" git read-tree HEAD
# Lista via arquivo NUL-separado — nunca interpole {LISTA_ARQUIVOS_TODOS} solta na linha de comando:
# um nome com espaço ("my view.blade.php") viraria dois pathspecs e o git aborta com exit 128,
# deixando CUR_REVIEW_STATE_SHA inválido e degradando a próxima rodada para "primeira revisão".
FILE_LIST=$(mktemp)
{CMD_DIFF} --name-only -z {PATHSPEC} > "$FILE_LIST"
GIT_INDEX_FILE="$TMP_INDEX" git add -A --pathspec-from-file="$FILE_LIST" --pathspec-file-nul
TREE=$(GIT_INDEX_FILE="$TMP_INDEX" git write-tree)
CUR_REVIEW_STATE_SHA=$(git commit-tree "$TREE" -p HEAD -m "triple-review: estado revisado")
rm -f "$TMP_INDEX" "$FILE_LIST"
git update-ref "refs/triple-review/$BRANCH_SAN.pending" "$CUR_REVIEW_STATE_SHA"   # âncora PROVISÓRIA
```
> `GIT_INDEX_FILE` aponta o git para um índice descartável — o índice real do usuário nunca é tocado. `write-tree` + `commit-tree` produzem um commit-objeto com o conteúdo exato do que foi revisado, **incluindo arquivos não rastreados**. Guarde `CUR_REVIEW_STATE_SHA` para o Passo 7.

> ⚠️ **A âncora é provisória (`.pending`) de propósito.** Ela só vira a âncora oficial no Passo 7, **junto** com a gravação do baseline. Se esta rodada terminar em `REVISÃO NÃO REALIZADA` (Passo 7 não grava), o ref oficial da rodada anterior **permanece intacto** e o `review_state_sha` do baseline continua ancorado. Sobrescrever o ref oficial aqui órfãozaria o último sha bom, ele seria coletado pelo `gc`, e a rodada seguinte perderia toda a memória de delta (degradaria para "primeira revisão").

**Carregar o baseline anterior:**
- Se `BASELINE_FILE` **existe** e parseia como JSON → leia `findings`, `checklist` e `review_state_sha` (chame-o `REVIEW_STATE_SHA_PREV`). Confirme que ele ainda resolve: `git rev-parse -q --verify "$REVIEW_STATE_SHA_PREV^{commit}"`.
  - Resolve → `PRIMEIRA_REVISAO=false`. O delta será calculado contra `REVIEW_STATE_SHA_PREV`.
  - **Não** resolve (rebase/gc apagou o objeto) → não há como calcular delta com honestidade: `PRIMEIRA_REVISAO=true` e registre no relatório *"baseline anterior não pôde ser ancorado (estado revisado anterior indisponível) — esta rodada é tratada como primeira revisão."*
- Se `BASELINE_FILE` **existe mas NÃO parseia como JSON** (truncado por rodada interrompida no Passo 7, ou editado à mão) → `PRIMEIRA_REVISAO=true` **e registre no relatório**: *"baseline anterior existe mas está corrompido (JSON inválido) — foi ignorado; esta rodada é tratada como primeira revisão."* **Nunca degrade em silêncio:** sem essa nota, o usuário receberia toda a dívida pré-existente como `NOVO` sem saber por quê.
- Se `BASELINE_FILE` **não existe** → `PRIMEIRA_REVISAO=true`. Trate todos os achados como `PRIMEIRA REVISÃO` (sem conotação de regressão — não há baseline para comparar).

---

## Passo 4 — Inferir FEATURE_CONTEXT (sem perguntar)

Normalize `$ARGUMENTS` (trim; espaços múltiplos → um). Se tiver **4 ou mais palavras**, use como `FEATURE_CONTEXT`.
Se tiver **1–3 palavras, não descarte**: use como semente e complemente com a inferência abaixo — `FEATURE_CONTEXT = "<argumento> — <síntese inferida>"`.

**Se estiver vazio, infira automaticamente — não pergunte ao usuário, não bloqueie:**
1. Branch atual (`git branch --show-current`) — se seguir o padrão de card (ex: `OC-1234`), leia a documentação correspondente se existir (`docs/<branch>/`)
2. Mensagens de commit do trecho revisado: `git log <base>..HEAD --oneline` (base do `CMD_DIFF`), ou os últimos 5 commits se `CMD_DIFF="git diff HEAD"`
3. Sintetize em 1–2 frases: módulo/fluxo tocado + o que a mudança faz do ponto de vista do usuário
4. Se nada disponível: `FEATURE_CONTEXT = "Não especificado — análise baseada apenas no diff"`

Informe antes de disparar (puramente informativo — não aguarde resposta):
> "📝 Contexto: [FEATURE_CONTEXT] (fonte: [argumento/branch/commits/docs]). Baseline: [primeira revisão desta branch / comparando com a revisão de <data do baseline>]. Os 3 agentes já estão sendo disparados. Se o contexto estiver incorreto, **interrompa (Esc)** e rode `/triple-review <descrição correta>` — do contrário a espera é de 3–10 min. [Se houve `git add -N`: "Arquivos não rastreados incluídos na revisão: <lista> (marcação já desfeita no índice)."] Disparando os 3 agentes..."

---

## Passo 5 — Disparar os 3 agentes em paralelo

Dispare **simultaneamente** (três chamadas Agent em uma mesma mensagem) com `run_in_background: true`.

> **Paralelismo:** "simultaneamente" = três Agent calls na mesma mensagem. A gravação do `DIFF_FILE` (Passo 2) é feita antes — não conflita com o paralelismo.

Após disparar, **registre o instante do disparo** (`T0`) e **agende um despertar de fallback para ~10 min** — o mesmo limiar de timeout do Passo 6, para não prometer um prazo e acordar em outro:

```bash
T0=$(date +%s)   # guarde; o Passo 6 compara com este valor
```

- **Ambiente com wakeup/timer:** agende o despertar em ~10 min a partir de `T0`. Esse é o gatilho concreto do timeout.
- **Ambiente sem wakeup/timer:** você **não pode** simplesmente "consolidar conforme as notificações chegarem" — se um agente travar sem emitir notificação, a 3ª notificação nunca chega e a revisão esperaria para sempre, sem sinal de erro. Nesse caso, use um gatilho de tempo real bloqueante e limitado (ex: aguardar até `date +%s` passar de `T0 + 600`) e então consolide com quem respondeu. **Nunca deixe o orquestrador sem um gatilho temporal.**

Emita imediatamente:
> "3 agentes em execução simultânea — aguardando resultados (estimativa: 3–10 min dependendo do tamanho do diff). Você será notificado quando cada um finalizar."

Ao receber cada notificação, emita uma confirmação específica, por exemplo:
> "Agente QA concluído (1/3) — aguardando Sênior e Gerente UX..."

**Se qualquer agente retornar erro de invocação** (agent type not found, unknown agent type, subagent not available, ou qualquer outra falha que impeça o agente de iniciar) — os três são agentes locais do projeto (`qa-reviewer.md` / `senior-reviewer.md` / `gerente-ux.md`), então podem faltar se a skill foi copiada para outro projeto sem os arquivos de agente:
1. **Imediatamente** (sem esperar o timeout do Passo 6), leia o arquivo de perfil correspondente (`.claude/agents/qa-reviewer.md` para o Agente 1, `.claude/agents/senior-reviewer.md` para o Agente 2, `.claude/agents/gerente-ux.md` para o Agente 3)
2. Redispare o agente afetado com o **subagent_type genérico disponível no ambiente** (neste projeto: `claude`; em outros ambientes pode ser `general-purpose` — use o que constar na lista de agentes)
3. Prefixe o prompt com o conteúdo do arquivo de perfil, pulando as linhas de frontmatter (o bloco `---` no início do arquivo), antes das instruções específicas
4. O agente de fallback recebe seu próprio timeout de 10 min a partir do re-disparo; o timeout original do Passo 6 aplica-se apenas aos agentes que não precisaram de fallback
5. **Uma única tentativa de fallback por agente** — se o redisparo também falhar, marque o agente como ausente e siga para o 6d; não tente uma terceira vez

Se um agente simplesmente não responder dentro do timeout (10 min), aplique o Passo 6d sem fallback.

---

Use os templates abaixo **copiando o texto e substituindo os placeholders `{STACK}`, `{DIFF_FILE}`, `{ARQUIVOS_COUNT}`, `{LINHAS_DIFF}`, `{LISTA_ARQUIVOS_TODOS}` e `{FEATURE_CONTEXT}` pelos valores coletados nos Passos 1–4**. **Resolva também os blocos condicionais entre colchetes `[Se X: "..."]`**: se a condição vale, inclua apenas o texto interno (sem colchetes); senão, remova a linha inteira. Nenhum placeholder `{...}` nem colchete condicional `[Se ...]` pode chegar literalmente ao agente. Fora isso, não altere o texto dos templates em runtime.

> **Memória do revisor (`VEREDITOS_ANTERIORES`) — obrigatório quando `PRIMEIRA_REVISAO=false`:**
> O baseline dá memória ao **orquestrador**, mas sem este bloco os **revisores continuam amnésicos**: cada agente re-deriva os vereditos do zero, e nada ancora a resposta desta rodada na da anterior. O resultado é que o checklist fechado torna determinística a **cobertura** (quais perguntas são feitas), mas **não o veredito** (que resposta cada pergunta recebe) — itens que exigem raciocínio multi-hop viram `PASS` numa rodada e `FAIL` na outra, em código que ninguém tocou.
>
> Antes de disparar, monte para cada dimensão o conjunto `VEREDITOS_ANTERIORES` = os itens do `baseline.checklist` daquela dimensão **cujo código não foi tocado** (teste de toque do Passo 6d, aplicado ao arquivo que o item cobre; na dúvida sobre o alvo do item, inclua-o). O veredito **e a evidência** vêm do próprio valor persistido no baseline (formato `"PASS — evidência"` — ver Passo 7); baseline antigo que só guardou o veredito → injete a linha sem a parte da evidência, nunca invente uma. Injete esse bloco no prompt do agente e exija a regra:
>
> ```
> ## Vereditos anteriores (código NÃO alterado desde a última revisão)
> <item-ID>: <PASS|FAIL|N/A> — <evidência de 1 linha da rodada anterior>
> ...
>
> Estes foram os SEUS vereditos na rodada anterior, sobre código que NÃO mudou desde então.
> Reconfirme cada um. Você PODE invertê-los — a rodada anterior pode ter errado — mas para
> inverter você precisa citar EVIDÊNCIA NOVA E CONCRETA (o trecho exato, o comando que você
> executou, o cenário de falha reproduzível). Inversão sem evidência nova é proibida.
> Toda inversão será reportada ao usuário como "cobertura instável" naquele item.
> ```
>
> Isso ancora sem petrificar: erro da rodada anterior ainda pode ser corrigido, mas o agente paga o preço de justificar, e o usuário vê que o revisor oscilou. Quando `PRIMEIRA_REVISAO=true`, omita o bloco inteiro (não há vereditos anteriores).

> **Contrato de saída exigido dos três agentes (comum aos três templates):** além dos achados e do bloco `## Resumo <Dimensão>`, cada agente **precisa** devolver o bloco `## Checklist <Dimensão>` com **todos** os itens do seu perfil marcados `PASS` / `FAIL` / `N/A` (com o ID estável do item e 1 linha de evidência) — **não só os FAILs**. Todo achado tem que ser o FAIL de um item do checklist. O checklist fechado é o que torna a cobertura uma função do código (não do sorteio) e é o que o baseline compara entre rodadas. Resposta sem esse bloco é tratada como **parcial** no Passo 6a.

---

### Agente 1 — QA (QA Reviewer)

> Nota: se `qa-reviewer` não existir como subagent_type no seu ambiente (skill copiada sem o arquivo do agente), o Passo 5 aplica o fallback automático.
- `subagent_type`: `qa-reviewer`
- `run_in_background`: `true`
- `description`: `"Triple Review — QA"`

**Prompt:**

```
Você é o QA Sênior revisando este diff antes de um build de produção.
Siga as etapas, o checklist fechado, as categorias e os critérios de severidade definidos no seu perfil de agente (qa-reviewer.md) — não pule etapas nem itens.

## Contexto
Stack: {STACK}
Feature: {FEATURE_CONTEXT}
[Se PRIMEIRA_REVISAO=false: "{VEREDITOS_ANTERIORES}"]

## Diff a revisar
O diff completo está gravado em: {DIFF_FILE} — leia esse arquivo como primeiro passo.
Arquivos alterados ({ARQUIVOS_COUNT} arquivos, {LINHAS_DIFF} linhas):
{LISTA_ARQUIVOS_TODOS}
[Se DIFF_GRANDE: "Diff grande — priorize arquivos de lógica de negócio (controllers/services/models), depois migrations/rotas, depois views; liste o que não analisou em 'Etapas não concluídas' e marque Cobertura: PARCIAL."]

Não avalie qualidade de código (papel do Sênior) nem UX (papel do Gerente) — apenas corretude funcional, casos de fronteira, bloqueios de negócio ausentes e falhas silenciosas, conforme seu perfil.

Formato de saída obrigatório (conforme seu perfil): os achados com [SEVERIDADE], o campo "Cenários cobertos", o bloco ## Checklist QA com TODOS os itens marcados PASS/FAIL/N/A (não só os FAILs) e o bloco ## Resumo QA. Todo achado precisa ser o FAIL de um item do checklist.
```

---

### Agente 2 — Sênior (Senior Reviewer)

> Nota: se `senior-reviewer` não existir como subagent_type no seu ambiente, o Passo 5 aplica o fallback automático.
- `subagent_type`: `senior-reviewer`
- `run_in_background`: `true`
- `description`: `"Triple Review — Sênior"`

**Prompt:**

```
Você é o Desenvolvedor Sênior revisando este diff antes de um build de produção.
Siga as etapas, o checklist fechado, as categorias e os critérios de severidade definidos no seu perfil de agente (senior-reviewer.md) — não pule etapas nem itens, e verifique ativamente no codebase em vez de apenas ler o diff.

## Contexto
Stack: {STACK}
Feature: {FEATURE_CONTEXT}
[Se PRIMEIRA_REVISAO=false: "{VEREDITOS_ANTERIORES}"]

## Diff a revisar
O diff completo está gravado em: {DIFF_FILE} — leia esse arquivo como primeiro passo.
Arquivos alterados ({ARQUIVOS_COUNT} arquivos, {LINHAS_DIFF} linhas):
{LISTA_ARQUIVOS_TODOS}
[Se DIFF_GRANDE: "Diff grande — priorize arquivos de lógica de negócio (controllers/services/models), depois migrations/rotas, depois views; liste o que não analisou em 'Etapas não concluídas' e marque Cobertura: PARCIAL."]

Se o stack incluir Ionic/Angular: verificar também tipagem TypeScript estrita, ausência de `any`, componentes standalone.

Não avalie UX (papel do Gerente) nem falhas silenciosas/bloqueios de negócio (papel do QA) — apenas design, corretude, convenções e manutenibilidade, conforme seu perfil.

Formato de saída obrigatório (conforme seu perfil): os achados com [SEVERIDADE], o campo "Verificações feitas", o bloco ## Checklist Sênior com TODOS os itens marcados PASS/FAIL/N/A (não só os FAILs) e o bloco ## Resumo Sênior. Todo achado precisa ser o FAIL de um item do checklist.
```

---

### Agente 3 — Gerente UX

> Nota: se `gerente-ux` não existir como subagent_type no seu ambiente, o Passo 5 aplica o fallback automático.
- `subagent_type`: `gerente-ux`
- `run_in_background`: `true`
- `description`: `"Triple Review — Gerente UX"`

**Prompt:**

```
Você é o Gerente UX. Avalie a mudança do ponto de vista do usuário final.
Siga as etapas, a grade heurística fechada e os critérios de severidade definidos no seu perfil de agente (gerente-ux.md) — não pule etapas nem itens.

## Contexto
Stack: {STACK}
Feature: {FEATURE_CONTEXT}
[Se PRIMEIRA_REVISAO=false: "{VEREDITOS_ANTERIORES}"]

## Diff a revisar
O diff completo está gravado em: {DIFF_FILE} — leia esse arquivo como primeiro passo.
Arquivos alterados ({ARQUIVOS_COUNT} arquivos, {LINHAS_DIFF} linhas):
{LISTA_ARQUIVOS_TODOS}
[Se DIFF_GRANDE: "Diff grande — priorize views/componentes/telas e mensagens visíveis ao usuário, depois controllers/rotas que alteram dados exibidos, depois o restante; liste o que não analisou em 'Etapas não concluídas' e marque Cobertura: PARCIAL."]

Formato de saída obrigatório (conforme seu perfil): pontos positivos, achados com [SEVERIDADE] [TELA/COMPONENTE], perguntas, o campo "Cenários cobertos", o bloco ## Checklist UX com TODOS os itens da grade marcados PASS/FAIL/N/A (não só os FAILs) e o bloco ## Resumo UX com veredito OK/ATENÇÃO/CRÍTICO. Todo achado precisa ser o FAIL de um item da grade.
```

---

## Passo 6 — Consolidar resultados (com baseline/delta)

Aguarde os três agentes. **Timeout:** o gatilho concreto é o despertar (ou a espera limitada) agendado no Passo 5, ambos em ~10 min a partir de `T0` — o mesmo prazo comunicado ao usuário. Ao acordar (por notificação de conclusão ou pelo gatilho temporal), se algum agente ainda não respondeu e já se passaram 10+ minutos de `T0`, consolide sem ele.

> **Onde consolidar — Opção A (inline, padrão) vs Opção B (subagente `consolidador-review`):**
> A consolidação abaixo (6a–6h) é **determinística e pesada de contexto** (um `git diff` por achado, fingerprint, gate de confiança, leitura/escrita do baseline).
> - **Padrão (Opção A):** o orquestrador executa 6a–6h inline. Menos latência, sem agente extra. Use quando `DIFF_GRANDE=false` (≤1000 linhas).
> - **Opção B (escalar):** se `DIFF_GRANDE=true` (>1000 linhas) **ou** se o contexto do orquestrador já estiver pesado a ponto de ameaçar o determinismo, delegue 6a–6h ao subagente **não-revisor** `consolidador-review` (`run_in_background: false`). Passe a ele: os 3 relatórios brutos, `BASELINE_FILE`, `REVIEW_STATE_SHA_PREV`, `CUR_REVIEW_STATE_SHA`, `DIFF_FILE`, os metadados (STACK/FEATURE_CONTEXT/contagens/`ARQUIVOS_ADD_N`) e quais agentes ficaram ausentes/parciais. Ele devolve o relatório consolidado + o JSON do novo baseline (que você grava no Passo 7). Contexto frio e focado → determinismo melhor; custo: +1 agente, +latência. **`consolidador-review` NÃO é um 4º revisor** — ele não gera achados novos, só classifica/filtra os dos três.
>
> As regras 6a–6h são as mesmas em A e B — em B o subagente as executa; em A, o orquestrador.

### 6a. Validação de resposta (antes de contar como "respondeu")

- A resposta deve conter o bloco `## Resumo QA` / `## Resumo Sênior` / `## Resumo UX` correspondente. Resposta vazia, truncada ou sem esse bloco → trate o agente como **ausente** (6g).
- A resposta deve conter também o bloco `## Checklist <Dimensão>` **completo** (todos os itens PASS/FAIL/N/A). Tem resumo mas **não** tem o checklist completo → o agente é **parcial**: consuma os achados que ele listou, marque a dimensão como `Cobertura: PARCIAL` e rebaixe a confiança da revisão. (Parcial ≠ ausente: parcial ainda contribui achados.)
- Se a resposta traz defeitos em prosa sem os labels `[BLOCKER]/[WARNING]/[INFO]`, não os ignore: classifique cada um pela definição de severidade do perfil do agente e marque `(classificado na consolidação)`.

**Se o timeout for atingido antes de todos responderem**, notifique o usuário antes de consolidar:
> "⏱ Timeout atingido — consolidando com [N]/3 agentes disponíveis. O relatório abaixo pode ter cobertura incompleta."

### 6b. Gate de confiança (C4)

**Descarte** qualquer achado que **não** traga um cenário de falha concreto (input/estado → resultado errado observável). "Seria bom", "por via das dúvidas", robustez preventiva sem consequência → não entra. **Conte** quantos foram descartados por esse gate e reporte a contagem (não os liste) — é o que impede achados-fantasma que aparecem/somem entre rodadas.

### 6c. Deduplicar achados

**Para achados de QA e Sênior:** são duplicatas se dois ou mais agentes citam o **mesmo arquivo + mesmo defeito subjacente** (mesma causa raiz ou mesmo sintoma). A proximidade de linha (±5 linhas) é um **sinal auxiliar** — quando presente, aumenta a confiança de que é duplicata, mas não é critério obrigatório.

**Para achados do Gerente UX** (formato `[TELA/COMPONENTE]`): é duplicata de um achado QA/Sênior quando **mesmo componente/tela + mesma ação do usuário + mesma consequência funcional**. Não exija correspondência de arquivo:linha.

Ao encontrar duplicata: mantenha uma ocorrência com tag `⚠️ detectado por múltiplos agentes (QA + Sênior / etc.)` e eleve a severidade para o nível mais alto entre as versões.

**Regras anti-alucinação na consolidação:**
- **Na dúvida, não mescle** — dois achados parecidos mas não claramente o mesmo defeito ficam separados.
- **Copie arquivo:linha exatamente como o agente reportou** — nunca reescreva, "corrija" ou invente localização.
- **Não fabrique consenso** — se um agente flagou algo e outro explicitamente considerou o mesmo ponto OK, mantenha o achado com a nota `(divergência: [agente X] não viu problema)` em vez de descartar ou rebaixar.

### 6d. Tags de origem — baseline/delta (C5)

Para **cada** achado deduplicado, componha um **fingerprint estável** (identidade que sobrevive ao deslocamento de linhas — não use a linha absoluta como identidade):

```
fingerprint = <arquivo> | <escopo> | <dimensão> | <item-ID-do-checklist> | <assinatura-normalizada>
```
onde `<escopo>` = símbolo que contém o achado (método/classe) quando determinável, senão o próprio `<item-ID>`; `<item-ID>` = o ID do item de checklist que gerou o FAIL (âncora mais estável); `<assinatura-normalizada>` = descrição curta em minúsculas, espaços colapsados, números/linhas removidos. Dois achados têm a mesma identidade quando têm o mesmo fingerprint.

**Se `PRIMEIRA_REVISAO=true`** (sem baseline ou estado anterior irresolvível): marque **todos** os achados como `PRIMEIRA REVISÃO` (sem conotação de regressão) e vá para 6e.

**Havendo baseline**, para cada achado, na ordem:

1. **Teste de toque (objetivo, via git) — compare os dois commit-objetos, nunca a working tree:**
   `git diff --unified=3 <REVIEW_STATE_SHA_PREV> <CUR_REVIEW_STATE_SHA> -- <arquivo>` — a linha citada está **TOCADA** se ela (±3 linhas) cai dentro de um trecho com linhas adicionadas/modificadas (`+`); sem hunks para o arquivo → **NÃO TOCADA**. O critério é git, nunca julgamento.

   > ⚠️ **Não use `git diff <SHA_PREV> -- <arquivo>` (contra a working tree).** Para um arquivo **não rastreado** (o caso de arquivo novo, trazido à revisão via `git add -N` no Passo 1 e destacado no Passo 2), esse diff não enxerga o conteúdo da working tree: ele emite só linhas `-` (uma "deleção fantasma"), nunca `+`. Como o teste só marca `TOCADA` quando há `+`, **todo arquivo novo sairia sempre como `NÃO TOCADA` → `PRÉ-EXISTENTE`**, e uma regressão que você acabou de introduzir nele seria rotulada dívida conhecida e não bloquearia. Ambos os shas são commit-objetos que já contêm o conteúdo real (Passo 3), então `diff <PREV> <CUR>` é correto para rastreados e não rastreados.
2. **Pertence ao baseline?** O fingerprint casa com algum do baseline?
3. Árvore de decisão (precedência de cima para baixo):
   - **NÃO TOCADA** → `PRÉ-EXISTENTE`. Adicione a frase literal: *"já existia; não foi introduzido pelas suas correções."*
     - se **não** estava no baseline → acrescente *" — lacuna de cobertura da rodada anterior."*
     - se **estava** no baseline → acrescente *" (persistente desde a última revisão)."*
   - **TOCADA e NÃO no baseline** → `NOVO`. Se o item de checklist correspondente estava `PASS` no baseline (piora de algo antes limpo) → escale para `REGRESSÃO` e **fixe no topo** do relatório (é o que o usuário mais precisa ver).
   - **TOCADA e no baseline** → `PERSISTENTE` (mexeu no código, mas o mesmo defeito continua).
4. **RESOLVIDO:** qualquer fingerprint do baseline que **não** aparece nesta rodada = `RESOLVIDO`. **Não liste** — só conte: "X achados resolvidos desde a última revisão."

**Explicações obrigatórias por tag** (alimentam a Retrospectiva do Passo 8 — sem elas o achado não entra no relatório):

- **`NOVO`/`REGRESSÃO`** → anexe `→ Causa:` citando o trecho do hunk (`git diff <PREV> <CUR> -- <arquivo>`) e explicando em 1–2 frases **como** a mudança introduziu o defeito. **A explicação é o teste da tag:** se o hunk que toca a linha não tem relação causal com o defeito (ex.: só reformatação, renomeação, mudança vizinha sem alterar o comportamento citado), a tag está errada — reclassifique como `PRÉ-EXISTENTE — lacuna de cobertura` e siga a regra de lacuna abaixo.
- **`PRÉ-EXISTENTE` com "lacuna de cobertura"** → anexe `→ Por que passou antes:` com **exatamente uma** categoria desta taxonomia fechada (não escreva prosa livre fora dela):
  - **(a) item vago** — o item de checklist que deveria pegar existia, mas é pergunta de julgamento, não procedimento (cite o item-ID)
  - **(b) fora do escopo revisado** — arquivo omitido/DIFF_GRANDE/cobertura PARCIAL declarada na rodada anterior
  - **(c) multi-hop** — detectá-lo exige cruzar informação de 2+ arquivos que nenhum item manda cruzar (cite o item-ID mais próximo)
  - **(d) classe descoberta** — nenhum item do checklist cobre essa classe de defeito
  - **(e) agente ausente/parcial** — a dimensão não rodou (ou rodou parcial) na rodada anterior

### 6e. Divergência de cobertura (denuncia o bug de amostragem)

Compare `baseline.checklist` com os checklists desta rodada. Para itens cujo código **não** foi tocado (teste de toque de 6d) mas cujo veredito **mudou** (`PASS`↔`FAIL`), emita `⚠️ cobertura instável no item <ID>` no relatório. Isso expõe exatamente a amostragem que causa a não-convergência.

> **Normalização na comparação:** o valor do checklist pode estar no formato antigo (`"PASS"`) ou no atual (`"PASS — evidência"`) — compare **apenas o veredito** (token antes de ` — `). Chaves de dimensão casam case-insensitive (`qa` ≡ `QA`, `senior` ≡ `Senior`); a grafia canônica ao **gravar** é minúscula (Passo 7). Evidência diferente com o mesmo veredito **não** é cobertura instável.

**Exceção — item alterado pela Retrospectiva:** antes de acusar instabilidade, confira `docs/triple-review-tuning/CHANGELOG.md`. Se o item foi reescrito/criado pela Retrospectiva (Passo 8) **depois** do `reviewed_at` do baseline, a mudança de veredito é esperada (o item mudou, não a amostragem) — reporte como `🔧 item <ID> atualizado na retrospectiva de <data> — reavaliado sob o novo procedimento`, não como cobertura instável.

### 6f. Regra de veredito (determinística — aplique nesta ordem)

Para o veredito, achados `PRIMEIRA REVISÃO` contam como `NOVO` (sem baseline não há como provar que são antigos).

1. **REVISÃO NÃO REALIZADA** → todos os três agentes não responderam — não buildar; repetir `/triple-review`.
2. **BLOQUEADO** → existe **qualquer** achado `[BLOCKER]` nos resultados consolidados (independe da tag de origem: um BLOCKER pré-existente ainda é perda/corrupção de dado — não se builda em cima dele). BLOCKERs `NOVO`/`REGRESSÃO` vão fixados no topo.
3. **APROVADO COM RESSALVAS** → zero `[BLOCKER]` **e** (≥1 `[WARNING]` com tag `NOVO`/`REGRESSÃO`, **ou** ≥1 agente ausente/parcial).
4. **APROVADO** → zero `[BLOCKER]` **e** **zero achado `NOVO` ou `REGRESSÃO`**. Achados `PRÉ-EXISTENTE`/`PERSISTENTE` de severidade `WARNING`/`INFO` **não impedem** APROVADO — aparecem listados como **dívida conhecida**.

> **⚠️ Mudança de semântica (destaque no relatório):** WARNINGs/INFOs **pré-existentes não travam mais o build**. O veredito passou a depender do que **mudou** desde a última revisão, não do acúmulo histórico de dívida. Assim o estado "limpo" (APROVADO) é alcançável mesmo com dívida conhecida na base — desde que você não tenha introduzido nada `NOVO`/`REGRESSÃO`. Deixe isso explícito no relatório para o usuário entender por que WARNINGs antigos aparecem listados mas não bloqueiam.

### 6g. Tratamento de agente que falhou

Quando qualquer agente não responder, adicione um **banner no topo do relatório** (antes do Resumo Executivo):

> "⚠️ COBERTURA INCOMPLETA: o agente [QA / Sênior / Gerente UX] não retornou. Achados de [funcional/bloqueios de negócio / design/corretude/convenções / UX] estão ausentes. O veredito não poderá ser APROVADO (e será BLOQUEADO se houver BLOCKER).
> → Para cobrir a dimensão ausente, rode `/triple-review` novamente (ou aguarde e repita se foi timeout)."

- Marque a seção ausente: `[AGENTE NÃO RESPONDEU — análise desta dimensão indisponível]`
- Um agente ausente, com ao menos um respondendo, impede que o veredito seja **APROVADO** — o melhor resultado possível nesse caso é **APROVADO COM RESSALVAS** (regra 6f-3). Se todos os três estiverem ausentes, aplica-se REVISÃO NÃO REALIZADA (6f-1), que tem precedência.
- Se o motivo foi falha imediata de invocação: aplique o fallback do Passo 5 antes de consolidar (não marque como ausente ainda).

### 6h. Relatório consolidado

> **Se o veredito for REVISÃO NÃO REALIZADA** (6f-1): não emita o relatório abaixo — emita apenas o banner do 6g e a instrução do Passo 9 para REVISÃO NÃO REALIZADA (os Passos 7 e 8 não rodam).

> **Antes de emitir o relatório:** substitua TODOS os `{placeholders}` abaixo com os valores reais coletados nos Passos 1–4. Nenhum `{placeholder}` deve aparecer literalmente no relatório final.
> **Nas três seções por agente, inclua apenas os achados (com a tag de origem) e o bloco de resumo do agente** — sem preâmbulos nem narrativa; a análise completa fica no contexto do subagente.

```
{banner de cobertura incompleta — se houver agente ausente}

## Triple Review — [nome da feature de FEATURE_CONTEXT]

**Stack:** {STACK}
**Diff:** {LINHAS_DIFF} linhas de código alterado em {ARQUIVOS_COUNT} arquivos
**Baseline:** {primeira revisão desta branch — sem comparação de delta / comparando com a revisão de <data do baseline>}
**Cobertura:** {COMPLETA se os três agentes reportaram COMPLETA; senão PARCIAL — qual agente reportou parcial/ausente e o que ficou de fora}
{se houve git add -N no Passo 1: "**Arquivos não rastreados incluídos na revisão:** [lista] (marcação já desfeita no índice)"}

---

### 🔁 Delta desta rodada
- **Regressões (introduzidas por correções recentes):** [lista de REGRESSÃO, cada uma com `→ Causa:` — ou "nenhuma"]
- **Novos (código tocado desde a última revisão):** [lista de NOVO, cada um com `→ Causa:` — ou "nenhum"]
- **Resolvidos desde a última revisão:** [N] achados
- **Dívida conhecida (não bloqueia):** [contagem de PRÉ-EXISTENTE + PERSISTENTE por severidade; lacunas de cobertura com `→ Por que passou antes: (categoria)`]
- **Descartados pelo gate de confiança (sem cenário concreto):** [N]
- {⚠️ cobertura instável no item [ID] — se houver divergência de checklist em código não tocado}
- {🔧 item [ID] atualizado na retrospectiva de [data] — se a divergência for explicada pelo CHANGELOG}

---

### QA — Funcional, Fronteira, Bloqueios e Falhas Silenciosas
{achados com tag de origem + Resumo QA — ou "[AGENTE NÃO RESPONDEU]" se falhou}

---

### Sênior — Design, Corretude, Convenções e Manutenibilidade
{achados com tag de origem + Resumo Sênior — ou "[AGENTE NÃO RESPONDEU]" se falhou}

---

### Gerente — UX e Fluxo
{achados com tag de origem + Resumo UX — ou "[AGENTE NÃO RESPONDEU]" se falhou}

---

## Resumo Executivo

**🔴 Bloqueadores — não buildar até corrigir:**
- [ARQUIVO:LINHA ou TELA] Descrição — _origem: NOVO/REGRESSÃO/PRÉ-EXISTENTE/PERSISTENTE_ _(fonte: QA / Sênior / Gerente / múltiplos)_

**🟡 Recomendados — corrigir antes de produção:**
- [ARQUIVO:LINHA ou TELA] Descrição — _origem: ..._ _(fonte: ...)_

**⚪ Cosméticos / Nice-to-have:**
- Descrição — _origem: ..._ _(fonte: ...)_

_(Achados PRÉ-EXISTENTE/PERSISTENTE de severidade WARNING/INFO aparecem como dívida conhecida e não impedem APROVADO — só REGRESSÃO/NOVO e qualquer BLOCKER pesam no veredito.)_

---

**Veredito final:** APROVADO / APROVADO COM RESSALVAS / BLOQUEADO / REVISÃO NÃO REALIZADA

**Confiança da revisão:** ALTA / MÉDIA / BAIXA
(Alta = três agentes responderam com cobertura COMPLETA; Média = 1 agente ausente/parcial ou 1+ agente com cobertura PARCIAL; Baixa = 2+ agentes ausentes ou cobertura PARCIAL generalizada)
{justificar se MÉDIA ou BAIXA}
```

---

## Passo 7 — Gravar o novo baseline

Após emitir o relatório (e **só** se o veredito não for REVISÃO NÃO REALIZADA), grave o baseline desta rodada para a próxima comparar. Em **Opção B**, use o JSON que o `consolidador-review` devolveu; em **Opção A**, monte-o inline.

Grave em `BASELINE_FILE` (Passo 3) um JSON com:
- `branch`, `reviewed_at` (ISO-8601 de agora), `review_state_sha` = `CUR_REVIEW_STATE_SHA` (Passo 3), `diff_base` = `BASE` (Passo 1);
- `findings`: **todos** os achados desta rodada (após dedup e gate de confiança), cada um com `fingerprint`, `arquivo`, `linha`, `escopo`, `dimensao`, `item`, `severidade`, `tag`, `assinatura`;
- `checklist`: para cada dimensão que respondeu, o mapa completo `{ "<item-ID>": "PASS|FAIL|N/A — <evidência de 1 linha>" }`. **Chaves de dimensão canônicas em minúsculas: `qa` / `senior` / `ux`.** A evidência persistida é o que alimenta o bloco `VEREDITOS_ANTERIORES` da próxima rodada (Passo 5) — sem ela os revisores recebem só o veredito, sem âncora. **Se o agente não reportou evidência para um item** (resposta parcial, ver 6a) → grave só o veredito, sem o separador (`"PASS"`), **nunca fabrique uma evidência para preencher o formato**. Ao **ler** um baseline antigo, aceite os dois formatos: valor só-veredito (`"PASS"`) e chaves de dimensão em outra caixa (`QA`/`Senior`) — case-insensitive.

> **Grafia canônica do campo `tag` no JSON persistido** (a prosa do relatório usa acentos; o JSON, não — para não depender de encoding ao comparar entre rodadas): `NOVO` · `REGRESSAO` · `PRE_EXISTENTE` · `PERSISTENTE` · `RESOLVIDO` · `PRIMEIRA_REVISAO`. A identidade comparada entre rodadas é sempre o `fingerprint`, nunca a `tag`.

```bash
# 1) escreva o JSON montado em $BASELINE_FILE (o diretório já foi criado no Passo 3)
# 2) só DEPOIS que o baseline estiver gravado com sucesso, promova a âncora provisória a oficial:
git update-ref "refs/triple-review/$BRANCH_SAN" "$CUR_REVIEW_STATE_SHA"
git update-ref -d "refs/triple-review/$BRANCH_SAN.pending"
```
> **Ordem obrigatória:** baseline gravado → âncora promovida. Assim o ref oficial e o `review_state_sha` do baseline **sempre** apontam para o mesmo objeto. Se a rodada abortar antes disto, o par (ref oficial + baseline) da rodada anterior segue íntegro e o delta da próxima rodada continua funcionando.

Feche o relatório com uma linha explícita:
> "🗃️ Baseline atualizado (`docs/triple-review-tuning/.baselines/<branch>.json`) — a próxima revisão desta branch comparará com este estado."

---

## Passo 8 — Retrospectiva automática (a skill se corrige)

Roda **automaticamente após o Passo 7** (baseline já gravado — nunca antes: a rodada atual termina com os prompts que a iniciaram, senão a comparação entre rodadas fica contaminada). Objetivo: cada lacuna de cobertura vira uma correção no checklist para **não escapar de novo**.

**Gatilho:** existir ao menos um achado com `→ Por que passou antes:` de categoria **(a)**, **(c)** ou **(d)**, ou uma reclassificação `NOVO→lacuna` feita no 6d. Sem gatilho, emita apenas: "🔧 Retrospectiva: nada a melhorar nesta rodada." — e siga ao Passo 9. Categorias **(b)** e **(e)** não geram mudança (são limites declarados de cobertura, já reportados).

**Ação por categoria (edite `docs/triple-review-tuning/checklist-overrides.md` — NUNCA os arquivos de agente, que podem vir de plugin somente-leitura e são compartilhados entre projetos):**

O arquivo tem uma seção por dimensão (`## QA`, `## Sênior`, `## UX`); os agentes o aplicam por cima do checklist base: **ID igual substitui o texto do item; IDs `QA-EXTRA-*`/`SR-EXTRA-*`/`UX-EXTRA-*` são adicionados**. Crie o arquivo com as três seções se não existir.

- **(a) item vago** e **(c) multi-hop** → **reescreva o item citado** como override em forma de **procedimento**: o que inspecionar, onde, qual caminho seguir entre arquivos, e o que constitui FAIL (ex: "para cada X anunciado, localize com Grep o mecanismo que o implementa; ausente → FAIL"). **Mantenha o ID, não altere a severidade.** Use o achado escapado como caso de teste mental: o item reescrito TEM que pegá-lo.
- **(d) classe descoberta** → **crie um item `*-EXTRA-NN`** na seção da dimensão, em forma de procedimento, com o próximo número livre. **Orçamento: máx. 1 item novo por rodada** — havendo mais de uma lacuna (d), implemente a de maior severidade e registre as demais no CHANGELOG como `PENDENTE` (a próxima rodada as herda como gatilho).
- Melhoria que for **genérica** (serviria a qualquer projeto, não só a este) → além do override local, registre no CHANGELOG com a marca `UPSTREAM?` — candidata a ser promovida ao plugin/base via PR, decisão do usuário.

**Limites imutáveis desta retrospectiva:**
- Só escreve em `checklist-overrides.md` e no CHANGELOG — nunca nos perfis de agente, nem nas regras de processo, veredito, tags ou consolidação do comando (mudança estrutural é decisão do usuário, não da retrospectiva).
- Toda mudança vale **a partir da próxima rodada** — nunca redispare agentes na rodada atual por causa dela.
- **Registre cada mudança** (inclusive as `PENDENTE`) em `docs/triple-review-tuning/CHANGELOG.md`, formato: `| data ISO | branch | item-ID | ação (reescrito/criado/pendente) | categoria | fingerprint do achado gatilho |`. Crie o arquivo com o cabeçalho da tabela se não existir.

Feche com uma linha no chat (fora do relatório):
> "🔧 Retrospectiva: [item QA-XX reescrito (multi-hop) / item SR-YY criado (classe descoberta) / N pendentes] — ver CHANGELOG. Valem a partir da próxima rodada."

---

## Passo 9 — Aguardar decisão

Formule a pergunta de forma condicional ao veredito:

- **Se BLOQUEADO:** "Existem [N] BLOCKERs que impedem o build seguro ([X] NOVO/REGRESSÃO, [Y] pré-existentes). Deseja corrigir agora ou quer que eu liste o plano de correção?"
- **Se APROVADO COM RESSALVAS por WARNINGs NOVO/REGRESSÃO:** "O build pode prosseguir, mas há [N] WARNINGs introduzidos/regredidos desde a última revisão. Deseja resolver algum antes de buildar?"
- **Se APROVADO COM RESSALVAS por agente ausente (0 WARNINGs de delta):** "Nenhum problema novo encontrado, mas a dimensão [QA / Sênior / UX] não foi revisada. Deseja repetir a revisão para cobri-la ou prosseguir assim?"
- **Se APROVADO:** "Revisão limpa no delta — nenhum achado novo ou regressão desde a última revisão (pode haver dívida conhecida listada, que não bloqueia). Pode buildar."
- **Se REVISÃO NÃO REALIZADA:** "Nenhum agente respondeu — a revisão não foi realizada. Repita /triple-review antes de buildar."

Não faça build, instale o app nem execute qualquer deploy sem aprovação explícita do usuário.

---

## Regras imutáveis

- **Sempre disparar os 3 revisores em paralelo** — nunca sequencialmente. **Nunca adicionar um 4º ângulo de revisão** (só ampliaria a superfície de análise, que é a causa da não-convergência). O `consolidador-review` opcional **não** é um revisor.
- **Nunca pular o Gerente** — mudanças puramente técnicas frequentemente têm impacto de UX invisível (ex: uma migration que altera saldo disponível muda o que o almoxarife vê na tela de reserva).
- **O diff completo nunca entra no contexto do orquestrador** — sempre via `DIFF_FILE`; aqui só circulam stat, métricas, o baseline e os achados consolidados.
- **Memória entre rodadas é obrigatória** — sempre carregar o baseline no Passo 3 e gravar o novo no Passo 7. É o que faz o ciclo convergir.
- **Tag de origem é objetiva** — sempre por `git diff` contra o estado revisado anterior, nunca por julgamento.
- **Nunca buildar antes** do relatório estar completo e aprovado pelo usuário.
- **Consultas ao banco são read-only** — nunca INSERT/UPDATE/DELETE/DDL durante o review. Nunca rode comandos proibidos pelo projeto (ex: suítes de teste que resetam banco) — veja a lista em `docs/triple-review-tuning/customizacao.md`, seção "Regras críticas do domínio (QA)".

## Acesso ao banco (para os agentes)

Os agentes podem consultar o banco para enriquecer a análise. Eles têm acesso ao sistema de arquivos e podem ler credenciais do `.env` do projeto (`DB_HOST`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, etc.).

**Primário — usuário de banco com permissão apenas de leitura (`GRANT SELECT`):**
```bash
mysql -h $DB_HOST -u $DB_READONLY_USER -p$DB_READONLY_PASSWORD $DB_DATABASE -e "SELECT ..."
```

**Secundário — se não houver usuário dedicado (não bloqueia DDL):**
```bash
mysql -h $DB_HOST -u $DB_USERNAME -p$DB_PASSWORD \
  --init-command="SET SESSION TRANSACTION READ ONLY" \
  $DB_DATABASE -e "SELECT ..."
```

**Via Tinker — use apenas `DB::select()`; Tinker não impõe read-only:**
```bash
php artisan tinker --execute="DB::select('SELECT ...')"
```

> **Nota:** `SET SESSION TRANSACTION READ ONLY` não bloqueia DDL no MySQL. Se o ambiente usa Oracle (via `yajra/oci8`), adapte o comando de conexão conforme as credenciais Oracle do `.env`. A opção mais segura é sempre um usuário de banco sem permissão de escrita.

Nunca executar INSERT, UPDATE, DELETE ou DDL.

---

## Customização para outros projetos

> Se você está adaptando esta skill para um projeto diferente, edite apenas os itens abaixo:

> **Toda a customização de conteúdo vive no OVERLAY do projeto** (`docs/triple-review-tuning/customizacao.md`) — os arquivos de agente e este comando são o motor genérico (possivelmente instalados via plugin, somente-leitura) e não devem ser editados por projeto. Copie o template do overlay (`templates/customizacao.md` no plugin) e preencha as seções.

| O que customizar | Onde fica | O que substituir |
|---|---|---|
| Convenções de código (itens SR-CONV, com marca `(crítica)`) | overlay → seção "Convenções de código (Sênior)" | Suas convenções; mantenha em sincronia com o CLAUDE.md do projeto |
| Regras de negócio críticas e precisão numérica | overlay → seções "Regras críticas do domínio (QA)" e "Precisão numérica" | As regras do seu domínio cujo bypass seria grave + padrão decimal |
| Perfil dos usuários e padrões visuais | overlay → seções "Perfil dos usuários (UX)" e "Padrões visuais e de interação (UX)" | Quem são seus usuários finais, condições de uso e padrões de UI |
| Stack | overlay → seção "Stack" (complementa a detecção por extensão do Passo 1) | Declare o stack real do projeto |
| Agentes locais (QA/Sênior/UX) | `subagent_type` dos 3 agentes + fallback no Passo 5 | Tipo genérico do seu ambiente + embed do `.md` correspondente se não tiver os agentes locais copiados |
| Consolidador (opcional) | `consolidador-review.md` + Passo 6 (Opção B) | Mantenha genérico; só ative quando o diff for grande ou o determinismo inline estiver ameaçado |
| Local do baseline | Passo 3 — `BASELINE_DIR` | Um caminho por branch que persista entre rodadas; recomende `.gitignore` desse diretório |
| Comando de banco | overlay → seção "Acesso ao banco" | Adapte se não usar MySQL/Laravel; sempre read-only |
