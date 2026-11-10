---
name: qlt-init-review
description: Review multiagente do diff atual ou de um PR. Pergunta quais revisores rodar (code, security, performance, tests, qa), dispara como subagents, grava uma fase por arquivo em .qlt/review/ e retoma de onde parou se a sessão cair.
argument-hint: "[número do PR]"
disable-model-invocation: true
---

# /qlt-init-review

Argumento: `$ARGUMENTS` (número do PR; vazio = diff da branch atual).

## Regras duras
1. Execute as fases em ordem. Cada fase grava o seu arquivo antes de a próxima começar.
2. Leia o estado dos arquivos em `.qlt/review/`, não da memória do contexto.
3. Se algo falhar, pare e pergunte.
4. Nunca edite o código do projeto. Nunca entre em plan mode.
5. Nunca escolha os revisores sozinho.

## Convenções
- `RAIZ` = `git rev-parse --show-toplevel`. Saída em `RAIZ/.qlt/review/`; execuções antigas em `RAIZ/.qlt/archive/<ts>/` (`ts` = `date +%Y%m%d-%H%M%S`). O `.qlt/` deve estar ignorado pelo git: antes de gravar, rode `git check-ignore -q .qlt/review/state.json`; se falhar, avise que falta `.qlt/` no ignore (global, `~/.config/git/ignore`, ou o `.gitignore` do repo) e pergunte se continua (os relatórios apareceriam no `git status`).
- Perguntar = `AskUserQuestion`. Sem essa ferramenta (outros clients), use lista numerada em texto e espere a resposta.
- `contract.md` fica na mesma pasta deste `SKILL.md`, `${CLAUDE_SKILL_DIR}` (o "Base directory" mostrado ao carregar a skill; sem ele, procure `qlt-init-review/contract.md` em `~/.claude/skills`, `~/.codex/skills` ou `~/.config/skillshare/skills`). `report-template.html`, o modelo do relatório visual (etapa 5), fica na mesma pasta e é achado do mesmo jeito.
- `KIT_AGENTS` = a pasta com os papéis `qlt-*.md` do kit: `${CLAUDE_SKILL_DIR}/../../agents/` (instalação como plugin ou clone do repo); se não existir, o primeiro entre `~/.claude/agents`, `~/.cursor/agents` e `~/.config/skillshare/agents` que tenha `qlt-*.md`.
- Nomes: instalado como plugin do Claude Code, o kit aparece como `/qlt:qlt-init-review` e os agents como `qlt:qlt-<dim>`; nas demais instalações, `/qlt-init-review` e `qlt-<dim>`. Use o nome exatamente como o client lista.
- Arquivo de cada revisor: code `10-code.md` · security `11-security.md` · performance `12-performance.md` · tests `13-tests.md` · qa `20-qa.md`. Revisor extra do projeto: `14-<dim>.md`, `15-…`, na ordem escolhida. `dim` = nome do agent sem o prefixo `qlt-`.
- `state.json`:
```json
{"status":"in_progress|complete","phase":"scope|choose|review1|review2|final|done","pr":null,"base":"main","head_sha":"<sha>","reviewers":{"code":{"status":"pending|done|failed","file":"10-code.md","source":"kit|project"}}}
```

## 0. Retomada
Se `.qlt/review/state.json` existe:
- `status: complete` → mova a pasta para `.qlt/archive/<ts>/` e siga para a etapa 1 (avise que arquivou).
- `status: in_progress` → antes de perguntar, **reconcilie com o disco**: revisor `pending` cujo arquivo existe e tem frontmatter com `verdict` vira `done`. Mostre a fase e os revisores `done`/`pending`/`failed`, e pergunte:
  - **Retomar** (roda só `pending` e `failed`; sem revisores gravados, volte à etapa 2);
  - **Começar do zero** (arquiva a pasta e vai para a etapa 1).
- Se `head_sha` ≠ `git rev-parse HEAD`, avise que os achados podem estar velhos, refaça o `00-scope.md` e ofereça uma terceira opção: **Retomar e refazer também os concluídos**.

