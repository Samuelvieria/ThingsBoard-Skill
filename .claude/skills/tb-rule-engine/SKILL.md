---
name: tb-rule-engine
description: Criação, edição e depuração de Rule Chains do ThingsBoard (filtros, enriquecimento, transformação em TBEL/JS, ações, roteamento de mensagens). Use ao construir, revisar ou depurar a lógica de processamento de mensagens (rule nodes) em JSON, ou ao explicar/planejar o fluxo de uma rule chain.
---

# ThingsBoard Rule Engine

## Antes de começar

Os `type` de cada rule node (ex: `org.thingsboard.rule.engine.filter.TbJsFilterNode`) são
nomes de classe Java internos que **mudam entre versões e entre CE/PE**, e um `type`
incorreto falha silenciosamente ou dá erro ao importar. Sempre que possível:

1. Peça para o usuário **exportar** uma rule chain existente da instância
   (Rule Chains → abrir → menu → Export) e use esse JSON como template real/verdadeiro
   em vez de escrever os nodes inteiramente de memória.
2. Se não houver rule chain de referência disponível, use a tabela em
   [references/node-types.md](references/node-types.md) como ponto de partida, mas avise
   que os `type` exatos devem ser confirmados na paleta de nodes da instância antes de
   importar via API/UI.
3. Prefira editar a estrutura de um chain existente (adicionar/remover nodes e
   conexões) a recriar o chain do zero.

## Estrutura do JSON de uma Rule Chain

Estrutura confirmada inspecionando rule chains reais via `GET /api/ruleChain/{id}/metadata`
em uma instância PE (2026-08-26) — mais precisa que assumir pela doc genérica:

```json
{
  "ruleChain": {
    "name": "...", "type": "CORE", "root": false, "debugMode": false,
    "firstRuleNodeId": { "entityType": "RULE_NODE", "id": "..." }
  },
  "metadata": {
    "ruleChainId": { "entityType": "RULE_CHAIN", "id": "..." },
    "firstNodeIndex": 0,
    "nodes": [
      {
        "id": { "entityType": "RULE_NODE", "id": "..." },
        "ruleChainId": { "entityType": "RULE_CHAIN", "id": "..." },
        "type": "org.thingsboard.rule.engine.filter.TbMsgTypeSwitchNode",
        "name": "Switch por tipo de mensagem",
        "debugSettings": { "failuresEnabled": true, "allEnabled": false, "allEnabledUntil": 0 },
        "singletonMode": false,
        "queueName": null,
        "configurationVersion": 0,
        "configuration": { },
        "additionalInfo": { "description": "", "layoutX": 300, "layoutY": 200 }
      }
    ],
    "connections": [
      { "fromIndex": 0, "toIndex": 1, "type": "Success" }
    ],
    "ruleChainConnections": null
  }
}
```

- `connections[].type` é o **label da relação de saída** do node de origem (ex:
  `Success`, `Failure`, `True`, `False`, ou labels específicos do node como
  `Post telemetry`/`Post attributes` de um Message Type Switch) — precisa bater
  exatamente com os labels que aquele tipo de node emite.
- **Debug por node é `debugSettings` (objeto), não um `debugMode` booleano** —
  `failuresEnabled` liga debug só em falhas, `allEnabled`/`allEnabledUntil` ligam debug
  completo com expiração automática (proteção contra esquecer debug ligado em produção).
  O `debugMode` booleano simples continua existindo, mas só no nível do **rule chain**
  (objeto `ruleChain`), não em cada node.
- `queueName` (nulo por padrão = usa a fila do rule chain) permite rotear um node
  específico para uma fila diferente (ex. `HighPriority`) — útil para isolar
  processamento de alarme/notificação da fila principal de telemetria. Ver conceito de
  filas em detalhe na skill `tb-deploy-admin`.
- `additionalInfo.layoutX/layoutY` guarda a posição visual do node no editor — omitir
  não quebra a lógica, mas o node aparece empilhado na origem ao importar.
- **Encadear chains** (composição): na instância inspecionada, isso é feito com um node
  comum do tipo `org.thingsboard.rule.engine.flow.TbRuleChainInputNode` (categoria Flow,
  nome exibido "Rule Chain") ligado por uma `connection` normal, com
  `configuration: { "ruleChainId": "<uuid-do-chain-alvo>", "forwardMsgToDefaultRuleChain": false }`
  — **não** pelo array `ruleChainConnections` (que apareceu `null` em todas as chains
  reais inspecionadas; trate-o como possivelmente legado/não usado nesta versão PE).
- `root: true` marca o rule chain padrão do tenant (recebe mensagens quando o device
  profile não aponta para um chain específico).

## Categorias de nodes

O Rule Engine organiza os nodes em **7 categorias** (confirmado na doc oficial):

| Categoria | Papel | Exemplos |
|---|---|---|
| **Filter** | Roteia sem alterar a mensagem (`True`/`False` ou conexão nomeada) | Script, Switch, Message Type Filter/Switch, Check Relation Presence, GPS Geofencing Filter |
| **Enrichment** | Adiciona dados à `metadata` sem alterar `msg` | Originator Attributes, Related Entity Data, Customer/Tenant Attributes, Originator Telemetry |
| **Transformation** | Reescreve `msg`/`metadata`/`msgType` | Script, JSON Path, Rename/Delete/Copy Key-Value Pairs, Change Originator, Split Array Msg |
| **Action** | Efeito colateral (persistência, alarme, controle) | Save Time Series, Save Attributes, Create/Clear Alarm, RPC Call Request, Log |
| **External** | Publica em sistemas fora do ThingsBoard | REST API Call, Send Email/SMS/Slack, Kafka, MQTT, AWS SNS/SQS/Lambda, AI Request |
| **Flow** | Controla o caminho da mensagem entre chains/filas | Rule Chain, Output, Checkpoint, Acknowledge |
| **Analytics** | Agregações/estatísticas sobre streams | aggregate stream, aggregate latest, alarms count |

