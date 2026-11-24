# Defesa de Arquitetura de Software — Pull Request 
 
## Autor do PR 
- **Nome do Estudante:** TIAGO ESPINDOLA 
- **Artefato(s) de Responsabilidade:** DIAGRAMA DE SEQUÊNCIA  
 
## O que este PR introduz na Documentação Viva?

Este PR adiciona um **Diagrama de Sequência** focado na resiliência do fluxo de acesso biométrico no campus, na automação energética e no envio de notificações, incluindo dois cenários de falha simulados (`Chaos 1` e `Chaos 2`).

- [ ] **Visão Funcional / Estrutural:** Não há Diagrama de Casos de Uso nem Diagrama de Classes/Interfaces neste artefato.
- [x] **Visão Comportamental:** Diagrama de Sequência com fluxo principal, ramificações (`alt/else`), repetição (`loop`) e cenários de caos.
- [ ] **Visão Arquitetural Macro:** Não há Modelo C4 (L1, L2 ou L3) incluído.
- [ ] **Governança:** Não há Registros de Decisão Arquitetural (ADR) anexados.

**Resumo das alterações:** O diagrama detalha a interação entre o `Usuário do Campus`, `Leitor Biométrico`, `API Gateway e IoT Broker`, `Serviço de Orquestração de Acesso e Energia`, `Banco de Dados Principal`, `Cache de Estado e Timers`, `Message Broker`, `Worker de Automação e Alertas`, `Dispositivos IoT` e `Serviço Externo de E-mail`. Ele não indica o caminho exato dentro de `/docs/` onde o arquivo será salvo.

---

## Defesa das Decisões de Design (Justificativa Técnica)

### 1. Escolha de Abstrações e Desacoplamento
* **Porta de entrada e orquestração:** O `Leitor Biométrico` chama `POST /acessos/validar (credencial, ambienteId)` no `API Gateway e IoT Broker`, que delega para `validarAcesso(credencial, ambienteId)` no `Serviço de Orquestração de Acesso e Energia`.
* **Comunicação assíncrona via eventos:** Em vez de acoplar a liberação da porta ao acionamento de energia ou ao envio de e-mail, o serviço de orquestração publica `OccupancyEvent (AmbienteOcupado)` — e `AccessLogPendente` quando há fallback — no `Message Broker`.
* **Separação entre energia e notificação:** O `Worker de Automação e Alertas` consome `OccupancyEvent (AmbienteOcupado)`, dispara `MQTT Publish` para ligar iluminação, ar-condicionado e bancadas nos `Dispositivos IoT` e só depois faz `POST /send` para o `Serviço Externo de E-mail`.
* **O que não aparece no diagrama:** Não há definição de classes, interfaces ou visibilidades de métodos, já que o foco aqui é o comportamento e a troca de mensagens entre os componentes.

### 2. Dinâmica e Resiliência
* **Chaos 1 — Lentidão ou queda no `Banco de Dados Principal` (`> 3000ms` ou falha):** Se a consulta (`SELECT Usuario, Permissao WHERE ambienteId AND horarioAtual`) estoura o tempo limite ou falha, o `Serviço de Orquestração de Acesso e Energia` tenta ler `fallback:permissao:{usuarioId}:{ambienteId}` no `Cache de Estado e Timers`:
  * **Se achar permissão válida no cache:** atualiza o estado em cache (`Ambiente=Ocupado, SessaoDeUso=Ativa, presentes=1`), manda `AccessLogPendente` e `OccupancyEvent` para o `Message Broker`, libera a trava e responde `200 OK (Acesso Concedido - Modo Degradado)`.
  * **Se der cache miss:** segue uma postura *fail-closed*, mantendo a trava fechada e devolvendo `503 / 403 Acesso Negado (Invariante de Segurança)`.
* **Chaos 2 — `Serviço Externo de E-mail` fora do ar (`503 Service Unavailable`):** O `Worker de Automação e Alertas` envia a mensagem para `retry.notificacoes` com *backoff* exponencial (`5s`, `15s`, `45s`), tenta reenviar em um `loop` de até 3 vezes e, se mesmo assim falhar, joga para `dlq.notificacoes` — sem derrubar ou atrasar a energização do ambiente, que já ocorreu antes.

### 3. Trade-offs
O diagrama não traz uma seção explícita de prós e contras, mas deixa claro o direcionamento tomado: priorizar a disponibilidade física e a continuidade do uso do espaço (via cache de fallback e persistência eventual de `AccessLogPendente`) quando o banco falha, bloquear por segurança quando não há respaldo em cache (*fail-closed*) e isolar falhas de notificação em filas de *retry* e *DLQ* para que um problema de e-mail não impeça o acesso nem o controle de energia.

---

## Consistência Cross-Diagramas (Checklist de Autoverificação)

Como apenas o Diagrama de Sequência foi enviado, não dá para validar os outros modelos ainda:

- [ ] **Métodos nas interfaces do Diagrama de Classes:** Aparecem operações como `validarAcesso(credencial, ambienteId)`, `POST /acessos/validar`, `SELECT Usuario, Permissao WHERE ambienteId AND horarioAtual`, `INSERT AccessLog`, `GET fallback:permissao:{usuarioId}:{ambienteId}`, `MQTT Publish` e `POST /send`, mas não há Diagrama de Classes anexado para conferir os nomes.
- [ ] **Nomenclatura do Modelo C4 L2/L3:** Os participantes e filas citados (`Usuário do Campus`, `Leitor Biométrico`, `API Gateway e IoT Broker`, `Serviço de Orquestração de Acesso e Energia`, `Banco de Dados Principal`, `Cache de Estado e Timers`, `Message Broker`, `retry.notificacoes`, `dlq.notificacoes`, `Worker de Automação e Alertas`, `Dispositivos IoT` e `Serviço Externo de E-mail`) não têm um modelo C4 correspondente aqui para comparação.
- [ ] **Transições de estado e gatilhos:** Há referências a estados e retornos como `Ambiente=Ocupado`, `SessaoDeUso=Ativa`, `presentes=1`, `AmbienteOcupado`, `Porta Liberada`, `Mantém Trava Bloqueada` e `ACK Atuadores Energizados`, mas não há Máquina de Estados ou Diagrama de Atividades para cruzar essas transições.
 
## Validação da Sintaxe Diagrams-as-Code / Models-as-Code 
- **Ferramenta de Compilação/Renderização Utilizada:** MERMAID
- **Status da Compilação:** Código limpo e sem erros de sintaxe ao renderizar. a
