# APS 2 — Dossiê de Arquitetura Viva

**Aluno(a):** Cauã Oliveira Dos Santos
**Grupo:** Star
**Projeto:** Marketplace B2B de Sobras Industriais
**Funcionalidade crítica:** Logística e Transporte Físico das Sobras — Despacho, Rastreio e Liberação da Custódia

## 1. Funcionalidade

Depois que a proposta é paga em custódia , a plataforma precisa conduzir o lote até o comprador sem perder o controle do dinheiro. O transporte físico é executado por uma **transportadora externa**. A plataforma orquestra o processo em três fases:

1. **Agendamento da coleta.** O vendedor informa a janela, o peso e o volume, e a plataforma solicita a coleta à transportadora, com até 3 tentativas. Se todas forem recusadas, a custódia é estornada ao comprador.
2. **Rastreio.** A transportadora envia checkpoints por webhook (`COLETADO`, `EM_TRANSITO`, `ENTREGUE`), e a plataforma atualiza a `Remessa`.
3. **Conferência e liberação.** O comprador confere a quantidade recebida. Se estiver conforme, o valor é liberado ao vendedor. Se houver divergência, a custódia é congelada e abre-se uma disputa.

A criticidade vem do risco financeiro (liberar valor sem entrega, ou reter valor sem destino) e da dependência de sistemas externos que podem falhar ou enviar eventos duplicados e fora de ordem.


## 2. Artefato 1 — Diagrama de Sequência

Arquivo: [`docs/diagrams/sequencia.mmd`](docs/diagrams/sequencia.mmd)