Catálogo completo com descrição de cada node em
[references/node-types.md](references/node-types.md).

## Script nodes (TBEL / JS) — Script Filter e Script Transformation

Variáveis disponíveis dentro do script: `msg` (payload, JSON parseado), `metadata`
(map string→string), `msgType` (string).

- **Script Filter** — deve retornar `boolean`:
  ```tbel
  return msg.temperature > 30;
  ```
  `true` → conexão `True`; `false` → `False`; exceção ou retorno não-boolean → `Failure`.

- **Script Transformation** — deve retornar um objeto com as três partes (ou um array de
  objetos para fan-out, uma mensagem de entrada virando N de saída):
  ```tbel
  return { msg: msg, metadata: metadata, msgType: msgType };
  ```
  Saídas: `Success` / `Failure`.

- **TBEL** (ThingsBoard Expression Language) é um fork do MVEL (não JS completo), roda em
  sandbox com limite de memória por execução e tem overhead de inicialização
  desprezível comparado ao engine JS (Nashorn) — é a linguagem **recomendada** nos
  seletores de linguagem dos Script nodes. Não permite instanciar classes Java
  diretamente (`new java.util.ArrayList()` falha), só chamar métodos estáticos.
  Referência completa de sintaxe e funções helper (string/bytes/hex/base64/data/geofencing)
  em [references/tbel.md](references/tbel.md).
- Chaves de `metadata` são sempre strings — números/booleans em metadata chegam como
  string e precisam de cast explícito no script (`parseDouble(metadata.threshold)` em
  TBEL, `Number(metadata.threshold)` em JS).
- Scripts rodam em ambiente isolado: **não é possível importar bibliotecas externas**.
- **`configuration` real de um Script Transformation node** (confirmado inspecionando um
  node real via API) guarda os dois scripts simultaneamente e alterna por `scriptLang` —
  útil para gerar/editar isso via REST API sem passar pela UI:
  ```json
  {
    "scriptLang": "TBEL",
    "jsScript": "return {msg: msg, metadata: metadata, msgType: msgType};",
    "tbelScript": "return {msg: msg, metadata: metadata, msgType: msgType};"
  }
  ```
  Ambos os campos (`jsScript`/`tbelScript`) costumam estar preenchidos mesmo quando só um
  é usado (o editor guarda o outro para o caso de trocar de linguagem depois); só o valor
  apontado por `scriptLang` é executado.

## Calculated Fields (fora do Rule Engine)

Feature separada dos rule chains, associada a um device/asset profile — útil quando o
cálculo é o mesmo para todos os devices de um profile, sem precisar de um rule chain
dedicado. Dois tipos: **Simple** (uma expressão matemática, sem script — comece por
aqui para conversão/normalização simples) e **Script** (TBEL completo, condicionais,
janelas históricas/rolling, múltiplas saídas). Detalhes e exemplos de ambos em
[references/tbel.md](references/tbel.md#calculated-fields--simple-vs-script).

## Padrões comuns

- **Roteamento por tipo de device**: node "Switch" logo após a entrada, com um branch
  por `deviceType`/`deviceProfile`, cada branch fazendo sua própria normalização.
- **Normalização de payload**: Transformation node convertendo payloads heterogêneos de
  devices diferentes para um schema comum antes de "Save Timeseries".
- **Geração de alarme com histerese**: Script Filter comparando o valor atual contra
  threshold + verificação de atributo "already alarmed" para evitar reabrir o alarme a
  cada mensagem; "Clear Alarm" no branch em que a condição volta ao normal.
- **Enriquecimento antes de chamada externa**: "Originator Attributes"/"Related Entity
  Data" para juntar contexto (ex: dados do customer) antes de um "REST API Call" node.
- **Fan-out**: um node com múltiplas conexões de saída do mesmo tipo (`Success`) para
  paralelizar, ex.: salvar telemetria E checar alarme ao mesmo tempo a partir do mesmo node.

## Debug

- Ativar "Debug" no rule chain (ou por node) liga o registro de eventos.
- Aba **Events** de cada node mostra mensagens de entrada (`In`) e saída (`Out`), erros
  de script e o `metadata`/`msgType` em cada etapa — é a forma mais rápida de achar por
  que uma mensagem não seguiu o caminho esperado.
- Debug mode tem custo de performance/storage — desligar em produção depois de investigar.

## PE

Professional Edition/Cloud tem nodes exclusivos além do CE, confirmados na doc oficial —
marcados **(PE/Cloud)** em [references/node-types.md](references/node-types.md). Os mais
relevantes: Duplicate to Group / Duplicate to Group by Name / Duplicate to Related
(Transformation), Add to Group / Change Owner / Generate Report / Generate Dashboard
Report / Save to Custom Table / Integration Downlink / Remove from Group (Action), e
Twilio SMS / Twilio Voice (External). Ainda assim, confirmar na paleta de nodes da
instância antes de assumir disponibilidade, pois isso pode mudar por versão.

## Referência

- [references/node-types.md](references/node-types.md) — catálogo completo de nodes por
  categoria (Filter, Enrichment, Transformation, Action, External, Flow, Analytics).
- [references/tbel.md](references/tbel.md) — referência completa da linguagem TBEL
  (tipos, controle de fluxo, manipulação de Maps/Lists/Sets, funções helper, Calculated
  Fields).
