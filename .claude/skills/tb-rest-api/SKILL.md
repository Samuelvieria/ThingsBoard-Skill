---
name: tb-rest-api
description: Integração com a REST API do ThingsBoard (CE/PE) — autenticação JWT, provisionamento de devices, telemetria, atributos, alarmes e relações de entidades. Use ao escrever, revisar ou depurar scripts/clients (Python, Node.js, curl) que consomem a API do ThingsBoard, ou ao integrar sistemas externos com telemetria/comandos de devices.
---

# ThingsBoard REST API

## Antes de começar

Endpoints e schemas podem variar entre versões (CE vs PE, releases diferentes). Antes de
gerar código de integração para uma instância real:

1. Pergunte (ou verifique) a URL base da instância (`https://<host>`).
2. Sempre que possível, confirme o contrato exato na documentação Swagger/OpenAPI da
   própria instância, em `https://<host>/swagger-ui/`. É a fonte da verdade — mais
   confiável do que qualquer tabela memorizada, especialmente em PE (que tem endpoints
   extras: white-labeling, Edge, Mobile Application Center, Scheduler, etc.).
3. Se não houver acesso à instância, use as convenções abaixo como ponto de partida, mas
   avise o usuário para validar contra o Swagger antes de rodar em produção.

## Conceitos gerais

- **Duas "faces" da API**: a REST API de usuário (JWT, usada por UI/backoffice/integrações
  server-to-server) e a API de transporte de device (HTTP/MQTT/CoAP, autenticada com o
  *device access token*, usada pelo próprio device para publicar telemetria/atributos).
  Não confundir as duas — device nunca deveria carregar credenciais de usuário JWT.
- Todos os endpoints são via HTTPS, corpo em JSON (`Content-Type: application/json`).
- Timestamps são epoch **milissegundos**, não segundos.
- `entityType` é uma string maiúscula do enum (`DEVICE`, `ASSET`, `CUSTOMER`, `TENANT`,
  `USER`, `DASHBOARD`, ...) e aparece tanto em paths quanto em bodies de `EntityId`.
- Scopes de atributos: `CLIENT_SCOPE` (setado pelo device), `SHARED_SCOPE` (setado pelo
  backend, visível ao device), `SERVER_SCOPE` (interno, nunca vai ao device).
- Listagens paginadas usam `pageSize`, `page`, `sortProperty`, `sortOrder` (`ASC`/`DESC`)
  como query params e retornam `{ data: [...], totalPages, totalElements, hasNext }`.

## Autenticação (JWT)

```
POST /api/auth/login
{ "username": "...", "password": "..." }
→ { "token": "<JWT>", "refreshToken": "<JWT>" }
```

- Enviar o token nas chamadas seguintes no header `X-Authorization: Bearer <token>`.
- Access token expira (tipicamente minutos); usar `POST /api/auth/token` com o
  `refreshToken` para renovar sem novo login.
- Para automações/serviços, prefira gerar um usuário técnico com role restrita em vez de
  reusar credenciais de admin humano.

## Devices

- Criar: `POST /api/device` com body `{ "name": "...", "type": "...", "deviceProfileId": {...} }`
  (ou variante `/api/device?accessToken=...` para setar o token no ato).
- Obter credenciais (access token do device): `GET /api/device/{deviceId}/credentials`.
- Buscar por nome: `GET /api/tenant/devices?deviceName=...`.
- Deletar: `DELETE /api/device/{deviceId}`.
- device profiles controlam regras de transporte, rule chain padrão, alarmes e provisionamento.

## Telemetria

Consultar (via API de usuário, JWT):
```
GET /api/plugins/telemetry/{entityType}/{entityId}/values/timeseries
    ?keys=temperature,humidity&startTs=...&endTs=...&agg=NONE&interval=...&limit=100
```
```
GET /api/plugins/telemetry/{entityType}/{entityId}/values/timeseries?keys=temperature   (latest)
```

Enviar telemetria em nome de uma entidade via API de usuário (uso administrativo/backfill,
não é o caminho normal de um device):
```
POST /api/plugins/telemetry/{entityType}/{entityId}/timeseries/ANY?scope=ANY
{ "ts": 1700000000000, "values": { "temperature": 21.5 } }
```

Caminho normal de um **device real** publicando telemetria (usa o access token do device,
não JWT):
```
POST /api/v1/{DEVICE_ACCESS_TOKEN}/telemetry
{ "temperature": 21.5, "humidity": 55 }
```
(equivalentes existem via MQTT tópico `v1/devices/me/telemetry` e via CoAP).

## Atributos

```
GET  /api/plugins/telemetry/{entityType}/{entityId}/attributes/{scope}
POST /api/plugins/telemetry/{entityType}/{entityId}/attributes/{scope}
DELETE /api/plugins/telemetry/{entityType}/{entityId}/{scope}?keys=k1,k2
```

## Alarmes

```
POST /api/alarm
{ "originator": {"entityType":"DEVICE","id":"..."}, "type": "High Temperature",
  "severity": "CRITICAL", "status": "ACTIVE_UNACK" }

GET  /api/alarm/{alarmId}
POST /api/alarm/{alarmId}/ack
POST /api/alarm/{alarmId}/clear
GET  /api/alarm/{entityType}/{entityId}?searchStatus=ACTIVE&pageSize=...&page=...
```

Normalmente alarmes são criados/limpos automaticamente por um rule chain (node
"Create Alarm"/"Clear Alarm") em vez de chamada manual via API — ver skill
`tb-rule-engine` para esse fluxo.

## Relações de entidades

```
POST /api/relation
{ "from": {...}, "to": {...}, "type": "Contains", "typeGroup": "COMMON" }
GET  /api/relations/info?fromId=...&fromType=...
```

## Erros comuns

- 401/403: token expirado (renovar) ou usuário sem permissão na role/tenant/customer.
- 400 em telemetria: `ts` fora de ms, ou `values` com tipos não suportados (evitar
  aninhar objetos complexos como valor de timeseries).
- Confundir `entityId` (UUID) com `id` de outro recurso (ex: deviceId vs device
  credentialsId são objetos diferentes).
- Rate limiting: instâncias PE costumam ter limites de API por tenant — para bulk
  operations, dar preferência a batch/telemetria via MQTT em vez de laços de POST via REST.

## Referência completa

Ver [references/endpoints.md](references/endpoints.md) para uma tabela mais ampla de
endpoints agrupados por recurso.
