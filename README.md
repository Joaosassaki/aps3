# APS 3 — Dossiê de Arquitetura Viva

**Unidade Curricular:** Modelagem de Software — Engenharia de Software (4º período, UniSenai)
**Projeto da equipe:** Sistema de Gestão de Aprendizagem e Estágios da UniSenai (Grupo 6)
**Autor:** _Nome Sobrenome_ (trabalho individual)

## 1. Funcionalidade crítica modelada

**Candidatura → Termo de Compromisso → Assinatura Eletrônica** (RF04 e RF05 do backlog da Sprint 2).

O aluno solicita candidatura a uma demanda técnica. O sistema consulta o SGA (situação acadêmica) e o Sistema Fiep (regularidade da empresa) em paralelo, valida a elegibilidade e a vigência do convênio e calcula a compatibilidade. Estando tudo regular, confirma a candidatura, gera o `TermoCompromisso` e o envia para a Plataforma de Assinatura Eletrônica. A plataforma responde de forma assíncrona (webhook), uma vez por parte signatária. Quando todas assinam, o termo se torna vigente.

**Por que é crítica:** o termo tem validade jurídica, depende de três integrações externas e não pode ficar em estado inválido (por exemplo, vigente sem todas as assinaturas ou com convênio vencido).

**Entidade da máquina de estados:** `TermoCompromisso`.

## 2. Estrutura do repositório

```
aps3-nome-sobrenome/
├── docs/
│   ├── architecture/
│   │   ├── workspace.mmd      # C4 Nível 2 (Mermaid)
│   │   └── adr-0001.md        # ADR 0001
│   └── diagrams/
│       ├── sequencia.mmd
│       ├── atividades.mmd
│       └── estados.mmd
└── README.md
```

Os blocos abaixo são cópias dos arquivos `.mmd`. A fonte de verdade são os arquivos em `docs/`.

## 3. Artefato 1 — Diagrama de Sequência

Convenções: `->>` é chamada síncrona bloqueante, `-)` é mensagem assíncrona, `-->>` é retorno com DTO. Fragmentos usados: `par` (consultas paralelas), `alt/else` (tratamento de exceção) e `loop` (eventos de assinatura).

```mermaid
sequenceDiagram
    autonumber
    actor A as Aluno
    participant CS as CandidaturaService
    participant SGA as SistemaAcademicoSGA
    participant FIEP as SistemaFiep
    participant DT as DemandaTecnica
    participant CV as Convenio
    participant PM as PoliticaMatching
    participant C as Candidatura
    participant TC as TermoCompromisso
    participant CR as CandidaturaRepository
    participant TR as TermoRepository
    participant AS as PlataformaAssinatura

    A ->> CS: candidatar(alunoId, demandaId)

    par Consulta academica (SGA)
        CS ->> SGA: consultarAluno(alunoId)
        SGA -->> CS: AlunoDTO
    and Consulta de regularidade (Fiep)
        CS ->> FIEP: consultarRegularidade(cnpj)
        FIEP -->> CS: RegularidadeDTO
    end

    CS ->> DT: validarElegibilidade(aluno: AlunoDTO)
    DT -->> CS: boolean
    CS ->> CV: estaVigente()
    CV -->> CS: boolean

    alt [elegivel E convenioVigente E empresaRegular]
        CS ->> PM: calcular(perfil: PerfilCompetenciasDTO, demanda: DemandaTecnicaDTO)
        PM -->> CS: double (grauCompatibilidade)
        CS ->> C: confirmar()
        C -->> CS: TermoCompromissoDTO (status RASCUNHO)
        CS ->> CR: salvar(candidatura: CandidaturaDTO)
        CR -->> CS: CandidaturaDTO
        CS ->> TC: enviarParaAssinatura()
        TC -) AS: solicitarAssinatura(termo: TermoPayloadDTO)
        Note over TC,AS: Termo passa a AguardandoAssinaturas. A plataforma responde depois, por webhook.
        CS ->> TR: salvar(termo: TermoCompromissoDTO)
        TR -->> CS: TermoCompromissoDTO
        CS -->> A: CandidaturaDTO (status AGUARDANDO_ASSINATURAS)

        loop ate todas as partes assinarem
            AS -) CS: receberEventoAssinatura(evento: AssinaturaEventoDTO)
            CS ->> TC: registrarAssinatura(evento: AssinaturaEventoDTO)
            TC -->> CS: TermoCompromissoDTO
            CS ->> TR: salvar(termo: TermoCompromissoDTO)
            TR -->> CS: TermoCompromissoDTO
        end

        CS -) A: notificar(NotificacaoDTO: termo vigente)
    else [inelegivel OU convenioVencido OU empresaIrregular]
        CS -->> A: CandidaturaRejeitadaDTO (motivo)
    end
```

