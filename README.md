# TBskills

Skills de ThingsBoard para Claude Code — e as ferramentas que fazem o agente
**verificar em vez de adivinhar**.

| Skill | Foco |
|---|---|
| `tb-rest-api` | JWT, devices, telemetria, atributos, alarmes, relações, RPC |
| `tb-rule-engine` | Rule chains, catálogo de nodes, `connections[].type`, TBEL, debug |
| `tb-widgets-dashboards` | Widgets custom, `controllerScript`, datasources, entity alias, design |
| `tb-deploy-admin` | Docker Compose, bancos, Kafka, licença PE, upgrade, backup, HA |

Cada skill tem um `SKILL.md` enxuto + `references/` carregado sob demanda. Foco em PE
self-hosted, com notas de CE onde relevante.

## O problema

Um agente que responde sobre ThingsBoard de memória inventa `type` de rule node, endpoint
que não existe naquela versão e `connections[].type` plausível-mas-errado — que **falha em
silêncio**: a conexão nunca dispara e nada aparece no log.

Documentação sozinha não resolve, porque documentação envelhece e ninguém percebe. Este
pack ataca por três lados:

1. **Conteúdo com proveniência fixada** — o catálogo de nodes cita tag e commit SHA exatos
2. **Uma ferramenta que consulta a instância real** — verificar fica mais barato que chutar
3. **CI que detecta quando o upstream diverge** — o conteúdo não apodrece em silêncio

## Ferramentas

Node.js ≥ 18, **zero dependências**. Nada para instalar.

### `tb.mjs` — verificação contra a instância real

Toda skill manda conferir contra a instância. Esta é a ferramenta que torna isso barato o
bastante para o agente realmente fazer, em vez de cair na tabela memorizada.

```bash
cp .tb.env.example .tb.env     # preencha o .tb.env (ignorado pelo git), NUNCA o .example

node scripts/tb.mjs check                     # alcançável, autenticado, CE ou PE, versão
node scripts/tb.mjs api "telemetry"           # busca paths no OpenAPI DESTA instância
node scripts/tb.mjs spec "/api/..." --method post   # params + schema do body
node scripts/tb.mjs nodes                     # rule nodes reais, incluindo os de PE
node scripts/tb.mjs export rulechain "Root Rule Chain" --out chain.json
node scripts/tb.mjs find device "sensor"      # resolve nome -> UUID
node scripts/tb.mjs telemetry <TOKEN> '{"temperature":21.5}'
node scripts/tb.mjs help
```

`api` e `spec` leem o `/v3/api-docs` da própria instância (cache 24h) — fonte da verdade
para a versão instalada. `nodes` lê os descritores de componente reais, o que cobre nodes
PE closed-source que um catálogo de CE não pode ter.

Escrita exige `--yes` explícito. O JWT vai para o tmp do SO; a senha nunca é gravada nem
impressa. `check` degrada em vez de desistir: sem credencial válida ainda reporta
edição e versão a partir do OpenAPI público.

### `verify-nodes.mjs` — anti-apodrecimento do catálogo

Reverifica [node-types.md](.claude/skills/tb-rule-engine/references/node-types.md) contra a
anotação `@RuleNode` das 223 classes de `rule-engine-components`, numa ref fixada.

```bash
node scripts/verify-nodes.mjs                    # diff contra o último release
node scripts/verify-nodes.mjs --ref v4.3.0       # contra uma versão específica
node scripts/verify-nodes.mjs --update-header    # grava tag + commit SHA no cabeçalho
```

O cabeçalho do catálogo carrega a proveniência exata, então "verificado no código-fonte
oficial" é afirmação checável, não alegação.

### `verify-tbel.mjs` — as funções TBEL documentadas existem?

`references/tbel.md` é o maior arquivo do pack. Uma função inventada ali é exatamente o
modo de falha que o pack combate: o agente escreve `date.addMilliseconds(500)` e o script
quebra em runtime dentro de um rule node.

```bash
node scripts/verify-tbel.mjs
```

Cruza os identificadores usados em blocos de código contra quatro fontes upstream:
`TbUtils.java` (globais registradas via `addImport`), `TbDate.java`,
`TbelCfTsRollingArg.java` (janela rolante de Calculated Fields) e o repo separado
`thingsboard/tbel` (métodos de coleção do fork do MVEL).

