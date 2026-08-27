# TBskills

Skills do Claude Code focadas em ThingsBoard (PE self-hosted, mas com notas de CE onde
relevante). Ficam em `.claude/skills/` e são carregadas automaticamente pelo Claude
quando a tarefa combina com a `description` de cada uma.

| Skill | Foco |
|---|---|
| `tb-rest-api` | Autenticação JWT, devices, telemetria, atributos, alarmes, relações — integração via REST API |
| `tb-rule-engine` | Estrutura de Rule Chains, tipos de node, TBEL/JS, debug |
| `tb-widgets-dashboards` | Desenvolvimento de widgets customizados e configuração de dashboards |
| `tb-deploy-admin` | Docker Compose, banco de dados, licenciamento PE, upgrade, backup, HA, troubleshooting |

Cada skill tem um `SKILL.md` enxuto com o essencial + uma pasta `references/` com
tabelas/exemplos mais extensos, carregados sob demanda.

## Princípio comum às 4 skills

ThingsBoard muda rápido entre versões e entre CE/PE (endpoints, `type` de rule node,
schema de widget, scripts de upgrade). Por isso cada skill começa com uma seção
"Antes de começar" que prioriza **verificar contra a instância real** (Swagger, export
de rule chain/dashboard, doc oficial da versão) em vez de confiar cegamente em conteúdo
memorizado — usar o conteúdo das skills como ponto de partida, não como verdade absoluta.

## Como estender

Para adicionar uma nova skill, criar `.claude/skills/<nome>/SKILL.md` com frontmatter
`name` + `description` e, se necessário, uma subpasta `references/` para material mais
extenso que não precisa estar sempre carregado.

## Licença

[MIT](LICENSE).