### Regra de consistência: mensagem → operação pública

| Linha de vida | Mensagem | Operação pública declarada | Situação |
|---|---|---|---|
| CandidaturaService | `candidatar(alunoId, demandaId)` | `+ candidatar(alunoId, demandaId): CandidaturaDTO` | Nova (camada de serviço) |
| CandidaturaService | `receberEventoAssinatura(evento)` | `+ receberEventoAssinatura(evento: AssinaturaEventoDTO): void` | Nova (endpoint do webhook) |
| SistemaAcademicoSGA | `consultarAluno(alunoId)` | `GET /alunos/{id}`, contrato da API externa | Externa |
| SistemaFiep | `consultarRegularidade(cnpj)` | `GET /empresas/{cnpj}/regularidade`, contrato da API externa | Externa |
| DemandaTecnica | `validarElegibilidade(aluno)` | `+ validarElegibilidade(a: Aluno): boolean` | Existente na Sprint 2 (o parâmetro vira `AlunoDTO`) |
| Convenio | `estaVigente()` | `+ estaVigente(): boolean` | Existente na Sprint 2 |
| PoliticaMatching | `calcular(perfil, demanda)` | `+ calcular(p, d): double` | Existente na Sprint 2 |
| Candidatura | `confirmar()` | `+ confirmar(): TermoCompromisso` | Existente na Sprint 2 |
| TermoCompromisso | `enviarParaAssinatura()` | `+ enviarParaAssinatura(): void` | Existente na Sprint 2 |
| TermoCompromisso | `registrarAssinatura(evento)` | `+ registrarAssinatura(e: AssinaturaEventoDTO): TermoCompromissoDTO` | **Nova: incluir no diagrama de classes** |
| CandidaturaRepository | `salvar(candidatura)` | `+ salvar(c: CandidaturaDTO): CandidaturaDTO` | Nova (interface) |
| TermoRepository | `salvar(termo)` | `+ salvar(t: TermoCompromissoDTO): TermoCompromissoDTO` | Nova (interface) |
| PlataformaAssinatura | `solicitarAssinatura(termo)` | `POST /solicitacoes`, contrato da API externa | Externa |

Os DTOs cruzam as fronteiras entre o domínio e as camadas de serviço e persistência, e entre o sistema e os serviços externos.

## 4. Artefato 2 — Diagrama de Atividades com Raias

São quatro raias: Aluno, Sistema Back-End, Serviços Externos e Coordenador. As decisões têm guardas mutuamente exclusivas. O **Fork/Join** paraleliza as consultas ao SGA e ao Fiep. O Mermaid não tem raias nativas, então as raias são emuladas com `subgraph`, e as barras de Fork e Join são nós pretos.