### `skill-lint.mjs` — a description dispara mesmo?

O modo de falha clássico: o corpo da skill está impecável e a `description` nunca casa,
porque foi escrita no vocabulário do autor e não no do usuário.

```bash
node scripts/skill-lint.mjs               # secrets + validate + triggers
node scripts/skill-lint.mjs triggers --verbose
```

- **`secrets`** — arquivo `.example` só pode conter placeholder, e o arquivo de credencial
  nunca pode estar rastreado
- **`validate`** — frontmatter, tamanho, links relativos, e a armadilha do `: ` não-quotado
  em YAML, que trunca a description em silêncio e mata o disparo da skill
- **`triggers`** — [tests/triggers.jsonl](tests/triggers.jsonl), 47 frases como usuários
  realmente escrevem (`"Can't compile script: null"`, `"docker compose do tb não sobe"`),
  mais negativas que devem *não* disparar

**Limite honesto:** é um scorer léxico, não o modelo. Prova que os termos discriminantes
estão na description certa e não na errada — condição necessária, não suficiente. A
validação real é rodar as frases no agente; este lint é o pré-voo barato de CI.

## O que essas ferramentas já encontraram

Todas achadas contra o código-fonte upstream, não contra documentação:

| Onde | O quê |
|---|---|
| `tbel.md` | `date.addMilliseconds()` não existe — a menor unidade é `addNanos` |
| `tbel.md` | os `add*` de `TbDate` são `void`; o doc dizia que retornavam o valor |
| `tbel.md` | `createSet()` não existe — as construtoras são `newSet()` e `toSet(list)` |
| `node-types.md` | faltavam `calculate delta` e `azure iot hub` |
| `node-types.md` | FQN abreviado (`...TbSynchronizationEndNode`) que um agente copiaria literal |
| `tb-rest-api` | `scope` em `.../timeseries/{scope}` é path param, não `?scope=ANY` |
| descriptions | 12 de 47 frases não casavam ou casavam com a skill errada (62% → 100%) |

## Configuração

```bash
cp .tb.env.example .tb.env
```

Preencha **apenas** o `.tb.env` — ele está no `.gitignore`. O `.tb.env.example` é
rastreado pelo git; credencial nele vai para o repositório. `skill-lint secrets` falha o
build se isso acontecer.

| Variável | |
|---|---|
| `TB_URL` | `https://host`, sem barra no fim |
| `TB_USER` | prefira um usuário técnico com role restrita, não admin humano |
| `TB_PASSWORD` | |
| `TB_INSECURE=1` | só para instância interna com certificado self-signed |

## CI

[`.github/workflows/verify.yml`](.github/workflows/verify.yml) roda os lints em cada
push/PR e o diff do catálogo mensalmente, abrindo issue quando o upstream diverge. Só
divergência de `connections[].type` quebra o build — nodes novos são informativos. O
caminho de alerta é testável sob demanda via `workflow_dispatch`.

## Estado da verificação

Explicitar o que foi e o que não foi verificado é parte do produto.

**Verificado end-to-end:** catálogo de nodes contra `v4.3.1.4`; funções TBEL contra quatro
fontes upstream; 47/47 frases no scorer léxico; `tb.mjs` contra ThingsBoard 4.3 real (CE
demo e uma instância PE 4.3.1.3PE — alcançabilidade, detecção CE/PE, versão, `api`, `spec`);
comandos autenticados contra um mock da API (login, refresh, retry de 401, paginação,
export, descritores, transporte de device); o caminho de drift do CI no Actions real,
incluindo a abertura automática de issue.

**Não verificado:** comandos autenticados contra uma instância PE real; as frases de
gatilho no agente de verdade (o scorer é léxico); e a prosa de
`widget-controller-api.md` / `widget-design-guide.md`, que ainda não têm verificador.

## Estender

Nova skill: `.claude/skills/<nome>/SKILL.md` com frontmatter `name` + `description`,
`references/` para material extenso. Adicione frases-gatilho em `tests/triggers.jsonl` e
rode `node scripts/skill-lint.mjs` antes de commitar.

## Licença

[MIT](LICENSE).
