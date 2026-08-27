---
name: tb-deploy-admin
description: Instalação, configuração e administração do ThingsBoard PE/CE self-hosted — Docker Compose (monolito e microsserviços), banco de dados (PostgreSQL/Cassandra/TimescaleDB), fila (Kafka), licenciamento PE, upgrade de versão, backup/restore e troubleshooting. Use ao planejar instalação, upgrade, HA, backup ou depurar problemas de infraestrutura do ThingsBoard.
---

# ThingsBoard Deploy & Administração (self-hosted / PE)

## Antes de começar

Procedimentos de upgrade e nomes exatos de imagens/scripts mudam por versão. Antes de
propor passos concretos:

1. Confirme a versão atual instalada (`Help → About` na UI, ou tag da imagem Docker em uso).
2. Se for uma operação de upgrade, confirme a versão de destino e avise o usuário para
   checar as **release/upgrade notes oficiais** daquele intervalo de versões — em geral
   ThingsBoard recomenda upgrade sequencial (não pular versões major) e o script de
   upgrade de banco muda por versão.
3. **PE usa registry Docker privado** (requer login com credenciais associadas à licença),
   diferente do CE que fica no Docker Hub público — não assumir que `docker pull` de uma
   imagem PE funciona sem `docker login` prévio.

## Modos de instalação

| Modo | Quando usar |
|---|---|
| **Monolítico** (`tb` único container/processo) | Ambientes pequenos/médios, PoC, single-tenant, menor complexidade operacional |
| **Microsserviços** (`tb-core`, `tb-rule-engine`, `tb-transport-*`, `tb-web-ui` separados + Kafka/Zookeeper) | Alta escala, necessidade de escalar componentes independentemente, HA real |

Fila (`queue`) e cache são configuráveis independente do modo:

- Fila: `in-memory` (só monolito/dev — mensagens **se perdem** em restart), `kafka`
  (padrão em produção/microsserviços — durável, persiste entre restarts, obrigatório em
  cluster), `rabbitmq`, `aws-sqs`, `pubsub` (GCP), `service-bus` (Azure)
- Cache: `caffeine` (in-process, só single instance) ou `redis` (compartilhado entre
  instâncias — obrigatório se houver mais de um `tb-core`)

### Filas do Rule Engine (dentro do backend de queue escolhido)

Confirmado na doc oficial: as filas garantem entrega sob carga, controlam picos e
ordenam mensagens submetidas aos rule chains. Configuração só por administrador de
sistema (não por tenant admin).

- **Filas padrão**: `Main` (entrada padrão, estratégia Burst), `HighPriority` (alarmes/
  notificações, Burst com retry), `SequentialByOriginator` (garante ordem por entidade).
- Um rule node individual pode ser roteado para uma fila específica via o campo
  `queueName` na configuração do node (visto em produção como `null` = usa a fila padrão
  do rule chain) — útil para isolar processamento de alarme/notificação numa fila de
  alta prioridade sem competir com o volume de telemetria normal na fila `Main`.
- **Estratégia de submit** (como mensagens chegam ao rule chain): Burst (tudo de uma
  vez), Batch (lotes fixos aguardando ack), Sequential by originator/by tenant/global
  (ordem garantida, mais lento quanto mais global).
- **Estratégia de retry**: Skip all failures, Skip all failures and timeouts, Retry
  failed, Retry timeout, Retry failed and timeout, Retry all (reenvia o lote inteiro se
  1 mensagem falhar).
- **Cuidado**: a fila `HighPriority` reententa indefinidamente por padrão — um erro
  lógico persistente num node dessa fila pode travar o processamento de alarmes
  permanentemente até alguém corrigir a causa raiz; monitorar lag dessa fila
  especificamente.
- Partições maiores = mais paralelismo; ajustar junto com intervalo de polling e timeout
  de processamento por fila.

## Docker Compose — pontos de atenção

Serviços típicos: `postgres` (ou banco escolhido), `thingsboard` (monolito) ou
`tb-core`/`tb-rule-engine`/`tb-mqtt-transport`/`tb-http-transport`/`tb-coap-transport`
(microsserviços), `kafka`+`zookeeper` (se fila = kafka), `redis` (se cache/sessão
compartilhada).

