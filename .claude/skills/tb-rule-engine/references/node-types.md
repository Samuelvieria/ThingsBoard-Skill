# Catálogo de rule nodes (fonte: documentação oficial ThingsBoard)

> Extraído de `thingsboard.io/docs` (CE) e `thingsboard.io/docs/pe` (PE) em 2026-08-26.
> Nomes e descrições são fiéis à doc oficial; os `type` internos (nome de classe Java)
> **não são documentados publicamente** — para obter o valor exato a usar em JSON de rule
> chain, exporte o node real da instância (Rule Chains → abrir → Export) ou consulte
> `GET /api/plugins/rulenode` (ver skill `tb-rest-api`).
>
> Itens marcados **(PE/Cloud)** só existem em Professional Edition / ThingsBoard Cloud,
> não em Community Edition — relevante já que a instância de referência é PE.

O Rule Engine organiza os nodes em **7 categorias**: Filter, Enrichment, Transformation,
Action, External, Flow e Analytics.

## `type` confirmados em instância PE real (não são chute)

Estes foram extraídos inspecionando rule chains reais via
`GET /api/ruleChain/{id}/metadata` — maior confiança que o restante da tabela abaixo
(que é aproximação baseada só na doc). Corrige duas suposições erradas da versão
anterior deste arquivo: **Save Time Series/Attributes ficam no pacote `telemetry`, não
`action`**.

| Node (nome exibido) | `type` confirmado |
|---|---|
| Script (Filter) | `org.thingsboard.rule.engine.filter.TbJsFilterNode` (mesma classe independente de TBEL ou JS estar selecionado — a linguagem é um campo dentro de `configuration`, não muda o `type`) |
| Message Type Switch | `org.thingsboard.rule.engine.filter.TbMsgTypeSwitchNode` |
| Message Type Filter | `org.thingsboard.rule.engine.filter.TbMsgTypeFilterNode` |
| Entity Type Filter ("Is Entity Group") | `org.thingsboard.rule.engine.filter.TbOriginatorTypeFilterNode` |
| Originator Attributes | `org.thingsboard.rule.engine.metadata.TbGetAttributesNode` |
| Script (Transformation) | `org.thingsboard.rule.engine.transform.TbTransformMsgNode` |
| Duplicate to Group **(PE)** | `org.thingsboard.rule.engine.transform.TbDuplicateMsgToGroupNode` |
| Save Time Series | `org.thingsboard.rule.engine.telemetry.TbMsgTimeseriesNode` |
| Save Attributes | `org.thingsboard.rule.engine.telemetry.TbMsgAttributesNode` |
| Add to Group **(PE)** | `org.thingsboard.rule.engine.action.TbAddToGroupNode` |
| Log | `org.thingsboard.rule.engine.action.TbLogNode` |
| RPC Call Request | `org.thingsboard.rule.engine.rpc.TbSendRPCRequestNode` |
| Rule Chain (Flow) | `org.thingsboard.rule.engine.flow.TbRuleChainInputNode` — `configuration: {ruleChainId, forwardMsgToDefaultRuleChain}` |

## Filter — decide se/para onde a mensagem segue, sem alterar o conteúdo

| Node | Descrição |
|---|---|
| Alarm Status Filter | Roteia mensagens de alarme por status (ACTIVE_UNACK, ACTIVE_ACK, CLEARED_UNACK, CLEARED_ACK, CLEARED) |
| Asset Profile Switch | Roteia para conexão nomeada conforme o perfil do asset originador |
| Check Fields Presence | Roteia conforme existência de campos específicos em `msg`/`metadata` |
| Check Relation Presence | Roteia conforme existir (ou não) uma relação entre originador e entidade alvo |
| Device Profile Switch | Roteia para conexão nomeada conforme o perfil do device originador |
| Entity Type Filter | Passa/bloqueia conforme o tipo de entidade originadora |
| Entity Type Switch | Roteia para conexão nomeada conforme o tipo de entidade originadora |
| GPS Geofencing Filter | Roteia conforme coordenadas GPS estarem dentro de uma cerca geográfica configurada |
| Message Type Filter | Passa/bloqueia conforme o tipo de mensagem |
| Message Type Switch | Roteia para conexão nomeada conforme o tipo de mensagem |
| Script (Script Filter) | Função TBEL/JS que retorna `boolean` → saídas `True`/`False` (erro/retorno não-boolean → `Failure`) |
| Switch | Função TBEL/JS que retorna uma ou mais conexões de destino (roteamento dinâmico) |

