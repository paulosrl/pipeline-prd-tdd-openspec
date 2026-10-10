---
name: openspec-propose
description: >
  Adaptador do Estágio 3 do pipeline PRD → TDD → OpenSpec. NÃO gera artefatos OpenSpec — isso é trabalho do comando nativo /opsx:propose, dentro do repositório. Esta skill faz o que o OpenSpec não faz: valida o par PRD aprovado + TDD aprovado (relatório de recepção), decompõe o escopo total em changes ordenadas com aprovação do usuário, emite um BRIEFING por change pronto para colar no /opsx:propose, e fornece o checklist de auditoria da rastreabilidade dos artefatos gerados. Use esta skill SEMPRE que o usuário: tiver PRD e TDD aprovados e perguntar "e agora?", pedir para "preparar o OpenSpec", "decompor em changes", "gerar o briefing do propose", "auditar o propose", ou mencionar a ponte entre PRD/TDD e OpenSpec. Não use para escrever requisitos (skill prd), decidir tecnologia (skill tdd), nem para gerar proposal/spec/design/tasks (comando nativo do OpenSpec).
---

# Skill: Ponte para o OpenSpec (Estágio 3 — adaptador)

## Posição no pipeline e divisão de trabalho

```
Estágio 1 — PRD (skill prd)           → aprovado (gate 1)
Estágio 2 — TDD (skill tdd)           → aprovado (gate 2)
Estágio 3 — ESTA SKILL (fora do repo) → recepção + decomposição + briefings + auditoria
            /opsx:propose (no repo)   → gera proposal/spec/design/tasks vendo o código
        ↓ gate 3: artefatos auditados e aprovados
Implementação: /opsx:apply → verificação → archive → próxima change
```

**Fronteira inviolável:** esta skill nunca escreve `proposal.md`, `spec.md`, `design.md` ou `tasks.md`. Quem os gera é o OpenSpec nativo, dentro do repositório, com acesso ao código — única posição de onde changes sobre código existente (brownfield) podem ser especificadas corretamente. Se o usuário pedir que esta skill gere os artefatos diretamente, explique a fronteira em duas frases e gere o briefing.

**Pré-condições:** PRD aprovado (gate 1) e TDD aprovado (gate 2), com o TDD declarando o PRD de origem. Faltando um, direcione à skill correspondente.

## Princípios

1. **O OpenSpec executa; esta skill prepara e confere.** Entrada de qualidade (briefing) e saída auditada (checklist) — o meio pertence à ferramenta nativa.
2. **Nada se redecide no caminho.** O briefing instrui explicitamente o propose a referenciar ADRs, nunca reabri-los. Tecnologia redecidida no `design.md` é defeito de auditoria.
3. **Lacuna se devolve, não se resolve.** Falta de requisito → `PEN-PRD-XXX` (volta ao Estágio 1). Falta de decisão técnica → `PEN-TDD-XXX` (volta ao Estágio 2). O briefing lista as pendências que tocam a change; nenhuma se resolve por improviso dentro do OpenSpec.
4. **Decomposição completa, especificação progressiva.** Todas as changes são nomeadas e ordenadas de uma vez; o briefing é emitido um por vez, quando a change anterior estiver arquivada — a partir da change 02, o propose nativo precisa ver o código que já existe.
5. **Rastreabilidade é auditável nos dois sentidos.** PRD → artefatos (todo RF virou spec? todo CA virou cenário?) e artefatos → PRD (há spec sem RF? cenário inventado?).

## Processo

### 1. Recepção — validar o par PRD + TDD

Leia os dois documentos e verifique:

1. O TDD referencia este PRD na versão vigente?
2. Todo `RST-XXX` do PRD foi tratado no TDD (ADR ou dispensa justificada)?
3. Há `PEN-TDD-XXX` aberta que bloqueie a primeira change?
4. Todo `ESC-XXX` tem RF; todo RF tem CA?
5. Os ADRs cobrem o mínimo para implementar (topologia, linguagem, persistência, interface, ferramental)?

Emita no chat o **relatório de recepção**: `ACEITO` | `ACEITO_COM_PENDENCIAS` (listadas, com estágio de destino) | `DEVOLVIDO` (o que falta, para qual skill). Prossiga apenas de ACEITO/ACEITO_COM_PENDENCIAS.

### 2. Decomposição — o mapa de changes

Derive as changes dos `ESC-XXX`, aplicando os critérios:

1. **Valor independente:** cada change entrega comportamento observável cujos CAs passam sem as changes seguintes.
2. **Fundação primeiro:** a change 01 estabelece entidade central + operação primária + esqueleto dos ADRs; as demais incrementam.
3. **Tamanho de ciclo:** uma change cabe em uma sessão de implementação; grande demais divide, menor que um comportamento completo consolida.
4. **Dependência em prosa:** ordem declarada com razão ("02 depende de 01 porque exporta o que 01 cadastra").
5. **Ajuste pequeno não é change:** correção trivial em código arquivado não merece o rito de propose.