```mermaid
sequenceDiagram
    autonumber
    actor Vendedor as Empresa Vendedora
    actor Comprador as Empresa Compradora
    participant RemessaService as RemessaService<br/>«Service»
    participant Remessa as Remessa<br/>«Aggregate»
    participant TransportadoraAPI as TransportadoraAPI<br/>«Externo»
    participant EscrowAPI as EscrowAPI<br/>«Externo»
    participant EventBus as EventBus<br/>«RabbitMQ»
    participant NotificacaoWorker as NotificacaoWorker<br/>«Consumer»

    Note over Vendedor,RemessaService: FASE 1 - Agendamento da coleta<br/>Pré-condição (Sprint 2): proposta PAGA_EM_CUSTODIA e Remessa criada

    Vendedor->>RemessaService: agendarColeta(cmd: AgendamentoColetaDTO)
    RemessaService->>Remessa: iniciarAgendamento(janela: JanelaColetaDTO)

    alt [estado == AGUARDANDO_AGENDAMENTO and custodia.status == RETIDA]
        Remessa-->>RemessaService: RemessaDTO(remessaId, estado=AGUARDANDO_AGENDAMENTO, tentativas=0)

        loop até coleta confirmada ou 3 tentativas recusadas
            RemessaService->>TransportadoraAPI: solicitarColeta(req: ColetaRequestDTO)
            TransportadoraAPI-->>RemessaService: ColetaResponseDTO(status, codigoRastreio, janelaConfirmada)
            alt [status == CONFIRMADA]
                RemessaService->>Remessa: confirmarColeta(codigoRastreio, janelaConfirmada)
                Remessa-->>RemessaService: RemessaDTO(estado=COLETA_AGENDADA)
            else [status != CONFIRMADA]
                RemessaService->>Remessa: registrarFalhaAgendamento(motivo)
                Remessa-->>RemessaService: RemessaDTO(estado, tentativas)
            end
        end

        alt [estado == COLETA_AGENDADA]
            Note over RemessaService,EventBus: Publicação via Transactional Outbox (ADR 0001)
            RemessaService-)EventBus: publicar(evt: ColetaAgendadaEvent)
            par notificar comprador
                EventBus-)NotificacaoWorker: notificarComprador(evt)
                NotificacaoWorker-)Comprador: e-mail e push com código de rastreio
            and notificar vendedor
                EventBus-)NotificacaoWorker: notificarVendedor(evt)
                NotificacaoWorker-)Vendedor: e-mail e push com ordem de coleta
            end
        else [estado == ESTORNADA]
            RemessaService->>EscrowAPI: estornarCustodia(req: EstornoDTO)
            EscrowAPI-->>RemessaService: EscrowEstornoDTO(status=ESTORNADO)
            RemessaService-)EventBus: publicar(evt: RemessaEstornadaEvent)
            EventBus-)NotificacaoWorker: notificarComprador(evt)
        end

        RemessaService-->>Vendedor: RemessaDTO(remessaId, estado)
    else [estado != AGUARDANDO_AGENDAMENTO or custodia.status != RETIDA]
        Remessa--xRemessaService: EstadoInvalidoException
        RemessaService-->>Vendedor: ErroTransacaoDTO(codigo, mensagem)
    end

    Note over TransportadoraAPI,RemessaService: FASE 2 - Rastreio do transporte físico<br/>Webhooks validados por HMAC-SHA256 (ADR 0001)

    loop a cada checkpoint de rastreio até status == ENTREGUE
        TransportadoraAPI-)RemessaService: webhookRastreio(evt: CheckpointDTO)
        RemessaService->>Remessa: registrarCheckpoint(cp: CheckpointDTO)
        Remessa-->>RemessaService: RemessaDTO(estado, ultimaPosicao)
    end

    RemessaService-)EventBus: publicar(evt: EntregaRegistradaEvent)
    EventBus-)NotificacaoWorker: notificarComprador(evt)
    NotificacaoWorker-)Comprador: e-mail e push para conferir a entrega

    Note over Comprador,EscrowAPI: FASE 3 - Conferência e liberação da custódia<br/>Pré-condição: estado == AGUARDANDO_CONFERENCIA

    Comprador->>RemessaService: conferirEntrega(cmd: ConferenciaDTO)
    RemessaService->>Remessa: registrarConferencia(cmd: ConferenciaDTO)
    Remessa-->>RemessaService: ConferenciaResultadoDTO(resultado)

    alt [resultado == APROVADA]
        RemessaService->>EscrowAPI: liberarCustodia(req: LiberacaoDTO)
        EscrowAPI-->>RemessaService: EscrowLiberacaoDTO(status=LIBERADO)
        RemessaService->>Remessa: concluir(transacaoId)
        Remessa-->>RemessaService: RemessaDTO(estado=CONCLUIDA)
        RemessaService-)EventBus: publicar(evt: RemessaConcluidaEvent)
        EventBus-)NotificacaoWorker: notificarVendedor(evt)
    else [resultado == REPROVADA]
        RemessaService->>EscrowAPI: congelarCustodia(req: CongelamentoDTO)
        EscrowAPI-->>RemessaService: EscrowCongelamentoDTO(status=CONGELADO)
        RemessaService->>Remessa: abrirDisputa(motivo)
        Remessa-->>RemessaService: RemessaDTO(estado=EM_DISPUTA)
        RemessaService-)EventBus: publicar(evt: DisputaAbertaEvent)
    end

    RemessaService-->>Comprador: ConferenciaResultadoDTO(remessaId, estado)
```

## 3. Artefato 2 — Diagrama de Atividades com Raias

Arquivo: [`docs/diagrams/atividades.mmd`](docs/diagrams/atividades.mmd)

