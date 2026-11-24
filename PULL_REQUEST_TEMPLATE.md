# Defesa de Arquitetura de Software — Pull Request

## Autor do PR
- **Nome do Estudante:** Matheus Gabriel Souza Thimoteo
- **Artefato(s) de Responsabilidade:** Diagrama de Classes e Componentes (`docs/architecture/diagrama-classes-componentes.mmd`)

## O que este PR introduz na Documentação Viva?
Adiciona à pasta `/docs/architecture/` a modelagem estrutural dos serviços centrais do **Smart Campus UniSenai**, aplicando Inversão de Dependência (DIP) e Injeção de Dependência para separar as regras de negócio de acesso e energia das tecnologias de infraestrutura:
- [x] **Visão Funcional / Estrutural:** Diagrama de Casos de Uso ou Diagrama de Classes e Interfaces.
- [ ] **Visão Comportamental:** Diagrama de Sequência, Atividades (com Raias/Fork-Join) ou Máquina de Estados.
- [ ] **Visão Arquitetural Macro:** Modelo C4 (L1, L2 ou L3 em Structurizr DSL, D2 ou PlantUML).
- [ ] **Governança:** Registros de Decisão Arquitetural (ADR).

## Defesa das Decisões de Design (Justificativa Técnica)

1. **Escolha de Abstrações e Desacoplamento:**
   Os serviços centrais (`ServicoOrquestracaoAcessoEnergia` e `WorkerAutomacaoAlertas`) não dependem diretamente de banco de dados, broker ou protocolo IoT, mas sim de sete interfaces (`IPermissaoRepository`, `IAcessoRepository`, `IEstadoOcupacaoCache`, `IEventoPublisher`, `IDispositivoAtuadorGateway`, `IGradeHorariaProvider` e `INotificacaoService`) injetadas via construtor. Todas as dependências foram definidas como privadas (`-`) para garantir o encapsulamento, deixando públicos (`+`) apenas os métodos de caso de uso. Assim, se precisarmos trocar PostgreSQL, Redis, RabbitMQ ou MQTT no futuro, a regra de negócio permanece intacta.

2. **Dinâmica e Resiliência:**
   Mesmo sendo uma visão estrutural, o diagrama já prepara os contratos para tolerância a falhas:
   - A interface `IEstadoOcupacaoCache` prevê o método `buscarPermissaoFallback()` para manter a liberação de acesso mesmo se o banco relacional sofrer *timeout*, além de `agendarDesligamentoAC()` para controlar no Redis a tolerância de 15 minutos do ar-condicionado sem depender de memória local.
   - A interface `IEventoPublisher` desacopla a validação síncrona da porta do acionamento assíncrono dos atuadores (`publicarEventoOcupacao`) e oferece `enfileirarRetentativa()` para reprocessar falhas em serviços externos como o envio de e-mails.

3. **Trade-offs:**
   - **Prós:** Facilita muito os testes unitários (basta criar *mocks* das interfaces), evita acoplamento com hardware/infraestrutura e permite evoluir os adaptadores de forma independente.
   - **Contras:** Aumenta a quantidade de arquivos e camadas de indireção no código (7 interfaces e suas respectivas implementações concretas), exigindo mais organização inicial da equipe.

## Consistência Cross-Diagramas (Checklist de Autoverificação)
- [x] Os nomes de métodos chamados no Diagrama de Sequência existem nas interfaces do Diagrama de Classes (`validarAcesso`, `buscarPermissaoVigente`, `registrarAcesso`, `buscarPermissaoFallback`, `publicarEventoOcupacao`, `enfileirarRetentativa` e ações dos atuadores).
- [x] Os contêineres e componentes citados correspondem exatamente à nomenclatura do Modelo C4 L2/L3 (`ServicoOrquestracaoAcessoEnergia`, `WorkerAutomacaoAlertas` e os adaptadores de PostgreSQL, Redis, RabbitMQ, MQTT e SMTP).
- [x] As transições de estado refletem os gatilhos disparados pelas atividades da aplicação (entrada/saída atualizando ocupação e disparando eventos de energia e desperdício).
