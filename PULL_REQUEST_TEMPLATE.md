# Defesa de Arquitetura de Software — Pull Request

## Autor do PR
- **Nome do Estudante:** TIAGO SCHMITZ
- **Artefato(s) de Responsabilidade:** ADR 0001

## O que este PR introduz na Documentação Viva?

Adiciona à pasta `/docs/` o registro **ADR 0001**, que formaliza a adoção do **RabbitMQ** (mensageria assíncrona via AMQP) e do **Redis** (cache distribuído de estado e *timers*) para desacoplar o controle de acesso biométrico da automação energética no **Smart Campus UniSenai**.

* [ ] **Visão Funcional / Estrutural:** Diagrama de Casos de Uso ou Diagrama de Classes e Interfaces.
* [ ] **Visão Comportamental:** Diagrama de Sequência, Atividades (com Raias/Fork-Join) ou Máquina de Estados.
* [ ] **Visão Arquitetural Macro:** Modelo C4 (L1, L2 ou L3 em Structurizr DSL, D2 ou PlantUML).
* [x] **Governança:** Registros de Decisão Arquitetural (ADR).

---

## Defesa das Decisões de Design (Justificativa Técnica)

1. **Escolha de Abstrações e Desacoplamento:** O desenho separa o `Serviço de Orquestração de Acesso e Energia` do `Worker de Automação e Alertas` por meio de eventos de domínio (`OccupancyEvent`, `WasteAlert`) e registros pendentes de auditoria (`AccessLogPendente`) trafegados via AMQP no RabbitMQ, além de externalizar estados (`Vazio`, `Ocupado`, `Pré-condicionado`), contagem atômica de ocupação, janela de 15 minutos do ar-condicionado e *fallback* de permissões críticas no Redis. Isso evita que a resposta ao `Leitor Biométrico` fique acoplada à latência ou disponibilidade de atuadores IoT, consultas pesadas ao `PostgreSQL` ou ao `Serviço Externo de E-mail`, e elimina estado/temporizadores em memória local (SPOF).


2. **Dinâmica e Resiliência:**
* **Timeouts, Circuit Breaker e Fallback:** *Timeouts* superiores a 3 segundos no `PostgreSQL` são tratados com *Circuit Breaker*, cache de *fallback* de permissões críticas no Redis e publicação assíncrona de `AccessLogPendente` no RabbitMQ.


* **Mensageria Assíncrona, Retentativas e DLQ:** Falhas em integrações externas, como o `Serviço Externo de E-mail`, são tratadas por filas de retentativa com *backoff* exponencial e *Dead Letter Queue* (`dlq.notificacoes`), sem interromper a automação energética.


* **Idempotência:** O `Worker de Automação e Alertas` utiliza chaves de idempotência para evitar comandos repetidos aos atuadores em caso de reentrega de mensagens pelo RabbitMQ.




3. **Trade-offs:**
* **Prós:** Baixa latência (em milissegundos) na resposta ao `Leitor Biométrico`; resiliência e tolerância a falhas (*Chaos Engineering*) frente a *timeouts* (> 3s) no PostgreSQL ou falhas de e-mail; eliminação de SPOF e escalabilidade horizontal de orquestradores e *workers*; e extensibilidade orientada a eventos para novos consumidores (como painéis analíticos e auditoria avançada).


* **Contras:** Maior complexidade operacional e de infraestrutura para provisionar, monitorar e manter RabbitMQ e Redis além do PostgreSQL; consistência eventual de alguns milissegundos entre liberação da porta, acionamento IoT e gravação de logs/notificações; e exigência de chaves de idempotência nos consumidores.





---

## Consistência Cross-Diagramas (Checklist de Autoverificação)

Nota: A ADR 0001 não apresenta diagramas de sequência, classes ou C4; os itens abaixo refletem apenas os elementos e conceitos explicitamente nomeados no texto da ADR.

* [ ] Os nomes de métodos chamados no Diagrama de Sequência existem nas interfaces do Diagrama de Classes. *(Não se aplica / não consta na ADR 0001)*


* [ ] Os contêineres e componentes citados correspondem exatamente à nomenclatura do Modelo C4 L2/L3. *(A ADR referencia `Serviço de Orquestração de Acesso e Energia`, `Worker de Automação e Alertas`, `Leitor Biométrico`, `RabbitMQ`, `Redis`, `PostgreSQL`, `Serviço Externo de E-mail` e `dlq.notificacoes`, sem especificar modelo C4)*


* [ ] As transições de estado refletem os gatilhos disparados pelas atividades da aplicação. *(A ADR cita os estados de ocupação `Vazio`, `Ocupado` e `Pré-condicionado` do `Ambiente`, `Sessão de Uso`, janela de 15 minutos do ar-condicionado e eventos `OccupancyEvent` e `WasteAlert`, sem detalhar máquina de estados ou atividades)*

## Validação da Sintaxe Diagrams-as-Code / Models-as-Code
- **Ferramenta de Compilação/Renderização Utilizada:** Não aplicável por se tratar de uma decisão técnica
- **Status da Compilação:** Por se tratar de uma ADR não possui código

