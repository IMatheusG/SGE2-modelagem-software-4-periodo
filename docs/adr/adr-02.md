# PR: Adoção de Node.js e Docker na Arquitetura de Mensageria (RabbitMQ) e Cache Distribuído (Redis) — Smart Campus UniSenai

## Resumo da Mudança

Esta Pull Request apresenta a versão da **ADR 0002** incorporando **Node.js** como runtime padrão para os serviços e *workers* orientados a eventos e **Docker** (junto a ambientes conteinerizados) para padronização, isolamento e escalabilidade da infraestrutura do **Smart Campus UniSenai**.

O objetivo é manter o desacoplamento entre o **controle de acesso biométrico** e a **automação energética** (RabbitMQ + Redis), garantindo baixa latência na catraca/porta, tolerância a falhas e reprodutibilidade operacional desde o desenvolvimento local até a produção.

---

## Contexto Técnico

No cenário do **Smart Campus UniSenai**, cada leitura biométrica em um `Ambiente` aciona uma `Sessão de Uso` (`Modo Operacional`), dispara o controle de iluminação e bancadas, gerencia a janela de 15 minutos do ar-condicionado (`Perfil Energético`), executa pré-condicionamento (`Política de Agendamento`) e identifica desperdícios (`Alerta de Desperdício`).

Para sustentar esse fluxo sem gargalos síncronos ou dependência de estado local:
- **Node.js** oferece *event loop* não bloqueante e alta concorrência de I/O, adequado para receber eventos de leitores biométricos, publicar/consumir mensagens AMQP no **RabbitMQ**, executar operações atômicas no **Redis** e lidar com integrações de rede (IoT, PostgreSQL e serviço externo de e-mail) sem travar threads em esperas de I/O.
- **Docker** resolve a heterogeneidade de ambientes e facilita o provisionamento reprodutível do **Serviço de Orquestração de Acesso e Energia**, do **Worker de Automação e Alertas**, do **RabbitMQ**, do **Redis** e do **PostgreSQL**, permitindo escalar réplicas de *workers* e simular falhas de rede/dependências em testes de resiliência (*Chaos Engineering*).

---

## Decisão (Atualizada com Node.js e Docker)

1. **Runtime Node.js para Orquestração e Workers:**
   - Implementar o `Serviço de Orquestração de Acesso e Energia` e o `Worker de Automação e Alertas` em **Node.js**, aproveitando I/O assíncrono para publicação/consumo no **RabbitMQ** (`OccupancyEvent`, `WasteAlert`, `AccessLogPendente`), leitura/escrita no **Redis** e controle de *timeouts* / *Circuit Breaker* frente ao **PostgreSQL** (> 3s).
   - Centralizar no `Worker de Automação e Alertas` o tratamento de retentativas com *backoff* exponencial e *Dead Letter Queue* (`dlq.notificacoes`) para falhas no `Serviço Externo de E-mail`.

2. **Conteinerização com Docker:**
   - Empacotar cada serviço Node.js em imagens **Docker** leves e determinísticas (ex.: *multi-stage builds* e usuário não-root), com *healthchecks* e variáveis de ambiente explícitas.
   - Orquestrar localmente e em ambientes de teste/produção os contêineres de **Node.js**, **RabbitMQ**, **Redis** e **PostgreSQL**, viabilizando subida rápida do ecossistema e testes controlados de indisponibilidade de dependências.

---

## Argumentação: Por que Node.js + Docker neste desenho?

| Critério Arquitetônico | Contribuição do Node.js | Contribuição do Docker |
| :--- | :--- | :--- |
| **Latência em Fluxos de I/O** | Modelo orientado a eventos e I/O assíncrono atende picos de leituras biométricas e chamadas de rede sem bloqueio por *thread*. | Permite iniciar serviços e dependências com configuração padronizada e *healthchecks* previsíveis. |
| **Resiliência e Falhas Externas** | Facilita *timeouts* curtos, *Circuit Breaker*, *backoff* exponencial e *fallback* no Redis quando o PostgreSQL excede 3s ou o e-mail falha. | Permite isolar ou derrubar contêineres (ex.: parar PostgreSQL ou e-mail mock) para validar *fallback* e DLQ em testes. |
| **Escalabilidade Horizontal** | Processos *stateless* (com estado em Redis e filas no RabbitMQ) escalam em múltiplas instâncias de orquestrador e *workers*. | Facilita subir múltiplas réplicas do mesmo contêiner de *worker* consumindo filas do RabbitMQ em paralelo. |
| **Padronização e DX** | Ecossistema único em JavaScript/TypeScript entre orquestração, *workers* e clientes AMQP/Redis. | Elimina "funciona na minha máquina" ao alinhar runtime, dependências e *brokers* entre devs, CI e campus. |

---

## Consequências Positivas

- **Baixa Latência no Acesso Físico:** O *event loop* do Node.js processa a validação e publica/consulta estados de forma não bloqueante, mantendo a resposta ao `Leitor Biométrico` rápida sem aguardar atuadores IoT ou envio de e-mail.
- **Resiliência e Tolerância a Falhas (Chaos Engineering):** Combinar *Circuit Breaker* e *fallback* em Redis no Node.js com testes em contêineres Docker torna verificável o comportamento durante *timeouts* (> 3s) no PostgreSQL ou quedas no serviço de e-mail.
- **Eliminação de SPOF e Escalabilidade Horizontal:** Estado de ocupação e temporizadores de 15 minutos ficam no Redis; múltiplas instâncias em contêineres Docker do orquestrador e dos *workers* operam de forma distribuída sobre o RabbitMQ.
- **Portabilidade e Reprodutibilidade:** Imagens Docker versionadas garantem que alterações de código e configuração de filas/cache sejam testadas e promovidas com consistência.

---

## Consequências Negativas (Trade-offs)

- **Maior Complexidade Operacional e de Infraestrutura:** Além de RabbitMQ, Redis e PostgreSQL, a equipe passa a gerenciar *pipelines* de imagem Docker, *registries*, orquestração de contêineres, *healthchecks* e observabilidade de processos Node.js.
- **Cuidados com CPU-Bound e *Event Loop* no Node.js:** Operações síncronas pesadas ou serializações grandes podem bloquear o *event loop* e elevar latências; tarefas pesadas devem ser evitadas ou isoladas.
- **Consistência Eventual nos Efeitos Colaterais:** Permanece a janela assíncrona entre liberação da porta, acionamento IoT e persistência eventual de logs/notificações.
- **Idempotência Obrigatória nos Consumidores:** Reentregas de mensagens no RabbitMQ (ou reinícios de contêineres durante processamento) exigem chaves de idempotência no `Worker de Automação e Alertas` para não duplicar comandos aos atuadores.

---

## Checklist de Revisão desta PR

- [ ] Validar se os *timeouts* de conexão/query no Node.js para PostgreSQL e Redis estão alinhados ao limite de *fallback* (> 3s).
- [ ] Conferir se os *workers* em Node.js aplicam chaves de idempotência antes de acionar iluminação, bancadas, ar-condicionado ou enviar alertas.
- [ ] Revisar `Dockerfile` e manifestos de contêineres quanto a *healthchecks*, *graceful shutdown* (fechamento limpo de conexões AMQP/Redis ao receber `SIGTERM`) e variáveis de ambiente.
- [ ] Garantir cobertura de testes de integração/resiliência simulando indisponibilidade do PostgreSQL e falha no envio de e-mail (`dlq.notificacoes`).