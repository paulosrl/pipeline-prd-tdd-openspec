# Pipeline PRD → TDD → OpenSpec

Fluxo em três estágios para transformar uma ideia de produto em software desenvolvido com assistência de IA, separando **o que o produto faz** (decisão do dono do produto) de **com que tecnologia ele é construído** (decisão técnica assistida) e de **como e em que ordem implementar** (OpenSpec).

```
Ideia / necessidade
        ↓
┌─────────────────────────────────────────────────────────────┐
│ ESTÁGIO 1 — skill prd                                       │
│ Produto: O QUE e POR QUÊ. Agnóstico de tecnologia.          │
│ Saída: prd-<nome>.md                                        │
└─────────────────────────────────────────────────────────────┘
        ↓ GATE 1 — usuário aprova o produto
┌─────────────────────────────────────────────────────────────┐
│ ESTÁGIO 2 — skill tdd                                       │
│ Técnica: COM O QUÊ. Stack, arquitetura, modelo físico.      │
│ Saída: tdd-<nome>.md                                        │
└─────────────────────────────────────────────────────────────┘
        ↓ GATE 2 — usuário aprova a tecnologia
┌─────────────────────────────────────────────────────────────┐
│ ESTÁGIO 3 — OpenSpec                                        │
│ Execução: COMO e EM QUE ORDEM. Changes, specs, tasks.       │
│ Entrada: PRD + TDD                                          │
└─────────────────────────────────────────────────────────────┘
        ↓ GATE 3 — propose aprovado
Implementação por assistente de código (apply → verify → archive)
```

---

## Instalação

Como plugin no Claude Code (recomendado):

```bash
claude plugin marketplace add paulosrl/pipeline-prd-tdd-openspec
claude plugin install sdd@pipeline-prd-tdd-openspec
```

As skills ficam disponíveis como `/sdd:prd`, `/sdd:tdd` e `/sdd:briefing`.

Outras formas — skills avulsas em `~/.claude/skills/`, claude.ai, e prompts portáteis para outras plataformas de IA — em [`docs/instalacao.md`](docs/instalacao.md).

---

## Por que três estágios

| Princípio | Consequência prática |
|---|---|
| **Autoridade separada** | Quem aprova "telefone exige 11 dígitos" é o dono do produto; quem aprova "PostgreSQL com pgvector" precisa de contexto técnico. Cada decisão tem seu gate e seu aprovador. |
| **Reversibilidade assimétrica** | Requisito errado se corrige com uma edição; stack errada se corrige com reescrita. Decisões de custo de reversão diferente não dividem o mesmo documento nem o mesmo gate. |
| **Reuso do PRD** | O mesmo PRD gera TDDs diferentes (protótipo local vs. web multiusuário) sem reescrever um requisito. |
| **Fim da redecisão** | O `design.md` de cada change OpenSpec referencia os ADRs do TDD em vez de redecidir tecnologia a cada mudança. |

---

## Estágio 1 — Criar o PRD (`skill prd`)

### O que é

Documento de requisitos de **produto**: problema, objetivos, atores, escopo, requisitos funcionais em EARS, regras de negócio, entidades conceituais, fluxos, critérios de aceite em Gherkin, restrições, fora do escopo e matriz de rastreabilidade. **Zero tecnologia.**

### Como usar

1. Invoque a skill descrevendo o produto: nome, dor que resolve, para quem, o que entra na primeira versão.
2. Responda a única rodada de perguntas de contexto (se houver lacunas). A skill **não pergunta** linguagem, banco ou plataforma — isso é Estágio 2.
3. Aprove bloco a bloco ("ok" / "pode seguir"):
   - **Bloco 1** — Fundamentos: resumo, problema, objetivos, atores
   - **Bloco 2** — Escopo: o que entra e o que explicitamente fica fora
   - **Bloco 3** — Comportamento: RFs em EARS e regras de negócio
   - **Bloco 4** — Estrutura: entidades conceituais e fluxos
   - **Bloco 5** — Verificação: RNFs de produto, critérios de aceite, restrições, matriz
4. Pediu correção? A skill corrige **só aquele bloco** e reapresenta.
5. Após a aprovação final, a skill salva `prd-<nome>.md` e declara o **Gate 1 cumprido**.

### Regras que o PRD obedece

- **Proibido dentro do PRD:** linguagem, banco, frameworks, Docker, CI, modelo físico de dados (tipos, PK, FK), protocolos, diagrama de serviços. Tudo isso pertence ao TDD.
- **Única exceção:** imposição externa declarada espontaneamente pelo usuário ("a instituição exige Python") vira `RST-XXX` com origem e data — registrada como fato, validada no Estágio 2.
- **Todo elemento tem ID estável** (`PROB`, `OBJ`, `AT`, `ESC`, `RF`, `RN`, `ENT`, `FL`, `CA`, `RST`, `PRE`, `RIS`, `FE`), nunca renumerado.
- **Todo RF segue EARS** (`EVT`, `IND`, `EST`, `OPC`, `UBI`) e aponta escopo, critérios de aceite e regras relacionadas.