## 1. Escopo → `00-scope.md` + `state.json`
- Com PR: diff = `gh pr diff <n>`. Compare `gh pr view <n> --json headRefOid` com o HEAD local; se diferir, avise que os revisores leem o código local, que não é o do PR.
- Sem PR: diff = `git diff` (não commitado) mais `git diff <base>...HEAD`. `base` = `git symbolic-ref --short refs/remotes/origin/HEAD` sem o prefixo `origin/`; sem isso, `main`.
- Diff vazio → avise e encerre.
- Grave em `00-scope.md`: o(s) comando(s) exatos do diff, `git diff --stat`, a lista de arquivos, `head_sha` e quais `RAIZ/.claude/qlt/*.md` existem. Grave `state.json` com `phase: choose`.

## 2. Escolha dos revisores (sempre)
- Descubra os `qlt-*.md` em `RAIZ/.claude/agents/` (projeto) e em `KIT_AGENTS` (kit). Mesmo nome → vale o do projeto.
- Pergunte com `AskUserQuestion` multiSelect: um item por revisor, com a `description` e "(projeto)" ou "(kit)". Cada pergunta aceita 2 a 4 opções: divida em perguntas equilibradas na mesma chamada (5 revisores = 3 + 2).
- Nada marcado: encerre. Grave a escolha em `state.json` (todos `pending`, `phase: review1`).

## 3. Fase 1 — todos os escolhidos, exceto `qa`, em paralelo
Dispare todos na **mesma mensagem**. Prompt de cada um:
```
Você é o revisor qlt-<dim>.
<texto integral de contract.md>
RAIZ: <RAIZ>
SCOPE: <RAIZ>/.qlt/review/00-scope.md
OUT: <RAIZ>/.qlt/review/<arquivo>
OUTROS: <demais revisores escolhidos, ou "nenhum">
REGRAS: <RAIZ>/.claude/qlt/common.md, <RAIZ>/.claude/qlt/<dim>.md (só os que existem, ou "nenhuma")
```
- **Claude e Cursor:** `Agent`/subagent. O `subagent_type` é `qlt-<dim>` quando o revisor veio do projeto (o agent de `.claude/agents/`) e o nome do kit quando veio de `KIT_AGENTS` (`qlt-<dim>` ou `qlt:qlt-<dim>`, como o client lista). Como o plugin tem prefixo, o agent do projeto não "sobrescreve" o do kit sozinho: a escolha é sua, pela origem. O papel já vem no agent; o prompt acima traz o resto.
- **Client sem agents (Codex, Grok, Hermes):** leia o papel em `RAIZ/.claude/agents/qlt-<dim>.md` ou, sem ele, `KIT_AGENTS/qlt-<dim>.md`, tire o frontmatter e use o corpo como prompt de um subagent genérico, antes do bloco acima. Sem subagents, rode os papéis **em sequência** você mesmo, cada um gravando o seu arquivo.
- O próprio revisor grava o seu arquivo. Quando cada um retornar, confira que o arquivo existe com `verdict` e marque `done`; senão `failed`. Atualize `state.json` a cada retorno.

## 4. Fase 2 — `qa`, se escolhido
Só depois da fase 1, para validar os testes novos do `qlt-tests` e não disputar CPU com ele. Mesmo prompt, com a linha extra `TESTES NOVOS: <RAIZ>/.qlt/review/13-tests.md (se existir)`. Grava `20-qa.md`. `phase: review2`.

## 5. Consolidação → `90-final.json`, `90-final.html` e `90-final.md`
Leia **só** os arquivos de `.qlt/review/`. A fonte é o `90-final.json`; o relatório visual e o resumo em texto saem dele, com os mesmos números.

**Decida:**
- Veredito global: o pior entre os revisores (`blocked` > `changes_requested` > `approved`); revisor `failed` conta como `blocked` ("revisão incompleta").
- Achados: funda os duplicados entre revisores e ordene por severidade (critical → info). Os totais do final contam **depois de fundir**; os de cada revisor ficam como estão no arquivo dele.
- Até 3 regras propostas para `.claude/qlt/*.md`. Grave-as só com aprovação do usuário (pergunta).