## Enrichment — adiciona contexto à `metadata` sem alterar `msg`

| Node | Descrição |
|---|---|
| Calculate delta | Calcula diferença entre valor atual e anterior (opcionalmente com tempo decorrido) |
| Customer attributes | Injeta atributos server-side ou última telemetria do customer do originador |
| Customer details | Injeta dados do customer (nome, email, endereço) |
| Fetch device credentials | Injeta tipo/valor das credenciais do device originador |
| Originator attributes | Injeta atributos client/shared/server e última telemetria do originador |
| Originator fields | Injeta detalhes da entidade originadora (nome, label, profile) |
| Originator telemetry | Injeta série histórica do originador num intervalo, com agregação opcional |
| Related device attributes | Injeta atributos/telemetria de device relacionado via query de relação |
| Related entity data | Injeta atributos/telemetria/campos de entidade relacionada via query de relação configurável |
| Tenant attributes | Injeta atributos server-side ou última telemetria do tenant |
| Tenant details | Injeta dados do tenant (nome, email, endereço) |

## Transformation — reescreve `msg`/`metadata`/`msgType`

| Node | Descrição |
|---|---|
| Change Originator | Redireciona a mensagem para outra entidade como novo originador |
| Copy Key-Value Pairs | Copia pares chave-valor entre `msg` e `metadata` |
| Deduplication | Faz buffer de mensagens do mesmo originador numa janela configurável |
| Delete Key-Value Pairs | Remove chaves específicas de `msg` ou `metadata` |
| JSON Path | Extrai parte dos dados via expressão JSONPath |
| Rename Keys | Renomeia chaves conforme mapeamento configurado |
| Script (Script Transformation) | Função TBEL/JS — ver seção dedicada abaixo |
| Split Array Msg | Divide um array JSON em mensagens individuais |
| To Email | Monta o objeto de e-mail a partir de `msg`/`metadata` |
| Duplicate to Group **(PE/Cloud)** | Duplica a mensagem para entidades de um grupo |
| Duplicate to Group by Name **(PE/Cloud)** | Duplica para grupo resolvido dinamicamente por nome |
| Duplicate to Related **(PE/Cloud)** | Duplica para entidades relacionadas ao originador |

### Script Transformation — contrato (confirmado na doc oficial)

Variáveis disponíveis: `msg` (JSON parseado), `metadata` (map string→string),
`msgType` (string). Retorno esperado — um objeto ou array de objetos (fan-out):

```javascript
return { msg: msg, metadata: metadata, msgType: msgType };

// fan-out (1 mensagem de entrada → N de saída)
return [
  { msg: msg1, metadata: metadata1, msgType: 'TYPE_A' },
  { msg: msg2, metadata: metadata2, msgType: 'TYPE_B' }
];
```

Saídas: `Success` (transformação ok) / `Failure` (erro no script).

Exemplo real (reformatar payload de API externa para telemetria ThingsBoard):

```tbel
var parsedTelemetry = [];
foreach(observation: msg.observations) {
  var values = {};
  values.put(msg.metric, observation.value);
  parsedTelemetry.add({
    ts: new Date(observation.time).getTime(),
    values: values
  });
}
return {
  msg: parsedTelemetry,
  metadata: metadata,
  msgType: "POST_TELEMETRY_REQUEST"
};
```

### Script Filter — contrato (confirmado na doc oficial)

Mesmas variáveis (`msg`, `metadata`, `msgType`). Deve retornar `boolean`:

```tbel
if (msgType != 'POST_TELEMETRY_REQUEST') {
    return false;
}
foreach (key: msg.keySet()) {
    var thresholdKey = key + 'Threshold';
    if (metadata.containsKey(thresholdKey)) {
        var value = msg[key];
        var threshold = parseDouble(metadata[thresholdKey]);
        if (value > threshold) {
            return true;
        }
    }
}
return false;
```

`true` → `True`, `false` → `False`, exceção/retorno não-boolean → `Failure`.

## Action — efeito colateral (persistência, alarme, controle)