### Teste de prontidão do Gate 1

1. **Troca de stack:** se a tecnologia mudar por completo, alguma linha do PRD muda? Se sim (fora RSTs), há vazamento técnico.
2. **Leigo:** o dono do produto entende e consegue aprovar cada seção?
3. **IA:** todo comportamento tem RF atômico e verificável com CA? O que não está escrito será inventado.
4. **Rastreabilidade:** nenhum ID órfão na matriz.
5. **Fora do escopo nomeado:** "features de V2" não basta.

---

## Estágio 2 — Decidir a tecnologia (`skill tdd`)

### O que é

Technical Design Document: cada decisão técnica registrada como **ADR leve** (contexto → opções → decisão → consequências), mais diagrama de arquitetura, modelo físico de dados derivado das entidades do PRD, e ferramental.

### Pré-condição

PRD aprovado (Gate 1). Sem PRD, a skill aceita operar, mas marca o TDD como sujeito a revisão.

### Como usar — modo assistido (padrão, para quem não tem profundidade técnica)

1. Invoque a skill apontando o PRD aprovado.
2. A skill lê o PRD primeiro e **não pergunta o que ele já responde** (RSTs podem pré-decidir ADRs inteiros).
3. Responda a entrevista **por consequências** — perguntas em linguagem comum, nunca jargão:
   - "Quantas pessoas usam ao mesmo tempo?"
   - "Os dados podem morar só neste computador?"
   - "Precisa funcionar sem internet?"
   - "Se o computador quebrar hoje, perder os dados é aceitável, ruim ou catastrófico?"
   - "Quem mantém isso funcionando depois de pronto?"
   - "Existe orçamento para servidor, ou precisa ser custo zero?"
   - **"Não sei" é resposta válida** — vira o default mais simples que satisfaz o PRD, marcado `⚠️ Assumido`.
4. Para cada decisão, a skill apresenta no máximo 3 opções (uma linha cada) e **uma recomendação justificada pelas suas respostas** ("recomendo A porque você disse que B; C só valeria se D"). Você **aprova ou veta** — nunca escolhe no vácuo.
5. Cada ADR vem classificado por custo de reversão:
   - 🚪 **porta de um sentido** — caro de reverter (linguagem, banco, topologia): atenção máxima aqui
   - 🔁 **porta de dois sentidos** — barato de trocar depois: aprove rápido
6. Aprove em três blocos:
   - **Bloco 1** — ADRs de porta de um sentido (topologia, linguagem, persistência, interface)
   - **Bloco 2** — ADRs restantes e ferramental
   - **Bloco 3** — modelo físico de dados e diagrama de arquitetura
7. Após a aprovação final, a skill salva `tdd-<nome>.md` e declara o **Gate 2 cumprido**.

### Como usar — modo expert

Declare a stack diretamente. A skill valida a coerência com o PRD (restrições, RNFs, entidades), aponta conflitos, registra suas escolhas como ADRs com a sua justificativa e completa por entrevista reduzida apenas o que você não declarou.

### Como usar — modo "como está"

Para o sistema que já existe sem TDD (como o modo homônimo da skill `prd`). A skill lê as decisões no código, nos specs e nos documentos, cada ADR com a evidência por arquivo e símbolo; onde as fontes divergem, pergunta numa rodada única, e cada correção vira uma dívida a fechar por change.

### Defaults institucionais

Ponto de partida que a entrevista confirma ou desvia — desvio vira ADR justificado, nunca escolha silenciosa:

| Região | Default |
|---|---|
| Banco (topologia servidor) | PostgreSQL + pgvector |
| Python | ≥ 3.12.3 · `uv` exclusivo · faixas de compatibilidade (nunca `==` nem livre) · `ruff` + `pytest` + `mypy`/`pyright` · CI mínima |
| Node.js/TypeScript | Active LTS · `pnpm` · caret ranges · `Biome` + `Vitest` · `strict: true` · CI mínima |
| Docker | Compose v2 (sem `version:`) · `.env` · imagens em major + variante (`postgres:16-alpine`), nunca `latest` |
| Produto local simples | Docker e CI podem ser **dispensados por ADR explícito** — simplicidade também é decisão registrada |

### Regras que o TDD obedece

