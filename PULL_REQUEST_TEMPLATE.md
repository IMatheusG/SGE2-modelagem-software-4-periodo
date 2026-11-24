# Defesa de Arquitetura de Software — Pull Request 
 
## Autor do PR 
- **Nome do Estudante:** Henrique Chittolina 
- **Artefato(s) de Responsabilidade:** C4 Nível 3 
 
## O que este PR introduz na Documentação Viva?

Este PR adiciona o modelo arquitetural de contêineres do sistema **Smart Campus UniSenai** em formato Mermaid (`C4Container`), acompanhado da análise de gargalos e decisões estruturais para a fronteira do sistema (`Smart Campus UniSenai`) e seus atores/sistemas externos.

- [ ] **Visão Funcional / Estrutural:** Diagrama de Casos de Uso ou Diagrama de Classes e Interfaces.
- [ ] **Visão Comportamental:** Diagrama de Sequência, Atividades (com Raias/Fork-Join) ou Máquina de Estados.
- [x] **Visão Arquitetural Macro:** Modelo C4 (L1, L2 ou L3 em Structurizr DSL, D2 ou PlantUML). *(Modelado em C4 Nível 2 via Mermaid `C4Container`)*
- [ ] **Governança:** Registros de Decisão Arquitetural (ADR).

---

## Defesa das Decisões de Design (Justificativa Técnica)

### 1. Escolha de Abstrações e Desacoplamento
A arquitetura separa o domínio em contêineres especializados com responsabilidades claras e protocolos adequados ao tipo de tráfego:
- **Ingestão e borda (`API Gateway e IoT Broker` — Node.js, REST/MQTT):** Recebe requisições HTTPS/TLS do `Leitor Biométrico`, consultas REST/JSON do `Portal Web de Gestão` e telemetria MQTT dos `Dispositivos IoT` (sensores de consumo/presença), encaminhando validações de acesso via REST/gRPC ao `Serviço de Orquestração de Acesso e Energia`.
- **Núcleo de acesso e regras de domínio (`Serviço de Orquestração de Acesso e Energia` — Node.js, TypeScript):** Valida identidade/permissão e logs no `Banco de Dados Principal` (PostgreSQL via SQL/TLS), consulta/atualiza o estado de ocupação no `Cache de Estado e Timers` (Redis via TCP) e publica eventos no `Message Broker` (RabbitMQ via AMQP).
- **Automação e integrações assíncronas (`Worker de Automação e Alertas` — Node.js, TypeScript):** Consome eventos do `Message Broker` (AMQP), gerencia o timer de 15 minutos do ar-condicionado no `Cache de Estado e Timers` (TCP), persiste dados de sessão/energia no `Banco de Dados Principal` (SQL/TLS), aciona atuadores (AC, iluminação e bancadas) nos `Dispositivos IoT` via MQTT, consulta o `Sistema de Grade Horária` (REST/HTTPS) e dispara alertas via `Serviço Externo de E-mail` (SMTP/REST).
- **Persistência e estado:** Separa dados transacionais, logs imutáveis de acesso, sessões e telemetria no `Banco de Dados Principal` (PostgreSQL) do estado ativo de ocupação, contagem de presentes e TTL de 15 minutos do AC no `Cache de Estado e Timers` (Redis).

### 2. Dinâmica e Resiliência
O diagrama evidencia mecanismos de isolamento de falhas e resiliência operacional:
- **Isolamento síncrono vs. assíncrono:** O acionamento físico de atuadores nos `Dispositivos IoT`, a consulta ao `Sistema de Grade Horária` e o envio de e-mails ficam no `Worker de Automação e Alertas` após consumo via `Message Broker` (RabbitMQ), evitando que latência ou indisponibilidade desses alvos bloqueiem a validação de acesso.
- **Estado distribuído e temporização:** O `Cache de Estado e Timers` (Redis) externaliza a contagem de ocupação e o TTL de 15 minutos do ar-condicionado em vez de mantê-los em memória local de uma única instância, reduzindo perda de estado em reinicializações e gargalos de escalabilidade horizontal.
- **Desacoplamento de telemetria e alertas:** A separação da ingestão MQTT no `API Gateway e IoT Broker` e o processamento assíncrono no `Worker de Automação e Alertas` reduzem a contenção sobre o fluxo de validação de acesso. *(Nota: detalhes finos de `alt/else`, políticas de DLQ/retentativas e guardas de estado não estão explicitados neste artefato C4 L2.)*

### 3. Trade-offs
- **Prós:** Menor latência no fluxo crítico de acesso biométrico, maior resiliência a falhas em atuadores/notificações/grade horária, escalabilidade horizontal do estado de ocupação/timers via Redis e redução de contenção entre telemetria e validação de acesso.
- **Contras:** Maior complexidade operacional e de infraestrutura ao introduzir mensageria (`Message Broker` / RabbitMQ) e cache dedicado (`Cache de Estado e Timers` / Redis), além de consistência eventual entre o registro de acesso/evento e as ações assíncronas de automação e alertas.

---

## Consistência Cross-Diagramas (Checklist de Autoverificação)

Como este PR entrega apenas o **C4 Nível 2 (Contêineres)** e ainda não contém Diagrama de Classes, Diagrama de Sequência ou Máquina de Estados no escopo fornecido:

- [ ] Os nomes de métodos chamados no Diagrama de Sequência existem nas interfaces do Diagrama de Classes. *(Pendente / fora do escopo deste PR — não há Diagrama de Sequência nem Diagrama de Classes anexados.)*
- [x] Os contêineres e componentes citados correspondem exatamente à nomenclatura do Modelo C4 L2/L3. *(Validado contra o `C4Container`: `Portal Web de Gestão`, `API Gateway e IoT Broker`, `Serviço de Orquestração de Acesso e Energia`, `Worker de Automação e Alertas`, `Banco de Dados Principal`, `Cache de Estado e Timers`, `Message Broker`, além dos externos `Leitor Biométrico`, `Dispositivos IoT`, `Sistema de Grade Horária` e `Serviço Externo de E-mail`.)*
- [ ] As transições de estado
 
## Validação da Sintaxe Diagrams-as-Code / Models-as-Code 
- **Ferramenta de Compilação/Renderização Utilizada:** Mermaid Live 
- **Status da Compilação:** Código limpo e sem erros de sintaxe ao renderizar. 