**1. Grave `90-final.json`** (JSON válido; texto puro, sem HTML; `` `trecho` `` vira código; campo vazio ou ausente esconde a seção):
```json
{"title":"Revisão do PR #<n> · <título>","subtitle":"<o que a mudança faz, 1 linha>","verdict":"approved|changes_requested|blocked",
 "meta":["branch `x` → `main`","head `abc1234`","<N> arquivos · +<A> / −<R>","revisores: code, qa"],"countNote":"<opcional>",
 "reviewers":[{"k":"code","v":"approved","c":{"critical":0,"high":0,"medium":0,"low":0,"info":0},"d":"<1 linha>"}],
 "findings":[{"sev":"critical|high|medium|low|info","rev":["code","qa"],"conf":"média (code) · alta (qa)","title":"…","where":["arquivo:linha"],"problem":"…","evidence":"…","fix":"…"}],
 "questions":["…"],"checks":[["comando","resultado","ok|warn|fail"]],"qaNote":"<opcional>",
 "extra":[{"t":"<título>","items":["…"]}],"gaps":["<o que ficou de fora, inclusive revisores dispensados>"],
 "rules":[["code.md","<regra proposta>"]],"rulesNote":"Não gravadas: aguardam aprovação."}
```
`checks` e `qaNote` vêm do `20-qa.md`; `extra` reúne outras verificações dos revisores (casos exercitados, mutações etc.). `evidence`, `fix` e `conf` podem ser `""`.

**2. Gere `90-final.html`** com o modelo do kit, `report-template.html` (mesma pasta de `contract.md`). Rode exatamente este comando, com `R` = `RAIZ/.qlt/review` e `T` = o caminho do modelo:
```bash
sed 's/</\\u003c/g' "$R/90-final.json" > "$R/.final.tmp" &&
awk 'FNR==NR{d=d $0 "\n"; next} /^\/\*QLT_DATA\*\/null$/{printf "%s", d; next} {print}' "$R/.final.tmp" "$T" > "$R/90-final.html"; rm -f "$R/.final.tmp"
```
O modelo não tem dados: o comando só troca a linha `/*QLT_DATA*/null` pelo JSON (o `sed` escreve `<` como `<` para o conteúdo nunca fechar o `<script>`), e sai um arquivo único, que abre sem servidor. Confira que `grep -c QLT_DATA "$R/90-final.html"` dá 0 e valide o JSON com `jq empty`, `python3 -m json.tool` ou `node`, o que houver. **Sempre use este modelo**: não escreva nem edite o HTML à mão e não invente outro layout.

**3. Grave `90-final.md`**, o mesmo conteúdo em markdown, curto, para quem lê no terminal ou é um agente. Frontmatter: `verdict`, `head_sha`, `reviewers` (veredito de cada), `counts` (os do JSON). Corpo: resumo por revisor · achados por severidade com `arquivo:linha`, problema, correção mínima e quem achou · perguntas para o dev · quem rodou e quem foi dispensado · regras propostas.

**4.** `state.json`: `status: complete`, `phase: done`.

## 6. Entrega
- Fora do Orca (`$ORCA_TERMINAL_HANDLE` vazio) → só mostre o resumo.
- `orca terminal list --json`: terminais com o mesmo `worktreePath` e `handle` diferente do seu. **Exatamente um** → é o dev. Nenhum ou mais de um → não adivinhe: mostre o resumo e diga o motivo.
- Com o dev: `orca terminal wait --terminal <DEV> --for tui-idle --timeout-ms 600000`. Ocioso → `orca terminal send --terminal <DEV> --enter --text "<uma linha>"`, sem aspas duplas dentro: veredito, nº de achados critical/high e `Detalhes em .qlt/review/90-final.md (visual: 90-final.html)`. Timeout → não force e não use `--enter`.
- Termine mostrando ao usuário: veredito, contagem por revisor e o link do relatório visual, `file://<RAIZ>/.qlt/review/90-final.html` (com o caminho de `90-final.md` ao lado).
