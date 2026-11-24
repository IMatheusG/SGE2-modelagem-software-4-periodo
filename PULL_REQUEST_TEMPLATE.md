# Defesa de Arquitetura de Software — Pull Request 
 
## Autor do PR 
- **Nome do Estudante:** Nicolas de Freitas Machado
- **Artefato(s) de Responsabilidade:** ADR 0002 
 
## O que este PR introduz na Documentação Viva?

Adiciona à pasta `/docs/` o registro **ADR 0002**, formalizando a adoção do **Node.js** como runtime dos serviços/workers e do **Docker** para conteinerização e padronização da infraestrutura do **Smart Campus UniSenai** sobre RabbitMQ e Redis.

- [ ] **Visão Funcional / Estrutural:** Diagrama de Casos de Uso ou Diagrama de Classes e Interfaces.
- [ ] **Visão Comportamental:** Diagrama de Sequência, Atividades (com Raias/Fork-Join) ou Máquina de Estados.
- [ ] **Visão Arquitetural Macro:** Modelo C4 (L1, L2 ou L3 em Structurizr DSL, D2 ou PlantUML).
- [x] **Governança:** Registros de Decisão Arquitetural (ADR).

## Defesa das Decisões de Design (Justificativa Técnica)

1. **Escolha de Abstrações e Desacoplamento:** Isola o fluxo biométrico do acoplamento síncrono a IoT, banco e e-mail por meio de mensageria assíncrona (RabbitMQ), estado/timers externalizados (Redis), runtime não bloqueante (Node.js) e processos conteinerizados *stateless* (Docker).
2. **Dinâmica e Resiliência:** Aplica *timeouts* (> 3s), *Circuit Breaker* e *fallback* no Redis quando o PostgreSQL falha, além de *backoff* exponencial e *Dead Letter Queue* (`dlq.notificacoes`) nos *workers* Node.js; o Docker viabiliza simular falhas de dependências e escalar réplicas.
3. **Trade-offs:**
   - **Prós:** baixa latência na porta, tolerância a falhas, escalabilidade horizontal e ambientes reprodutíveis.
   - **Contras:** maior complexidade operacional (containers, *healthchecks*, observabilidade), consistência eventual, necessidade de idempotência nos consumidores e atenção para não bloquear o *event loop* do Node.js com tarefas *CPU-bound*.

## Consistência Cross-Diagramas (Checklist de Autoverificação)

- [x] Os nomes de métodos chamados no Diagrama de Sequência existem nas interfaces do Diagrama de Classes.
- [x] Os contêineres e componentes citados correspondem exatamente à nomenclatura do Modelo C4 L2/L3.
- [x] As transições de estado refletem os gatilhos disparados pelas atividades da aplicação.

## Validação da Sintaxe Diagrams-as-Code / Models-as-Code 
- **Ferramenta de Compilação/Renderização Utilizada:** Não aplicável.
- **Status da Compilação:** Por se tratar de um decisão técnica, não possui código, apenas decisões arquiteturais.
