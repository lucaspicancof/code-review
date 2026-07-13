# code-review

Meus plugins de revisão de código para o [Claude Code](https://claude.com/claude-code).

Criei esse repositório pra parar de passar a skill por zip/copy-paste — todo mundo instala do mesmo lugar, atualiza com um comando e ninguém fica com versão divergente.

## Plugins

### triple-review

Revisão pré-build com três revisores especializados rodando **em paralelo**, cada um olhando o diff por um ângulo diferente:

- **QA** — corretude funcional, casos de fronteira, bloqueios de negócio ausentes e falhas silenciosas (checklist `QA-*`, 25 itens)
- **Sênior** — design/arquitetura, corretude e regressão, convenções do projeto e manutenibilidade, com verificação ativa no codebase (checklist `SR-*`)
- **Gerente UX** — fluxo, clareza, perda de trabalho e consistência do ponto de vista do usuário final, ancorado nas 10 Heurísticas de Nielsen (grade `UX-*`)

No final, um orquestrador consolida tudo: deduplica, aplica gate de confiança e compara com o **baseline da branch**. Cada achado sai rotulado — `NOVO` / `REGRESSÃO` / `PRÉ-EXISTENTE` / `PERSISTENTE` — e o veredito depende do que **mudou** desde a última rodada. Ou seja: dívida antiga vira dívida conhecida, não trava o build de novo a cada revisão.

De quebra, tem uma **retrospectiva automática**: quando um problema escapa de uma rodada e aparece na seguinte, a skill descobre por que o checklist não pegou e melhora o item — o checklist do seu projeto evolui sozinho.

## Instalação

> ⚠️ **Os comandos abaixo rodam dentro do Claude Code no terminal** (o CLI — aquele que você abre digitando `claude`). Não é no bash direto, nem funciona em qualquer versão da extensão de IDE. Abra o Claude Code no terminal e digite:

```
/plugin marketplace add lucaspicancof/code-review
/plugin install triple-review@code-review
```

Pronto — `/triple-review` fica disponível em qualquer projeto.

## Configuração por projeto (recomendado)

O plugin é só o **motor genérico** — ele não sabe nada do seu projeto. As regras de negócio críticas, convenções de código, perfis de usuário e padrões de UI vivem num **overlay dentro do seu repositório**:

1. Copie [`plugins/triple-review/templates/customizacao.md`](plugins/triple-review/templates/customizacao.md) para `docs/triple-review-tuning/customizacao.md` no seu projeto
2. Preencha as seções — os títulos `##` são fixos, os agentes procuram por eles pelo nome, então não renomeie
3. Adicione `docs/triple-review-tuning/.baselines/` ao `.gitignore` (é onde fica a memória entre rodadas)

Sem o overlay a skill funciona normal, só que com defaults genéricos. Com as regras do seu domínio declaradas ela rende **muito** mais — é a diferença entre "revisor que acabou de chegar na empresa" e "revisor que conhece o sistema".

> A retrospectiva automática grava as melhorias de checklist em `docs/triple-review-tuning/checklist-overrides.md` — no **seu projeto**, nunca no plugin. Assim as lacunas do seu domínio viram itens de procedimento sem afetar quem usa o plugin em outro projeto. Quando uma melhoria é genérica (serviria pra qualquer projeto), ela fica marcada `UPSTREAM?` no CHANGELOG local — aí é só abrir um PR aqui pra promover ela pro motor.

## Uso

```
/triple-review [descrição do fluxo de usuário]
```

A descrição é opcional — sem argumento, a skill infere o contexto da branch, dos commits e da pasta `docs/` da branch.

Exemplo:

```
/triple-review reserva de material — operador aprova a reserva e registra a entrega à fábrica
```

## Atualização

Quando eu subir melhorias aqui, é só rodar:

```
/plugin update triple-review@code-review
```

## Contribuindo

Melhorias são bem-vindas via PR. Regras da casa:

- O motor (comando + agentes) tem que continuar **genérico** — nada específico de um projeto/domínio fora dos templates. Coisa do seu projeto vai no overlay do seu projeto.
- Preserve os invariantes: checklists fechados com IDs estáveis, veredito determinístico, baseline/delta por branch, diff via arquivo (nunca no contexto do orquestrador) e banco **sempre read-only**.
- Mudança em item de checklist mantém o ID; item novo entra com o próximo número livre da categoria.