- **O PRD é soberano:** o TDD não altera requisito, regra ou critério de aceite. Conflito ou lacuna descoberta vira `PEN-TDD-XXX`, devolvida ao dono do PRD.
- **Toda decisão cita a causa:** resposta da entrevista, RST ou RNF. Decisão sem causa rastreável é opinião, não ADR.
- **Teste do leigo:** você deve conseguir explicar com as próprias palavras por que cada decisão 🚪 foi tomada. Se não consegue, a justificativa está em jargão — exija reescrita.

---

## Estágio 3 — OpenSpec (`skill briefing`)

1. Instale e inicialize o OpenSpec no repositório do projeto (`openspec init`), com o `openspec/config.yaml` do kit Trilha AI preenchido: é nele que ficam as regras por artefato (língua, origem citada, um teste por cenário).
2. Invoque a skill `briefing` com o PRD e o TDD aprovados: ela confere o par, decompõe o escopo em changes ordenadas (com a tabela "Ciclo"), emite um briefing por change e audita o que o `/opsx:propose` gerou. A decomposição em changes vem do escopo do PRD; a tecnologia vem dos ADRs do TDD.
3. Em cada change gerada, o `design.md` **referencia** os ADRs (`conforme ADR-003`) em vez de redecidir.
4. Ciclo por change: `propose` → aprovação (Gate 3) → `apply` com assistente de código → verificação (critérios de aceite do PRD viram testes) → `archive`.
5. Mudança de decisão técnica no meio da implementação = **nova versão do TDD**, nunca ajuste silencioso no `design.md`.

---

## Convenções de arquivos

```
projeto/
├── sdd/
│   ├── prd-<nome>.md                  # Estágio 1 — fonte de verdade do produto
│   ├── tdd-<nome>.md                  # Estágio 2 — fonte de verdade das decisões técnicas
│   ├── decomposicao-<nome>.md         # Estágio 3 — as changes, a ordem e a tabela "Ciclo"
│   └── briefing-openspec/
│       └── briefing-change-NN-<nome>.md
└── openspec/                          # Estágio 3 — changes, specs, tasks
    ├── config.yaml                    # contexto e regras por artefato (do kit)
    ├── specs/
    └── changes/
        └── NN-<change>/               # arquivada como AAAA-MM-DD-NN-<change>
            ├── proposal.md
            ├── design.md              # referencia ADRs do TDD
            ├── specs/<capability>/spec.md
            └── tasks.md
```

O ciclo de cada change, o nome das pastas, o `ANDAMENTO.md` e a verificação do repositório são regras do kit **Trilha AI** (`CLAUDE-comum.md`), que todo repositório do método copia; este plugin só produz os documentos de `sdd/`.

- Nomes em kebab-case: `prd-agenda-de-contatos.md`, `tdd-agenda-de-contatos.md`.
- O TDD declara no cabeçalho o PRD e a versão de origem; cada change declara o TDD que a governa.
- Versionamento: correção editorial = patch; decisão nova ou alterada = minor; mudança que invalida changes já arquivadas = major.

---

## Erros comuns e como o pipeline os evita

| Erro | Mecanismo de prevenção |
|---|---|
| PRD com stack embutida que trava o produto numa tecnologia | Tabela de proibições do Estágio 1 + teste da troca de stack |
| Leigo "aprovando" banco de dados que não entende | Entrevista por consequências + recomendação justificada + teste do leigo |
| Tecnologia decidida tarde, como remendo (ex.: linguagem entrando por alteração avulsa no PRD) | Estágio 2 é o lugar nomeado da decisão, antes de qualquer spec |
| IA inventando modelo de dados | Modelo físico derivado das `ENT-XXX` no TDD, antes do código |
| `design.md` de cada change redecidindo arquitetura | Regra de referência aos ADRs (TDD §9) |
| Decisão técnica alterando requisito por conta própria | Soberania do PRD + `PEN-TDD` devolvida ao dono |
| Escolha silenciosa de ferramenta | Defaults declarados; desvio só com ADR justificado |

---

## Perguntas frequentes

**Posso pular o TDD num projeto trivial?**
Não pule — reduza. O modo expert com meia dúzia de ADRs de uma linha custa minutos e deixa registrado por que o projeto é simples. É o ADR de dispensa (Docker, CI) que protege a simplicidade de virar acidente.

**E se eu já sei a stack?**
Modo expert do Estágio 2. A skill só valida coerência com o PRD e registra.

**O PRD pode citar tecnologia que o cliente impôs?**
Sim, como `RST-XXX` com origem e data — fato imposto, não decisão. O TDD a valida e a converte em ADR "decidido por RST-XXX".

**Quem aprova cada gate?**
Gate 1: dono do produto. Gate 2: dono do produto, com a assistência da entrevista (ou o técnico responsável, no modo expert). Gate 3: quem responde pela implementação.
