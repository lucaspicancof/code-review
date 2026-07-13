# code-review

Plugins de revisão de código para [Claude Code](https://claude.com/claude-code).

## Plugins

### triple-review

Revisão pré-build em paralelo com três revisores especializados rodando simultaneamente:

- **QA** — corretude funcional, casos de fronteira, bloqueios de negócio ausentes e falhas silenciosas (checklist `QA-*`, 25 itens)
- **Sênior** — design/arquitetura, corretude e regressão, convenções do projeto e manutenibilidade, com verificação ativa no codebase (checklist `SR-*`)
- **Gerente UX** — fluxo, clareza, perda de trabalho e consistência do ponto de vista do usuário final, ancorado nas 10 Heurísticas de Nielsen (grade `UX-*`)

O orquestrador consolida com deduplicação, gate de confiança e **baseline/delta por branch**: cada achado sai rotulado `NOVO` / `REGRESSÃO` / `PRÉ-EXISTENTE` / `PERSISTENTE`, o veredito depende do que **mudou** desde a última revisão (dívida antiga não trava build), e uma **retrospectiva automática** converte lacunas de cobertura em melhorias de checklist do projeto.

## Instalação

No Claude Code:

```
/plugin marketplace add lucaspicancof/code-review
/plugin install triple-review@code-review
```

Pronto — `/triple-review` fica disponível em qualquer projeto.

## Configuração por projeto (recomendado)

O plugin é o **motor genérico**. O conteúdo específico do seu projeto (regras de negócio críticas, convenções de código, perfis de usuário, padrões de UI) vive num **overlay dentro do projeto**:

1. Copie [`plugins/triple-review/templates/customizacao.md`](plugins/triple-review/templates/customizacao.md) para `docs/triple-review-tuning/customizacao.md` no seu projeto
2. Preencha as seções (os títulos `##` são fixos — os agentes os citam pelo nome)
3. Recomendado: adicione `docs/triple-review-tuning/.baselines/` ao `.gitignore`

Sem overlay a skill funciona com defaults genéricos — mas rende muito mais com as regras do seu domínio declaradas.

> A retrospectiva automática grava melhorias de checklist em `docs/triple-review-tuning/checklist-overrides.md` (no **projeto**, nunca no plugin) — suas lacunas viram itens de procedimento sem afetar outros projetos. Melhorias genéricas ficam marcadas `UPSTREAM?` no CHANGELOG local: abra um PR aqui para promovê-las ao motor.

## Uso

```
/triple-review [descrição do fluxo de usuário — opcional; sem argumento, o contexto é inferido de branch/commits/docs]
```

## Atualização

```
/plugin update triple-review@code-review
```

## Contribuindo

Melhorias são bem-vindas via PR. Regras da casa:
- O motor (comando + agentes) deve permanecer **genérico** — nada específico de um projeto/domínio fora dos templates
- Preserve os invariantes: checklists fechados com IDs estáveis, veredito determinístico, baseline/delta, diff via arquivo (nunca no contexto do orquestrador), banco read-only
- Mudança em item de checklist deve manter o ID; item novo entra com o próximo número livre da categoria