Apresente a tabela (change, ESC/RF cobertos, depende de, razão) e **aguarde aprovação**. Aprovada, salve `decomposicao-<nome>.md` — único arquivo de registro desta skill.

### 3. Briefing — a entrada do /opsx:propose

Para a change da vez, emita o briefing como bloco de texto pronto para colar no comando nativo, dentro do repositório:

```markdown
# Briefing de propose — Change 0N: [nome]

## Contexto
Produto: [nome] — PRD prd-<nome>.md v<X.Y> (gate 1) · TDD tdd-<nome>.md v<X.Y> (gate 2).
Esta é a change 0N de [total]; changes anteriores arquivadas: [lista ou "nenhuma"].

## Escopo DESTA change
[ESC-XXX cobertos. Para cada RF do recorte: ID, tipo EARS, descrição normativa
na íntegra. Regras de negócio (RN-XXX) na íntegra. Entidades (ENT-XXX) com
atributos de negócio. Fluxos (FL-XXX) com exceções. CAs (Gherkin) na íntegra.]

## Decisões técnicas herdadas (NÃO redecidir)
[Tabela: aspecto → decisão → ADR-XXX. Extraída do TDD, só o que toca esta change.]
Instrução: o design.md desta change deve REFERENCIAR estes ADRs pelo ID.
Decisão local legítima = organização interna de código que o TDD não cobre.
Qualquer necessidade de contrariar ou estender um ADR: PARE e reporte como
pendência — não decida.

## Fora desta change
[ESCs das changes seguintes + FE-XXX do PRD. Comportamento daqui não entra.]

## Pendências herdadas
[PEN-PRD/PEN-TDD que tocam esta change, com instrução de contorno, ou "nenhuma"]

## Exigências sobre os artefatos
1. Todo requisito do spec deve citar o RF de origem; todo cenário, o CA de origem.
2. Cláusulas em RFC 2119 (DEVE / NÃO DEVE / PODE), preservando a atomicidade do RF.
3. tasks com teste obrigatório ao final: cada cenário vira verificação executável.
4. Nenhum comportamento além dos RFs listados — o que falta se reporta, não se inventa.
```

Entregue com as instruções de uso:

1. No repositório (primeira vez): `openspec init`
2. Rode `/opsx:propose` colando o briefing
3. Traga os artefatos gerados de volta para a auditoria (passo 4)

### 4. Auditoria — conferir o que o OpenSpec gerou

Receba os artefatos (colados ou como arquivos) e execute o checklist:

| # | Verificação | Defeito se |
|---|---|---|
| 1 | Cobertura PRD → spec | RF do recorte sem spec correspondente |
| 2 | Cobertura CA → cenário | CA do recorte sem cenário |
| 3 | Sentido inverso | Spec/cenário sem origem no PRD (invenção) |
| 4 | Herança técnica | design.md redecide ou omite ADR aplicável |
| 5 | Decisões locais | DT local que na verdade contraria/estende ADR |
| 6 | Escopo | Comportamento de change futura ou de FE-XXX vazou |
| 7 | Tasks | Task sem spec de origem; ausência de tasks de teste |
| 8 | Pendências | Pendência herdada resolvida por improviso em vez de contornada |

Emita o **laudo de auditoria**: `APROVADO` (gate 3 cumprido — seguir ao apply) ou lista numerada de defeitos com a correção a pedir ao `/opsx:propose`. Reaudite após correção.

### 5. Ciclo

Após o archive da change, retorne ao passo 3 para o briefing da próxima — agora com "changes arquivadas" atualizado, para o propose nativo tratar o código existente como brownfield.

## Qualidade — antes de emitir cada briefing

1. **Autossuficiência:** o briefing contém os textos na íntegra (RF, RN, CA, ADR)? O /opsx:propose não tem acesso ao PRD/TDD — referência por ID sem o texto é briefing quebrado.
2. **Recorte limpo:** nenhum RF aparece em duas changes; nenhum RF do escopo total ficou órfão de change.
3. **Herança mínima suficiente:** só os ADRs que tocam a change — herança total é ruído.
4. **Teste do assistente:** com o briefing, o propose consegue trabalhar sem perguntas de produto ou de tecnologia?

## Saída

Arquivos desta skill: apenas `decomposicao-<nome>.md` (tabela aprovada + regra do ajuste pequeno) e, por ciclo, `briefing-change-0N-<nome>.md`. Artefatos OpenSpec vivem no repositório, gerados pela ferramenta nativa — nunca por esta skill.

Resumo por etapa no chat: recepção (veredicto), decomposição (nº de changes), briefing (change e recorte), auditoria (laudo).