```mermaid
flowchart TB
    classDef ini fill:#111,stroke:#111,color:#fff
    classDef fim fill:#fff,stroke:#111,stroke-width:4px,color:#111
    classDef bar fill:#111,stroke:#111,color:#fff,font-weight:bold
    classDef dec fill:#fff8e1,stroke:#f9a825,color:#111

    subgraph R1["Raia 1 - Atores (Empresa Vendedora / Empresa Compradora)"]
        A1["Vendedor informa janela, peso e volume (agendarColeta)"]
        A2["Comprador recebe o código de rastreio"]
        A3["Comprador confere quantidade e qualidade (conferirEntrega)"]
    end

    subgraph R2["Raia 2 - Marketplace API (RemessaService / Remessa)"]
        INI(("Início")):::ini
        B1["Remessa.iniciarAgendamento(janela)"]
        D1{"Remessa apta ao agendamento?"}:::dec
        B2["TransportadoraAPI.solicitarColeta(req)"]
        D2{"Coleta confirmada?"}:::dec
        B3["Remessa.confirmarColeta() → COLETA_AGENDADA"]
        B4["Remessa.registrarFalhaAgendamento(motivo)"]
        D3{"Ainda restam tentativas?"}:::dec
        B5["Remessa → ESTORNADA"]
        B6["Publicar ColetaAgendadaEvent via Outbox"]
        B7["Remessa.registrarCheckpoint(cp)"]
        D4{"Checkpoint é de entrega?"}:::dec
        B8["Remessa → AGUARDANDO_CONFERENCIA e publicar EntregaRegistradaEvent"]
        B9["Remessa.registrarConferencia(cmd)"]
        D5{"Resultado da conferência?"}:::dec
        B10["Remessa.concluir() → CONCLUIDA"]
        B11["Remessa.abrirDisputa(motivo) → EM_DISPUTA"]
        FIM1(("Fim")):::fim
        FIM2(("Fim")):::fim
        FIM3(("Fim")):::fim
        FIM4(("Fim")):::fim
    end

    subgraph R3["Raia 3 - Serviço Externo (TransportadoraAPI)"]
        T1["Transportadora avalia e responde à solicitação de coleta"]
        T2["Transportadora coleta e transporta o lote e envia webhooks de rastreio"]
    end

    subgraph R4["Raia 4 - Serviço Externo (Gateway de Escrow)"]
        E1["EscrowAPI.estornarCustodia(req)"]
        E2["EscrowAPI.liberarCustodia(req)"]
        E3["EscrowAPI.congelarCustodia(req)"]
    end

    subgraph R5["Raia 5 - Mensageria e Consumidores (RabbitMQ / NotificacaoWorker)"]
        F1["══ FORK ══"]:::bar
        M1["NotificacaoWorker.notificarComprador(evt) com rastreio"]
        M2["NotificacaoWorker.notificarVendedor(evt) com ordem de coleta"]
        J1["══ JOIN ══"]:::bar
        M3["NotificacaoWorker.notificarComprador(evt) sobre o estorno"]
        M4["NotificacaoWorker.notificarComprador(evt) para conferir a entrega"]
        M5["NotificacaoWorker.notificarVendedor(evt) sobre valor liberado"]
    end

    INI --> A1 --> B1 --> D1
    D1 -->|"[estado == AGUARDANDO_AGENDAMENTO and custodia.status == RETIDA]"| B2
    D1 -->|"[estado != AGUARDANDO_AGENDAMENTO or custodia.status != RETIDA]"| FIM1

    B2 --> T1 --> D2
    D2 -->|"[status == CONFIRMADA]"| B3
    D2 -->|"[status != CONFIRMADA]"| B4
    B4 --> D3
    D3 -->|"[tentativas ≤ 2]"| B2
    D3 -->|"[tentativas ≥ 3]"| B5
    B5 --> E1 --> M3 --> FIM2

    B3 --> B6 --> F1
    F1 --> M1 --> A2 --> J1
    F1 --> M2 --> J1

    J1 --> T2 --> B7 --> D4
    D4 -->|"[status != ENTREGUE]"| T2
    D4 -->|"[status == ENTREGUE]"| B8

    B8 --> M4 --> A3 --> B9 --> D5
    D5 -->|"[resultado == APROVADA]"| E2 --> B10 --> M5 --> FIM3
    D5 -->|"[resultado == REPROVADA]"| E3 --> B11 --> FIM4
```

## 4. Artefato 3 — Diagrama de Transição de Estados

Arquivo: [`docs/diagrams/estados.mmd`](docs/diagrams/estados.mmd)