```mermaid
flowchart TB
    classDef bar fill:#000,stroke:#000,color:#fff
    classDef decisao fill:#fff7d6,stroke:#b58900,color:#000

    subgraph R1["Raia 1 - Aluno (ator)"]
        INI(("●"))
        A1["Seleciona a vaga e solicita candidatura"]
        A2["Visualiza a rejeicao e o motivo"]
    end

    subgraph R2["Raia 2 - Sistema Back-End (CandidaturaService)"]
        S1["Receber solicitacao e carregar DemandaTecnica"]
        F1["FORK"]:::bar
        J1["JOIN"]:::bar
        S2["Validar elegibilidade e vigencia do Convenio"]
        D1{"Pode se candidatar?"}:::decisao
        S3["Calcular grau de compatibilidade (PoliticaMatching)"]
        S4["Confirmar Candidatura e gerar Termo (RASCUNHO)"]
        S5["Enviar Termo para assinatura (AGUARDANDO_ASSINATURAS)"]
        D2{"Resultado das assinaturas?"}:::decisao
        S6["Ativar vigencia do Termo (VIGENTE)"]
        S7["Notificar aluno e empresa"]
        S8["Marcar Termo como RECUSADO ou EXPIRADO"]
        S9["Rejeitar Candidatura e registrar motivo"]
    end

    subgraph R3["Raia 3 - Servicos Externos"]
        E1["SGA: consultar matricula e coeficiente"]
        E2["Fiep: consultar regularidade da empresa"]
        E3["Plataforma de Assinatura: coletar assinaturas"]
    end

    subgraph R4["Raia 4 - Coordenador Pedagogico"]
        C1["Acompanha o estagio e registra ciclos"]
    end

    FIM(("◉"))

    INI --> A1 --> S1 --> F1
    F1 --> E1
    F1 --> E2
    E1 --> J1
    E2 --> J1
    J1 --> S2 --> D1
    D1 -->|"[elegivel E convenioVigente E empresaRegular]"| S3
    D1 -->|"[inelegivel OU convenioVencido OU empresaIrregular]"| S9
    S3 --> S4 --> S5 --> E3 --> D2
    D2 -->|"[todasAssinaturas]"| S6
    D2 -->|"[recusada OU prazoVencido]"| S8
    S6 --> S7 --> C1 --> FIM
    S8 --> FIM
    S9 --> A2 --> FIM
```

## 5. Artefato 3 — Diagrama de Transição de Estados (`TermoCompromisso`)

Transições no formato `Gatilho [Guarda] / Ação()`. O estado composto `AguardandoAssinaturas` tem subestados, e `Vigente` tem as ações internas `entry /`, `do /` e `exit /`. Os estados terminais são `Recusado`, `Expirado` e `Encerrado`. Entre `Rascunho` e `Vigente` só se chega passando por `AguardandoAssinaturas`, o que protege o termo contra estados inválidos.

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Rascunho : candidaturaConfirmada / criarTermo()
    Rascunho --> AguardandoAssinaturas : enviarParaAssinatura [convenioVigente] / solicitarAssinatura()

    state AguardandoAssinaturas {
        [*] --> SemAssinaturas
        SemAssinaturas --> ParcialmenteAssinado : assinaturaRecebida [faltamPartes] / registrarAssinatura()
        ParcialmenteAssinado --> ParcialmenteAssinado : assinaturaRecebida [faltamPartes] / registrarAssinatura()
    }

    AguardandoAssinaturas --> Vigente : assinaturaRecebida [todasAssinaturas] / ativarVigencia()
    AguardandoAssinaturas --> Recusado : assinaturaRecusada / notificarPartes()
    AguardandoAssinaturas --> Expirado : prazoExpirado [prazoVencido] / cancelarSolicitacao()
    Vigente --> Encerrado : fimVigencia [vigenciaFimAtingida] / encerrar()

    Recusado --> [*]
    Expirado --> [*]
    Encerrado --> [*]

    Rascunho : entry / gerarNumero()
    Vigente : entry / registrarInicioVigencia()
    Vigente : do / monitorarVigencia()
    Vigente : exit / notificarEncerramento()