| Node | Descrição |
|---|---|
| Calculated fields | Processa calculated fields sem persistir os dados originais |
| Clear alarm | Limpa alarmes ativos do originador |
| Copy to view | Replica mudanças de atributos para "views" de entidade |
| Create alarm | Cria ou atualiza alarme |
| Create relation | Cria relação entre entidades |
| Delay | Retém mensagem antes de encaminhar (**deprecado**) |
| Delete attributes | Remove atributos do originador |
| Delete relation | Remove relação entre entidades |
| Device profile | Avalia regras de alarme de device profile (**deprecado**) |
| Device state | Altera estado de conectividade do device |
| Generator | Gera mensagens em intervalos configuráveis (útil para simular/testar) |
| GPS geofencing events | Monitora coordenadas contra geofences e dispara eventos de entrada/saída |
| Log | Grava entrada de log customizada (debug) |
| Math function | Executa operação matemática sobre valores da mensagem |
| Message count | Conta mensagens em um intervalo |
| Push to edge | Encaminha para instância ThingsBoard Edge |
| REST call reply | Envia resposta HTTP (para RPC/Webhook síncronos) |
| RPC call reply | Responde a uma requisição RPC vinda do device |
| RPC call request | Envia comando RPC para o device |
| Save attributes | Persiste dados como atributos |
| Save time series | Persiste dados como telemetria |
| Assign to customer | Atribui entidade a um customer |
| Unassign from customer | Remove atribuição de customer |
| Push to cloud | Encaminha para o cloud (**Edge only**) |
| Add to group **(PE/Cloud)** | Adiciona entidade a um grupo |
| Change owner **(PE/Cloud)** | Muda o proprietário (owner) da entidade |
| Generate dashboard report **(PE/Cloud)** | Gera screenshot/relatório de um dashboard |
| Generate report **(PE/Cloud)** | Gera relatório |
| Integration downlink **(PE/Cloud)** | Envia downlink através de uma integração |
| Remove from group **(PE/Cloud)** | Remove entidade de um grupo |
| Save to custom table **(PE/Cloud)** | Persiste dados em tabela customizada (Cassandra) |

## External — publica dados fora do ThingsBoard

| Node | Descrição |
|---|---|
| AI Request | Envia requisição a provedor de LLM configurado; substitui dados da mensagem pela resposta |
| AWS Lambda | Invoca função Lambda de forma síncrona com `msg` como payload |
| AWS SNS | Publica em tópico SNS |
| AWS SQS | Envia para fila SQS (Standard ou FIFO) |
| Azure IoT Hub | Publica via MQTT com SAS token ou certificado X.509 |
| GCP Pub/Sub | Publica em tópico Google Cloud Pub/Sub |
| Kafka | Publica em tópico Kafka |
| MQTT | Publica em broker MQTT externo |
| RabbitMQ | Publica em exchange RabbitMQ |
| REST API Call | Faz requisição HTTP a endpoint externo (com resposta) |
| Send Email | Envia e-mail via SMTP |
| Send Notification | Envia via Notification Center do ThingsBoard |
| Send SMS | Envia SMS (AWS SNS, Twilio ou SMPP) |
| Send to Slack | Publica em canal/DM do Slack |
| Twilio SMS **(PE/Cloud)** | SMS via Twilio com credenciais no próprio node |
| Twilio Voice **(PE/Cloud)** | Chamada de voz text-to-speech via Twilio |

## Flow — controla o caminho da mensagem entre chains/filas

| Node | Descrição |
|---|---|
| Acknowledge | Confirma (ack) a mensagem na fila e encaminha para os próximos nodes |
| Checkpoint | Transfere a mensagem para uma fila específica (processamento separado/sequencial) |
| Rule Chain | Encaminha a mensagem para outro rule chain configurado (composição/reuso) |
| Output | Retorna o resultado de um sub-chain para o Rule Chain node que o chamou |

Padrão de composição: um chain raiz despacha por tipo de device via "Rule Chain" node
para sub-chains especializados; cada sub-chain processa e devolve via "Output" node.

## Analytics — agregações e estatísticas sobre streams

| Node | Descrição |
|---|---|
| aggregate latest | Agrega periodicamente atributos/última telemetria de entidades filhas para um conjunto de entidades pai |
| aggregate stream | Calcula MIN/MAX/SUM/AVG/COUNT/UNIQUE sobre um stream de entrada |
| alarms count | Conta alarmes ao receber mensagem de novo alarme |
| alarms count (deprecated) | Versão anterior do node acima (deprecado) |

## Boas práticas ao escrever/editar rule chain JSON

- O label da conexão (`connections[].type`) precisa bater exatamente com a saída que
  aquele node emite (`Success`/`Failure`, `True`/`False`, ou nome customizado de um
  Switch) — conferir a doc do node específico antes de assumir o label.
- TBEL é a linguagem recomendada sobre JavaScript puro nos Script nodes por rodar em
  sandbox mais restrito e com melhor performance (ver `references/tbel.md`).
