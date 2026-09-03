# TBskills

Skills de ThingsBoard para Claude Code (PE self-hosted, com notas de CE onde relevante),
mais as ferramentas que fazem o agente **verificar em vez de adivinhar**.

| Skill | Foco |
|---|---|
| `tb-rest-api` | JWT, devices, telemetria, atributos, alarmes, relações, RPC |
| `tb-rule-engine` | Rule chains, catálogo de nodes, `connections[].type`, TBEL, debug |
| `tb-widgets-dashboards` | Widgets custom, `controllerScript`, datasources, entity alias, design |
| `tb-deploy-admin` | Docker Compose, bancos, Kafka, licença PE, upgrade, backup, HA |

Cada skill tem um `SKILL.md` enxuto + `references/` carregado sob demanda.

## O problema que este pack resolve

Um agente que responde sobre ThingsBoard de memória inventa `type` de rule node, `endpoint`
que não existe naquela versão, e `connections[].type` plausível-mas-errado — que **falha em
silêncio**: a conexão nunca dispara e nada aparece no log.

Documentação sozinha não resolve isso, porque documentação envelhece. O pack ataca por três
lados: conteúdo verificado com proveniência fixada, uma ferramenta que consulta a instância
real, e CI que detecta quando o upstream diverge.

## Ferramentas

Node.js ≥ 18, **zero dependências**. Não precisa instalar nada.

### `scripts/tb.mjs` — verificação contra a instância real

Toda skill manda conferir contra a instância. Esta é a ferramenta que torna isso barato o
bastante para o agente realmente fazer, em vez de cair na tabela memorizada.

```bash
cp .tb.env.example .tb.env    # preencha; está no .gitignore

node scripts/tb.mjs check                    # alcançável, autenticado, CE ou PE, versão
node scripts/tb.mjs api "telemetry"          # busca paths no OpenAPI DESTA instância
node scripts/tb.mjs spec "/api/..." --method post   # params + schema do body
node scripts/tb.mjs nodes                    # rule nodes reais, incluindo os de PE
node scripts/tb.mjs export rulechain "Root Rule Chain" --out chain.json
node scripts/tb.mjs find device "sensor"     # resolve nome -> UUID
node scripts/tb.mjs telemetry <TOKEN> '{"temperature":21.5}'
node scripts/tb.mjs help
```

`api`/`spec` leem o `/v3/api-docs` da instância (cache 24h) — fonte da verdade para a
versão instalada, em vez de qualquer tabela. `nodes` lê os descritores de componente reais,
o que cobre nodes PE closed-source que o catálogo CE não pode ter.

Escrita exige `--yes` explícito. O JWT vai para o tmp do SO com permissão restrita; a senha
nunca é gravada nem impressa.

### `scripts/verify-nodes.mjs` — anti-apodrecimento do catálogo

Reverifica [node-types.md](.claude/skills/tb-rule-engine/references/node-types.md) contra a
anotação `@RuleNode` do código-fonte upstream, numa ref fixada.

```bash
node scripts/verify-nodes.mjs                    # diff contra o último release
node scripts/verify-nodes.mjs --ref v4.3.0       # contra uma versão específica
node scripts/verify-nodes.mjs --update-header    # grava tag + commit SHA no cabeçalho
```

O cabeçalho do catálogo carrega a proveniência exata (tag, commit, data), então "verificado
no código-fonte oficial" é uma afirmação checável, não uma alegação.

### `scripts/verify-tbel.mjs` — as funções TBEL documentadas existem?

`references/tbel.md` é o maior arquivo do pack. Uma função inventada ali é exatamente o
modo de falha que o pack existe para evitar — o agente escreve `date.addMilliseconds(500)`
e o script quebra em runtime dentro de um rule node.

```bash
node scripts/verify-tbel.mjs                  # contra o último release
node scripts/verify-tbel.mjs --ref v4.3.1.4 --json
```

Cruza os identificadores usados em blocos de código do doc contra quatro fontes upstream:
`TbUtils.java` (as globais registradas via `addImport`), `TbDate.java`,
`TbelCfTsRollingArg.java` (janela rolante de Calculated Fields) e o repo separado
`thingsboard/tbel` (métodos de coleção do fork do MVEL).

Achou dois erros reais na primeira execução: `addMilliseconds` (só existe `addNanos`) e
`createSet` (as construtoras são `newSet()` e `toSet(list)`).

### `scripts/skill-lint.mjs` — a description casa com o que o usuário digita?

O modo de falha clássico de skill: o corpo está impecável e a `description` nunca dispara,
porque foi escrita no vocabulário do autor e não no do usuário.

```bash
node scripts/skill-lint.mjs            # validate + triggers
node scripts/skill-lint.mjs triggers --verbose
```

`validate` checa frontmatter, tamanho, links relativos e a armadilha do `: ` não-quotado em
YAML (que trunca a description em silêncio e mata o disparo da skill).

`triggers` roda [tests/triggers.jsonl](tests/triggers.jsonl) — 47 frases como usuários
realmente escrevem (`"Can't compile script: null"`, `"docker compose do tb não sobe"`,
`"meu widget não atualiza"`), mais casos negativos que devem *não* disparar.

**Limite honesto:** é um scorer léxico, não o modelo. Prova que os termos discriminantes
existem na description certa e não na errada — condição necessária, não suficiente. A
validação de verdade é rodar as frases no agente (`claude plugin eval` / `/skill-doctor`);
este lint é o pré-voo barato de CI.

## CI

[`.github/workflows/verify.yml`](.github/workflows/verify.yml) roda o lint em cada push/PR
e o diff do catálogo mensalmente, abrindo issue quando o upstream diverge. Só divergência
de `connections[].type` quebra o build — nodes novos são informativos.

## Princípio

ThingsBoard muda rápido entre versões e entre CE/PE. Cada skill abre com "Antes de começar"
priorizando **verificar contra a instância real** — e agora aponta o comando concreto para
isso. Conteúdo do pack é ponto de partida; a instância é a verdade.

## Estender

Nova skill: `.claude/skills/<nome>/SKILL.md` com frontmatter `name` + `description`,
`references/` para o material extenso. Adicione frases-gatilho em `tests/triggers.jsonl` e
rode `node scripts/skill-lint.mjs` antes de commitar.

## Licença

[MIT](LICENSE).