```

## 6. Artefato 4 — Visão de Contêineres (C4 Nível 2)

Os conectores explicitam tecnologia e protocolo. As chamadas ao SGA e ao Fiep são síncronas, e a assinatura é assíncrona, com retorno por webhook.

```mermaid
C4Container
    title Gestao de Estagios UniSenai - Conteineres (C4 Nivel 2) - Candidatura e Termo de Compromisso

    Person(aluno, "Aluno", "Candidata-se a vagas e acompanha o status")
    Person(coord, "Coordenador Pedagogico", "Homologa convenios e acompanha o estagio")
    Person(empresa, "Empresa Parceira (RH)", "Publica demandas e assina o termo")

    System_Boundary(sis, "Sistema de Gestao de Aprendizagem e Estagios") {
        Container(spa, "Aplicacao Web", "React + TypeScript", "Interface dos tres perfis")
        Container(api, "API de Estagios", "Node.js + NestJS", "CandidaturaService, matching, regras de elegibilidade e maquina de estados do Termo")
        Container(webhook, "Receptor de Webhooks", "Supabase Edge Function (Deno)", "Recebe eventos de assinatura e valida a origem")
        ContainerDb(db, "Banco de Dados", "Supabase PostgreSQL 15 + RLS", "alunos, demandas, convenios, candidaturas, termos")
        Container(auth, "Autenticacao", "Supabase Auth", "Login e emissao de JWT com perfis")
        Container(storage, "Armazenamento de Arquivos", "Supabase Storage", "PDF do termo assinado")
    }

    System_Ext(sga, "Sistema Academico (SGA)", "Matricula, situacao e coeficiente (somente leitura)")
    System_Ext(fiep, "Sistema Fiep", "Dados cadastrais e regularidade das empresas (somente leitura)")
    System_Ext(assinatura, "Plataforma de Assinatura Eletronica", "Coleta e formaliza as assinaturas do termo")

    Rel(aluno, spa, "Usa", "HTTPS")
    Rel(coord, spa, "Usa", "HTTPS")
    Rel(empresa, spa, "Usa", "HTTPS")
    Rel(spa, auth, "Autentica", "HTTPS / JWT")
    Rel(spa, api, "Chama operacoes", "HTTPS / REST + JSON")
    Rel(api, db, "Le e grava", "HTTPS / PostgREST (supabase-js)")
    Rel(api, storage, "Envia e le PDFs", "HTTPS / S3 API")
    Rel(api, sga, "Consulta aluno (sincrono)", "HTTPS / REST + JSON")
    Rel(api, fiep, "Consulta regularidade (sincrono)", "HTTPS / REST + JSON")
    Rel(api, assinatura, "Solicita assinatura (assincrono)", "HTTPS / REST + JSON")
    Rel(assinatura, webhook, "Notifica assinatura", "HTTPS / Webhook + HMAC")
    Rel(webhook, api, "Repassa evento", "HTTPS / REST + JSON")
```

## 7. Artefato 5 — ADR 0001

Veja [`docs/architecture/adr-0001.md`](docs/architecture/adr-0001.md): **Adoção do Supabase (PostgreSQL gerenciado com RLS) como camada de dados**.

## 8. Consistência entre os modelos

| Conceito | Sequência | Atividades | Estados |
|---|---|---|---|
| Consulta paralela ao SGA e ao Fiep | `par ... and ... end` | Fork/Join | — |
| Elegibilidade e convênio vigente | `alt [elegivel E convenioVigente E empresaRegular]` | Decisão `Pode se candidatar?` | Guarda `[convenioVigente]` em `enviarParaAssinatura` |
| Envio do termo | `enviarParaAssinatura()` | `Enviar Termo para assinatura` | `Rascunho → AguardandoAssinaturas` |
| Assinatura assíncrona | `-)` + `loop` + `registrarAssinatura()` | `Plataforma de Assinatura: coletar assinaturas` | `assinaturaRecebida [faltamPartes]` |
| Todas as partes assinaram | fim do `loop` | `[todasAssinaturas]` | `→ Vigente / ativarVigencia()` |
| Recusa ou prazo vencido | — (coberto nos estados e nas atividades) | `[recusada OU prazoVencido]` | `Recusado` e `Expirado` |

## 9. Observações e pendências

- **Fiep na candidatura:** a consulta de regularidade no momento da candidatura é uma decisão desta APS. Na Sprint 2 ela aparece como `include` de *Homologar convênio*.
- **Diagrama de classes:** incluir `registrarAssinatura()` em `TermoCompromisso` e as interfaces `CandidaturaRepository` e `TermoRepository`.
- **Estados da `Candidatura`:** `status` é `String` na Sprint 2. Os estados do termo (`RASCUNHO`, `AGUARDANDO_ASSINATURAS`, `VIGENTE`, ...) devem usar exatamente estes nomes no código.