```mermaid
stateDiagram-v2
    [*] --> AguardandoAgendamento

    state AguardandoAgendamento
    AguardandoAgendamento : do / aguardarJanelaDeColeta()
    AguardandoAgendamento : exit / encerrarPendenciaDeAgendamento()
    AguardandoAgendamento --> AguardandoAgendamento : coletaRecusada [tentativas ≤ 2] / registrarFalhaAgendamento()
    AguardandoAgendamento --> Estornada : coletaRecusada [tentativas ≥ 3] / registrarFalhaAgendamento()
    AguardandoAgendamento --> ColetaAgendada : coletaConfirmada [transportadora.status == CONFIRMADA] / confirmarColeta()

    state ColetaAgendada
    ColetaAgendada : entry / publicarColetaAgendadaEvent()
    ColetaAgendada --> EmTransito : checkpointRecebido [status == COLETADO] / registrarCheckpoint()

    state EmTransito {
        [*] --> Coletado
        Coletado --> EmRota : checkpointRecebido [status == EM_TRANSITO] / registrarCheckpoint()
        EmRota : do / monitorarRastreio()
        EmRota --> EmRota : checkpointRecebido [status == EM_TRANSITO] / registrarCheckpoint()
    }

    EmTransito --> AguardandoConferencia : checkpointRecebido [status == ENTREGUE] / registrarCheckpoint()

    state AguardandoConferencia
    AguardandoConferencia : entry / publicarEntregaRegistradaEvent()
    AguardandoConferencia : exit / registrarResultadoDaConferencia()
    AguardandoConferencia --> Concluida : conferirEntrega [qtdRecebida == qtdProposta and custodia.status == RETIDA] / concluir()
    AguardandoConferencia --> EmDisputa : conferirEntrega [qtdRecebida != qtdProposta] / abrirDisputa()

    state EmDisputa
    EmDisputa : do / aguardarLaudo()
    EmDisputa --> Concluida : laudoEmitido [decisao == VENDEDOR] / concluir()
    EmDisputa --> Estornada : laudoEmitido [decisao == COMPRADOR] / estornar()

    Concluida --> [*]
    Estornada --> [*]
```

## 5. Artefato 4 — Visão Arquitetural C4 (Nível 2 — Contêineres)

Arquivo: [`docs/architecture/workspace.mmd`](docs/architecture/workspace.mmd) (Mermaid C4)

```mermaid
C4Container
    title Marketplace B2B de Sobras - Visão de Contêineres (C4 Nível 2) - Despacho, Rastreio e Liberação da Custódia

    Person(vendedor, "Empresa Vendedora", "Indústria que publica a sobra, agenda a coleta e despacha o lote")
    Person(comprador, "Empresa Compradora", "Coletora que recebe o lote e confere a entrega")

    System_Ext(transportadora, "API da Transportadora", "Executa o transporte físico e envia eventos de rastreio")
    System_Ext(escrow, "Gateway de Escrow", "Mantém o valor em custódia e libera, congela ou estorna")
    System_Ext(canais, "Provedor de E-mail e Push", "Entrega de notificações transacionais")

    System_Boundary(mkt, "Marketplace B2B de Sobras") {
        Container(spa, "Portal Web", "React 18 + TypeScript", "Agendamento de coleta, acompanhamento do rastreio e conferência da entrega")
        Container(api, "Marketplace API", "Node.js 20 + NestJS", "Módulos Proposta, Transacao e Remessa; RemessaService e agregado Remessa (máquina de estados)")
        Container(worker, "Notificacao Worker", "Node.js 20", "Consome eventos de domínio e dispara notificações")
        ContainerQueue(mq, "Message Broker", "RabbitMQ 3.13", "Exchange topic marketplace.events com filas duráveis e DLQ")
        ContainerDb(db, "Banco do Marketplace", "PostgreSQL 16", "Tabelas remessa, checkpoint e outbox")
    }

    Rel(vendedor, spa, "Agenda a coleta e acompanha a remessa", "HTTPS")
    Rel(comprador, spa, "Acompanha o rastreio e confere a entrega", "HTTPS")
    Rel(spa, api, "agendarColeta e conferirEntrega", "HTTPS / REST JSON, JWT")
    Rel(api, db, "Lê e grava Remessa, checkpoints e outbox", "SQL / TCP 5432")
    Rel(api, transportadora, "solicitarColeta(req)", "HTTPS REST, TLS 1.3")
    Rel(transportadora, api, "webhookRastreio(CheckpointDTO)", "HTTPS REST, assinatura HMAC-SHA256")
    Rel(api, escrow, "liberarCustodia, congelarCustodia e estornarCustodia com Idempotency-Key", "HTTPS REST, TLS 1.3")
    Rel(api, mq, "Publica eventos de domínio via Outbox", "AMQP 0-9-1 assíncrono")
    Rel(mq, worker, "Entrega eventos de domínio", "AMQP 0-9-1")
    Rel(worker, canais, "Envia e-mail e push", "HTTPS REST")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```