Portas padrão a mapear:

| Porta | Protocolo |
|---|---|
| 8080 | HTTP (UI + REST API) |
| 1883 | MQTT |
| 5683/5684 | CoAP / CoAP+DTLS |
| 7070 | Edge RPC (se usar ThingsBoard Edge) |
| 9090 | gRPC transport (comunicação interna entre microsserviços) |

Variáveis de ambiente centrais (nomes indicativos, confirmar contra `.env`/docs da
versão exata):

- `DATABASE_TS_TYPE`: `sql` | `cassandra` | `timescale` (define onde fica timeseries)
- `SPRING_DATASOURCE_URL` / `_USERNAME` / `_PASSWORD`: conexão com o banco relacional
- `TB_QUEUE_TYPE`: tipo de fila (ver acima)
- `CACHE_TYPE`: `caffeine` | `redis`
- `TB_SERVICE_ID` / licença: em PE, variáveis específicas de ativação de licença
  (nome exato varia por versão — checar doc oficial de instalação PE correspondente)

Ver [references/docker-compose-example.yml](references/docker-compose-example.yml) para
um esqueleto ilustrativo (monolítico, Postgres).

## Licenciamento (PE)

- Instância PE precisa de chave de licença ativa; sem ela, funcionalidades PE ficam
  bloqueadas ou o serviço recusa subir, dependendo da versão.
- Ativação normalmente feita via UI (`Settings → License`) ou variável de ambiente na
  subida do container.
- Erros de "license invalid/expired" nos logs geralmente indicam: chave vencida, chave
  associada a outro `TB_SERVICE_ID`/host, ou relógio do host dessincronizado (NTP).

## Upgrade de versão

Fluxo geral (validar passo a passo contra a doc oficial da versão específica):

1. Backup completo do banco (ver seção Backup) e, se possível, snapshot dos volumes.
2. Parar o serviço ThingsBoard (manter banco no ar).
3. Rodar o script de upgrade de schema (`upgrade.sh`/`upgrade.bat` ou equivalente da
   imagem, tipicamente com `--fromVersion=X.Y.Z`).
4. Subir a nova versão da imagem/binário.
5. Validar: rule chains, dashboards e widgets continuam funcionando; checar logs por
   erros de migração.
6. Em microsserviços, atualizar todos os componentes (`tb-core`, `tb-rule-engine`,
   transports) para a mesma versão — não misturar versões entre componentes.

## Backup & Restore

- **Postgres**: `pg_dump`/`pg_restore` do banco `thingsboard` (metadados + timeseries se
  `DATABASE_TS_TYPE=sql`).
- **Cassandra**: snapshot via `nodetool snapshot` + cópia dos SSTables.
- **Complementar**: exportar rule chains e dashboards críticos como JSON (via UI/API) —
  mais rápido de restaurar seletivamente do que um restore completo de banco.
- Testar o restore periodicamente em ambiente separado — backup não testado não é backup.

## HA e escalabilidade

- Múltiplas instâncias `tb-core`/`tb-rule-engine` atrás de load balancer (para
  `tb-core`, sticky session ou WebSocket-aware LB para telemetria em tempo real na UI).
- Kafka como fila obrigatório para múltiplas instâncias de rule engine coordenarem
  particionamento de mensagens.
- Redis obrigatório para cache/sessão compartilhados entre instâncias de `tb-core`.
- Transports (`mqtt`, `http`, `coap`) escalam horizontalmente de forma mais simples,
  atrás de LB de rede (TCP para MQTT).

## Troubleshooting

- Logs: dentro do container em `/var/log/thingsboard` (ou `docker logs <container>` se
  não houver volume de log mapeado).
- Erros comuns:
  - Conexão recusada ao banco → checar ordem de subida (banco pronto antes do TB) e
    healthcheck no compose.
  - Fila com lag crescente → checar throughput de `tb-rule-engine`, possível gargalo em
    node externo (REST call, email) dentro de uma rule chain.
  - Licença inválida (PE) → ver seção Licenciamento.
  - UI carrega mas dashboards não atualizam em tempo real → checar WebSocket
    (`/api/ws`) não bloqueado por proxy/LB intermediário.

## Referência

Ver [references/docker-compose-example.yml](references/docker-compose-example.yml).
