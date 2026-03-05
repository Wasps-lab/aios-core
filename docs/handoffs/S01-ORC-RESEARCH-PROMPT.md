# HANDOFF PROMPT — Sessao S01-ORC-RESEARCH

> **INSTRUCAO:** Cola este prompt inteiro numa nova sessao do Claude Code
> aberta na pasta `C:\Users\aless\Wasps\`. Ele contem 5 partes independentes
> que, juntas, dao todo o contexto necessario para implementar as melhorias.
>
> Se o prompt for demasiado longo para uma unica mensagem, divide por partes
> (cada `---PARTE X---` e auto-contida).

---

# ---PARTE 1: CONTEXTO E MISSAO---

## Quem es tu

Es uma nova sessao do Claude Code. Vais implementar melhorias concretas no
pipeline de research do orchestrator **Wasps Intelligence**.

**Working directory:** `C:\Users\aless\Wasps\`
**Foco:** Research pipeline (F1 + F1.5 + Gate). NAO tocar em fases posteriores.

## O que aconteceu antes

Uma sessao anterior analisou o repositorio **AIOS Core** (github.com/SynkraAI/aios-core),
um framework open-source de orquestracao de agentes IA com:
- 12 agentes especializados com contratos formais (inputs/outputs)
- Workflows YAML com phases, sequence, execution_modes, complexity_tiers
- Quality gates com verdicts (APPROVED/NEEDS_REVISION/BLOCKED)
- State persistence via ficheiros YAML
- Backlog management com lifecycle (IDEA -> TODO -> IN PROGRESS -> DONE -> ARCHIVED)
- Context brackets para gestao de context window (FRESH/MODERATE/DEPLETED/CRITICAL)
- Transitions com confidence scores entre fases

Dessa analise, foram extraidas **7 sugestoes de melhoria** (ORC-S01 a ORC-S07) para o
research pipeline do Wasps, priorizadas por impacto.

## A tua missao

Implementar TODAS as 7 sugestoes, pela ordem de prioridade:
1. **ALTA:** ORC-S03, ORC-S04, ORC-S07 (criar ficheiros novos)
2. **MEDIA:** ORC-S05, ORC-S01 (editar agentes existentes + reorganizar)
3. **BAIXA:** ORC-S06, ORC-S02 (documentar propostas)

Alem disso, implementar 2 padroes tecnicos transversais:
- Contratos formais de agentes (inputs/outputs YAML)
- Estado de execucao persistente por projecto

---

# ---PARTE 2: O QUE JA EXISTE (LER PRIMEIRO)---

**IMPORTANTE:** Antes de implementar qualquer coisa, le TODOS estes ficheiros
pela ordem indicada. Compreende como cada agente funciona, como comunicam
(ficheiros + YAML), e que gaps existem.

## Ficheiros obrigatorios (ler por esta ordem)

1. `orchestrator/agents/MELHORIA-PROCESSO-PESQUISA.md` — documento master de melhorias anteriores (TEM CONTEXTO CRUCIAL)
2. `orchestrator/agents/researcher-v2.md` — agente principal de research (F1)
3. `orchestrator/agents/research-database-schema.md` — schema do YAML centralizado
4. `orchestrator/agents/seo-auditor.md` — agente SEO (F1.5)
5. `orchestrator/agents/research-synthesizer.md` — sintetizador (F1.5)
6. `orchestrator/agents/competitive-analyst.md` — competitivo (F1.5)
7. `orchestrator/agents/brand-architect.md` — marca (F1.5)
8. `orchestrator/agents/micro-validator.md` — validacao com trafego (Gate)

## Ficheiros de referencia (ler se necessario)

9. `adc/research/research-database.yaml` — YAML real produzido (~700 linhas, projecto ADC)
10. `adc/DECISOES-ADC.md` — decisoes reais tomadas no projecto ADC

## Estrutura actual do pipeline

```
F1 (Discovery) — researcher-v2 executa passos P00 a P06 sequencialmente
  |
  v
F1.5 (Deep Analysis) — 4 agentes em paralelo:
  ├── seo-auditor
  ├── competitive-analyst
  ├── brand-architect
  └── research-synthesizer (depende dos 3 acima)
  |
  v
GATE — micro-validator (decisao GO/PIVOT/KILL)
  |
  v
F2 (Offer) — fora do escopo desta sessao
```

## Como comunicam actualmente
- **Ficheiros .md:** Cada agente produz ficheiros no formato `F1-P01-nome.md` ou `F1.5-SEO-nome.md`
- **YAML centralizado:** `{projecto}/research/research-database.yaml` — todas as seccoes actualizadas por agentes diferentes
- **Sem contratos formais:** Ninguem declara explicitamente o que precisa receber e o que produz
- **Sem estado persistente:** Se a sessao cair, nao ha forma de saber onde parou
- **Sem quality gate formal:** A decisao de avancar para F2 e informal

---

# ---PARTE 3: PADROES AIOS CORE (REFERENCIA)---

Estes sao os padroes reais extraidos do AIOS Core que inspiram as melhorias.
USA-OS como referencia de formato e estrutura, mas ADAPTA ao contexto Wasps
(que nao tem runtime de execucao — tudo e Claude Code + humano).

## Padrao 1: Workflow YAML (spec-pipeline.yaml)

```yaml
workflow:
  id: spec-pipeline
  name: Spec Pipeline - Requirements to Specification
  version: "1.0"
  type: pipeline

  # Triggers que iniciam o workflow
  triggers:
    - event: command
      command: "*create-spec"
      action: run_pipeline

  # Configuracao global
  config:
    strictGate: true        # BLOCKED verdict halts pipeline
    maxRetries: 2
    outputDir: docs/stories/{storyId}/spec/

  # Fases baseadas em complexidade
  phases:
    SIMPLE:
      description: "Tarefa direta, padroes existentes"
      steps: [gather, spec, critique]
      estimated_time: "30-60 min"
    STANDARD:
      description: "Complexidade moderada"
      steps: [gather, assess, research, spec, critique, plan]
      estimated_time: "2-4 hours"
    COMPLEX:
      description: "Alta complexidade, multiplas iteracoes"
      steps: [gather, assess, research, spec, critique_1, revise, critique_2, plan]
      estimated_time: "4-8 hours"
      flags:
        - "Architectural review recommended"

  # Sequencia de steps
  sequence:
    - step: gather
      phase: 1
      agent: pm
      task: spec-gather-requirements.md
      inputs:
        storyId: "{storyId}"
        source: "{source|user}"
      outputs:
        - requirements.json
      elicit: true           # Requer interaccao do utilizador
      on_success:
        log: "Requirements gathered"
        next: assess
      on_failure:
        action: halt

    - step: assess
      phase: 2
      agent: architect
      inputs:
        requirements: "docs/stories/{storyId}/spec/requirements.json"
      outputs:
        - complexity.json
      skip_if: "source === 'simple'"
      on_success:
        dynamic_phases: true  # Usa resultado para determinar fases
        next: research

    - step: critique
      phase: 5
      agent: qa
      gate: true             # Blocking gate
      on_verdict:
        APPROVED:
          next: plan
        NEEDS_REVISION:
          action: return_to_spec
          max_iterations: 2
        BLOCKED:
          action: halt
          escalate_to: "@architect"

  # Resume support
  resume:
    enabled: true
    state_file: docs/stories/{storyId}/spec/.pipeline-state.json
    checkpoints:
      - after: gather
        state: requirements_gathered
      - after: critique
        state: critique_complete
    resume_from:
      requirements_gathered: assess
      critique_complete: plan

  # Completion
  completion:
    success_message: "Spec Pipeline Complete"
    next_steps:
      - "Review specification"
      - "Start development"
```

**O que adaptar para o Wasps:**
- Mesma estrutura de phases/sequence/resume
- Substituir agents do AIOS por agentes do research (researcher-v2, seo-auditor, etc.)
- O "runtime" e Claude Code lendo o YAML como guia, nao execucao automatica
- Manter `skip_if`, `parallel`, `gate`, `on_verdict`

## Padrao 2: Workflow Transitions (workflow-patterns.yaml)

```yaml
workflows:
  story_development:
    agent_sequence: [po, dev, qa, devops]
    transitions:
      validated:
        trigger: "validate-story-draft completed"
        confidence: 0.90
        greeting_message: "Story validated! Ready to implement."
        next_steps:
          - command: develop-yolo
            description: "Autonomous mode (no interruptions)"
            priority: 1
          - command: develop-interactive
            description: "Interactive mode with checkpoints"
            priority: 2
      in_development:
        trigger: "develop completed"
        confidence: 0.85
        greeting_message: "Development complete! Ready for QA."
        next_steps:
          - command: review-qa
            priority: 1

  # State integration (como persistir estado entre sessoes)
  state_integration:
    state_file_location: ".aios/{instance-id}-state.yaml"
    detection_priority: [state_file, command_history]
    commands:
      start: "Creates state file, shows step 1"
      continue: "Loads state, advances to next step"
      status: "Shows progress bar and step checklist"
      skip: "Skips optional step"
      abort: "Sets status to aborted, preserves state"
```

**O que adaptar:** Definir transitions entre F1, F1.5 e GATE com confidence scores e next_steps sugeridos.

## Padrao 3: Quality Gate Config (quality-gate-config.yaml)

```yaml
version: "1.0"

# Checks individuais com severidade
layer1:
  enabled: true
  failFast: true
  checks:
    lint:
      enabled: true
      command: "npm run lint"
      failOn: "error"
      timeout: 60000
    test:
      enabled: true
      command: "npm test"
      timeout: 300000
      coverage:
        enabled: true
        minimum: 80

# Verdicts
# APPROVED: all checks pass
# NEEDS_REVISION: non-blocking issues found
# BLOCKED: critical issues found, pipeline halts
```

**O que adaptar:** Substituir checks de codigo por checks de research (scorecard completude, personas validadas, competitors, SEO keywords, hipoteses marcadas).

## Padrao 4: Agent Contracts (task-v3-schema)

```yaml
inputs:
  - campo: storyId
    tipo: string
    origem: "User Input"
    obrigatorio: true
    validacao: "Must match STORY-\\d+ pattern"
  - campo: requirements
    tipo: file
    origem: "Context"
    obrigatorio: true

outputs:
  - campo: spec.md
    tipo: file
    destino: "docs/stories/{storyId}/spec/"
    persistido: true

executionModes:
  yolo: { enabled: true, prompts: "0-1" }
  interactive: { enabled: true, prompts: "5-10" }
  preflight: { enabled: true, prompts: "10-15" }
  default: interactive
```

**O que adaptar:** Cada agente de research declara formalmente os seus inputs e outputs no mesmo formato.

## Padrao 5: Backlog Management

```
Item Types: F (Follow-up), T (Technical Debt), E (Enhancement)
Priority: Critical > High > Medium > Low
IDs: [STORY-013-F1], [STORY-013-T2]
Lifecycle: IDEA -> TODO -> IN PROGRESS -> DONE -> ARCHIVED
```

**O que adaptar:** Research Backlog com tipos GAPS (G), HYPOTHESES (H), ENHANCEMENTS (E) e IDs `[RB-{PROJ}-G01]`.

---

# ---PARTE 4: PLANO DE IMPLEMENTACAO---

Implementa pela ordem abaixo. Cada sugestao tem descricao, exemplo de output esperado, e criterios de validacao.

## PRIORIDADE ALTA

### ORC-S03 — Research Pipeline YAML formal

**Criar:** `orchestrator/workflows/research-pipeline.yaml`

**O que fazer:**
- Documentar todo o pipeline F1 -> F1.5 -> GATE como workflow YAML formal
- Incluir TODOS os steps reais (P00 a P06 do F1, os 4 agentes do F1.5, GATE)
- Definir 3 complexity_tiers: SIMPLE, STANDARD, COMPLEX
- Definir 3 execution_modes: yolo, interactive, preflight
- Declarar inputs/outputs de cada step
- Marcar `parallel: true` para F1.5
- Marcar `gate: true` para GATE
- Declarar `updates_yaml` por step (que seccoes do research-database.yaml cada agente toca)
- Incluir `skip_if` para steps condicionais (ex: P00 skip se nao ha assets)
- Incluir transitions com confidence scores
- Incluir resume support com checkpoints
- Incluir completion message com next_steps

**Exemplo parcial de output:**

```yaml
workflow:
  id: research-pipeline
  name: "Research Pipeline — Discovery to Validation"
  version: "2.0"

  execution_modes:
    - mode: yolo
      description: "Executa todo o pipeline autonomamente (projectos novos simples)"
      prompts: "0-2"
    - mode: interactive
      description: "Checkpoints entre fases (default, recomendado)"
      prompts: "5-8"
    - mode: preflight
      description: "Planeia tudo antes de executar (projectos complexos como BMH)"
      prompts: "10-15"

  complexity_tiers:
    SIMPLE:
      description: "Projecto novo, pouco historico, nicho claro"
      phases: [F1, F1.5, GATE]
      skip: [P00, P02B]
      estimated_sessions: 2-3
    STANDARD:
      description: "Projecto com algum historico e assets"
      phases: [F1, F1.5, GATE]
      skip: []
      estimated_sessions: 4-5
    COMPLEX:
      description: "Projecto maduro com muito historico (BMH)"
      phases: [F1, F1.5, F1.5-DEEP, GATE]
      skip: []
      extra: [historical-analysis, rebrand-assessment]
      estimated_sessions: 6-8

  sequence:
    # F1 Steps
    - step: P00-auditoria-assets
      phase: F1
      agent: researcher-v2
      description: "Inventario de assets existentes"
      inputs: [project-briefing, existing-materials]
      outputs: [F1-P00-auditoria-assets.md]
      updates_yaml: [assets]
      skip_if: "no_existing_assets"

    - step: P05-espionagem-competitiva
      phase: F1
      agent: researcher-v2
      description: "Analise competitiva inicial"
      inputs: [project-briefing, competitor-urls]
      outputs: [F1-P05-espionagem-competitiva.md]
      updates_yaml: [competitors]

    # ... (TODOS os steps P00-P06 do researcher-v2)

    # F1.5 Steps (paralelo)
    - step: F1.5-parallel
      phase: F1.5
      parallel: true
      agents:
        - agent: seo-auditor
          inputs: [research-database.yaml]
          outputs: [F1.5-SEO-*.md]
          updates_yaml: [seo]
        - agent: competitive-analyst
          inputs: [research-database.yaml]
          outputs: [F1.5-COMP-*.md]
          updates_yaml: [competitors]
        - agent: brand-architect
          inputs: [research-database.yaml]
          outputs: [F1.5-BRAND-*.md]
          updates_yaml: [brand]
        - agent: research-synthesizer
          inputs: [research-database.yaml, all-F1.5-outputs]
          outputs: [F1.5-SYN-*.md, F1.5-SYN-scorecard.md]
          updates_yaml: [synthesis]
          depends_on: [seo-auditor, competitive-analyst, brand-architect]

    # GATE
    - step: GATE-1-micro-validation
      phase: GATE
      agent: micro-validator
      gate: true
      inputs: [F1.5-SYN-scorecard.md, research-database.yaml]
      outputs: [GATE-1-result.md]
      on_verdict:
        GO: { next: F2-offer }
        PIVOT: { action: return_to_F1, pass: [gaps-identified] }
        KILL: { action: halt, escalate_to: "human" }

  transitions:
    F1_complete:
      trigger: "All P00-P06 outputs exist + YAML updated"
      confidence: 0.90
      next: F1.5-parallel
    F1.5_complete:
      trigger: "Scorecard completude >= 80%"
      confidence: 0.85
      next: GATE-1-micro-validation

  resume:
    enabled: true
    state_file: "{projecto}/research/.research-state.yaml"
    checkpoints:
      - after: F1
        state: discovery_complete
      - after: F1.5
        state: analysis_complete
      - after: GATE
        state: validated
```

**Criterio de validacao:** O ficheiro deve conter TODOS os steps reais do pipeline (verificar contra os agentes lidos na Parte 2).

---

### ORC-S04 — Research Backlog por projecto

**Criar:** `orchestrator/templates/research-backlog-tmpl.md`

**O que fazer:**
- Template reutilizavel para qualquer projecto
- 3 categorias: GAPS (pesquisa incompleta), HYPOTHESES (precisam validacao), ENHANCEMENTS (melhorias)
- IDs contextuais: `[RB-{PROJ}-G01]`, `[RB-{PROJ}-H01]`, `[RB-{PROJ}-E01]`
- Lifecycle: IDEA -> TODO -> IN PROGRESS -> DONE -> ARCHIVED
- Seccao de estatisticas (total, por tipo, por status)
- Instrucoes de como os agentes devem alimentar o backlog

**Exemplo de output esperado:**

```markdown
# Research Backlog — {PROJECTO}

> Este backlog e alimentado automaticamente pelos agentes de research
> durante as fases F1 e F1.5. Items aqui representam gaps, hipoteses
> nao validadas e melhorias identificadas.

## Estatisticas
| Tipo | Total | TODO | In Progress | Done |
|------|-------|------|-------------|------|
| GAPS | 0 | 0 | 0 | 0 |
| HYPOTHESES | 0 | 0 | 0 | 0 |
| ENHANCEMENTS | 0 | 0 | 0 | 0 |

## GAPS (Pesquisa Incompleta)
Items que precisam de mais investigacao.

| ID | Titulo | Origem | Status | Prioridade |
|----|--------|--------|--------|------------|
| [RB-{PROJ}-G01] | Exemplo: dados de pricing incompletos | F1-P03 | TODO | High |

## HYPOTHESES (Precisam Validacao)
Hipoteses formuladas durante o research que ainda nao foram validadas com dados.

| ID | Titulo | Evidencia | Status | Prioridade |
|----|--------|-----------|--------|------------|
| [RB-{PROJ}-H01] | Exemplo: audiencia prefere video curto | [HIPOTESE] sem dados | TODO | Medium |

## ENHANCEMENTS (Melhorias)
Ideias de melhoria identificadas durante o research.

| ID | Titulo | Impacto Estimado | Status | Prioridade |
|----|--------|------------------|--------|------------|
| [RB-{PROJ}-E01] | Exemplo: adicionar analise de CPC | Alto | IDEA | Low |

## Instrucoes para Agentes
- Ao encontrar um gap durante qualquer fase, adicionar aqui com tag [DADO], [INFERENCIA] ou [HIPOTESE]
- Ao validar uma hipotese, mover de HYPOTHESES para DONE com evidencia
- Ao encontrar melhoria possivel, adicionar em ENHANCEMENTS com impacto estimado
- IDs sao sequenciais: G01, G02... H01, H02... E01, E02...
```

---

### ORC-S07 — Quality Gate formal entre F1.5 e F2

**Criar:** `orchestrator/gates/research-completeness-gate.yaml`

**O que fazer:**
- Checklist objectiva com 5+ criterios
- Cada criterio tem: id, nome, condition, severity (BLOCK/WARN/INFO), message
- Gate decision: GO / CONCERNS / FAIL
- Bloqueia avanco para F2 se criterios BLOCK nao passarem

**Output esperado completo:**

```yaml
gate:
  id: research-completeness-gate
  name: "Research Completeness Gate"
  version: "1.0"
  trigger: "F1.5 complete (all specialist agents finished)"
  evaluator: "human + research-synthesizer scorecard"

  checks:
    - id: RG-01
      name: "Scorecard completude"
      condition: "scorecard.overall >= 75%"
      severity: BLOCK
      message: "Research incompleto. Scorecard em {score}%. Minimo: 75%."
      how_to_check: "Verificar F1.5-SYN-scorecard.md, campo overall_score"

    - id: RG-02
      name: "Personas validadas com evidencia"
      condition: "personas.count >= 2 AND personas.all_have_evidence == true"
      severity: BLOCK
      message: "Menos de 2 personas validadas ou personas sem evidencia."
      how_to_check: "Verificar research-database.yaml secao audience.personas"

    - id: RG-03
      name: "Competitors analisados"
      condition: "competitors.analyzed >= 3"
      severity: WARN
      message: "Menos de 3 competitors analisados. Considerar adicionar mais."
      how_to_check: "Contar entries em research-database.yaml secao competitors"

    - id: RG-04
      name: "SEO keywords com volume validado"
      condition: "seo.validated_keywords >= 10"
      severity: WARN
      message: "Menos de 10 keywords com volume validado."
      how_to_check: "Verificar research-database.yaml secao seo.keywords com volume > 0"

    - id: RG-05
      name: "Zero hipoteses nao marcadas"
      condition: "hypotheses.untagged == 0"
      severity: BLOCK
      message: "Existem afirmacoes sem tag [DADO]/[INFERENCIA]/[HIPOTESE]."
      how_to_check: "Grep por afirmacoes sem tags nos ficheiros F1.5"

    - id: RG-06
      name: "YAML centralizado actualizado"
      condition: "yaml.last_updated within 48h AND yaml.sections_filled >= 80%"
      severity: WARN
      message: "YAML desactualizado ou incompleto."
      how_to_check: "Verificar timestamp e seccoes preenchidas no research-database.yaml"

    - id: RG-07
      name: "Research Backlog triado"
      condition: "backlog.gaps.unresolved <= 3"
      severity: WARN
      message: "{count} gaps nao resolvidos no Research Backlog."
      how_to_check: "Verificar RESEARCH-BACKLOG.md secao GAPS com status != DONE"

  decision:
    all_BLOCK_pass_and_no_WARN: GO
    all_BLOCK_pass_some_WARN: CONCERNS
    any_BLOCK_fail: FAIL

  actions:
    GO: "Avancar para F2 (Offer Design)"
    CONCERNS: "Avancar com ressalvas documentadas no backlog"
    FAIL: "Voltar a F1.5, resolver BLOCK items, re-executar gate"
```

---

## PRIORIDADE MEDIA

### ORC-S05 — Transition Triggers + Contratos nos agentes

**Editar:** CADA agente de research (6 ficheiros)
**NAO reescrever o agente inteiro** — apenas ADICIONAR seccoes ao final.

**Para cada agente, adicionar 2 seccoes:**

**Seccao 1: Contrato Formal (inputs/outputs)**

```markdown
## Contrato Formal

### Inputs
| Campo | Tipo | Origem | Obrigatorio | Descricao |
|-------|------|--------|-------------|-----------|
| research-database.yaml | file | Context | Sim | YAML centralizado do projecto |
| project-briefing | text | User Input | Sim | Brief inicial do projecto |

### Outputs
| Campo | Tipo | Destino | Persistido | Descricao |
|-------|------|---------|------------|-----------|
| F1.5-SEO-audit.md | file | {proj}/research/ | Sim | Relatorio completo de SEO |
| research-database.yaml | file | {proj}/research/ | Sim | YAML actualizado (seccao seo) |

### Seccoes YAML que actualiza
- `seo.keywords`
- `seo.competitors_seo`
- `seo.content_gaps`
```

**Seccao 2: Completion Output (transition trigger)**

```markdown
## Completion Output

Ao terminar, SEMPRE emitir no final do ultimo ficheiro produzido:

---
### TRANSITION SUGGESTION
- **Status:** COMPLETE | PARTIAL | BLOCKED
- **Confidence:** 85%
- **Next recommended:** @research-synthesizer (F1.5 synthesis)
- **Reason:** "Auditoria SEO completa. 15 keywords validadas, 3 content gaps identificados."
- **Blockers:** Nenhum
- **Gaps identified:** [lista de gaps para adicionar ao Research Backlog]
- **YAML sections updated:** [seo.keywords, seo.competitors_seo]
---
```

**Agentes a editar (adaptar inputs/outputs para cada um):**

| Agente | Ficheiro | Inputs principais | Outputs principais | YAML sections |
|--------|----------|-------------------|--------------------| --------------|
| researcher-v2 | `orchestrator/agents/researcher-v2.md` | project-briefing, competitor-urls | F1-P00 a F1-P06.md | audience, competitors, market |
| seo-auditor | `orchestrator/agents/seo-auditor.md` | research-database.yaml | F1.5-SEO-*.md | seo |
| competitive-analyst | `orchestrator/agents/competitive-analyst.md` | research-database.yaml | F1.5-COMP-*.md | competitors |
| brand-architect | `orchestrator/agents/brand-architect.md` | research-database.yaml | F1.5-BRAND-*.md | brand |
| research-synthesizer | `orchestrator/agents/research-synthesizer.md` | research-database.yaml, all F1.5 outputs | F1.5-SYN-*.md, scorecard | synthesis |
| micro-validator | `orchestrator/agents/micro-validator.md` | scorecard, research-database.yaml | GATE-1-result.md | (nao actualiza) |

---

### ORC-S01 — Reorganizar como Research Squad

**O que fazer:**
- Criar directorio `orchestrator/squads/research/`
- Mover (ou criar symlinks) agentes de research para `orchestrator/squads/research/agents/`
- Criar `orchestrator/squads/research/squad.yaml` com metadata

**squad.yaml esperado:**

```yaml
squad:
  id: research-squad
  name: "Research & Discovery Squad"
  version: "1.0"
  description: "Squad especializado em pesquisa de mercado, analise competitiva, SEO e validacao"

  agents:
    - id: researcher-v2
      role: "Lead Researcher"
      phase: F1
      file: agents/researcher-v2.md
    - id: seo-auditor
      role: "SEO Specialist"
      phase: F1.5
      file: agents/seo-auditor.md
    - id: competitive-analyst
      role: "Competitive Intelligence"
      phase: F1.5
      file: agents/competitive-analyst.md
    - id: brand-architect
      role: "Brand Strategist"
      phase: F1.5
      file: agents/brand-architect.md
    - id: research-synthesizer
      role: "Research Synthesizer"
      phase: F1.5
      file: agents/research-synthesizer.md
    - id: micro-validator
      role: "Validation Gate"
      phase: GATE
      file: agents/micro-validator.md

  workflows:
    - research-pipeline.yaml

  gates:
    - research-completeness-gate.yaml

  templates:
    - research-backlog-tmpl.md

  data_schema:
    - research-database-schema.md
```

**NOTA:** NAO mover os ficheiros fisicamente se isso quebrar referencias existentes.
Criar o squad.yaml como indice/manifest que aponta para os ficheiros actuais.
So mover fisicamente quando confirmado pelo Alessandro.

---

## PRIORIDADE BAIXA (DOCUMENTAR, NAO IMPLEMENTAR)

### ORC-S06 — Context Tiers no YAML central

**Criar:** `orchestrator/proposals/ORC-S06-context-tiers.md`

**Conteudo da proposta:**
- Definir 3 tiers de dados no research-database.yaml:
  - ESSENTIAL (~100 linhas): metadata do projecto, personas top-level, keywords top 5, competitor names
  - STANDARD (~300 linhas): tudo acima + detalhes de competitors, all keywords, brand attributes
  - DEEP (~700 linhas): YAML completo
- Cada agente declara o tier que precisa no seu contrato
- Implementar quando tiver 4+ projectos activos (actualmente tem 2-3)
- Estimar esforco: 1 sessao de ~2h

### ORC-S02 — Lentes de analise no Research Synthesizer

**Criar:** `orchestrator/proposals/ORC-S02-analysis-lenses.md`

**Conteudo da proposta:**
- Adicionar 3 "lentes" ao research-synthesizer:
  - **Customer-Obsessed:** "O que o cliente realmente precisa? Que dor resolve?"
  - **Contrarian:** "E se todos estiverem errados? Que assumption nao foi testada?"
  - **Opportunity-Hunter:** "Que oportunidade ninguem esta a explorar?"
- Forca analise multi-perspectiva em vez de voz unica neutra
- Estimar esforco: 30 min de edicao no synthesizer

---

## PADRAO TECNICO TRANSVERSAL: Estado de Execucao Persistente

**Criar template:** `orchestrator/templates/research-state-tmpl.yaml`

**O que fazer:**
- Template de estado que e instanciado como `{projecto}/research/.research-state.yaml`
- Rastreia: phase actual, steps completados, gaps encontrados, backlog items
- Resolve o problema de "agentes nao sabem o estado uns dos outros"
- Resolve o problema de "se a sessao cair, nao ha forma de retomar"

**Template esperado:**

```yaml
# Research Pipeline State — {PROJECTO}
# Auto-generated. Updated by agents during execution.

pipeline_instance: "{proj}-research-{date}"
project: "{PROJECTO}"
status: active  # active | paused | completed | aborted
created: "{date}"
last_updated: "{date}"

current_phase: F1  # F1 | F1.5 | GATE | COMPLETE
current_step: P00  # Step ID dentro da phase

completed_steps:
  # Preenchido automaticamente pelos agentes
  # - step_id: { status: done, date: YYYY-MM-DD, agent: agent-name, outputs: [files] }

pending_steps:
  # Gerado a partir do research-pipeline.yaml
  # - step_id: { status: pending, agent: agent-name }

gaps_found: 0
backlog_items: 0
yaml_sections_updated: []

# Historico de sessoes
sessions:
  # - date: YYYY-MM-DD
  #   duration: "2h"
  #   steps_completed: [P00, P01]
  #   notes: "..."
```

---

# ---PARTE 5: REGRAS, DECISOES E OUTPUT---

## Regras

- **NAO alterar ficheiros dentro de `adc/`, `bmh/`, `eft/`, `fmr/`** — sao projectos, o orchestrator e generico
- **NAO reescrever agentes inteiros** — adicionar seccoes, nao substituir conteudo existente
- **NAO fazer push para GitHub/VPS** sem instrucao explicita do Alessandro
- **PROPOR antes de implementar** se algo nao estiver claro
- **Manter simplicidade** — o runtime e Claude Code + humano, nao ha infra de execucao automatica
- **Filosofia incremental** — cada sessao melhora um pouco, nao reescreve tudo
- **Focar APENAS em research** — ignorar completamente fases posteriores (Offer, Copy, Tech, Launch)
- **Naming convention existente:** manter `F1-P01-nome.md` e `F1.5-SYN-nome.md`
- **YAML centralizado:** manter como ficheiro unico (nao fragmentar)

## Decisoes ja tomadas

| # | Decisao | Status |
|---|---------|--------|
| ORC-D01 | Manter comunicacao via ficheiros + YAML (nao message queue) | DECIDIDO: MANTER |
| ORC-D02 | YAML centralizado como ficheiro unico | DECIDIDO: MANTER (tiers futuro) |
| ORC-D03 | Contratos formais de agentes em YAML | TODO nesta sessao (ORC-S05) |
| ORC-D04 | Feedback loop via Research Backlog | TODO nesta sessao (ORC-S04) |
| ORC-D05 | Naming convention: F1-P01-nome.md, F1.5-SYN-nome.md | DECIDIDO: MANTER |

## Output esperado desta sessao

| # | Ficheiro | Tipo | Sugestao |
|---|----------|------|----------|
| 1 | `orchestrator/workflows/research-pipeline.yaml` | NOVO | ORC-S03 |
| 2 | `orchestrator/templates/research-backlog-tmpl.md` | NOVO | ORC-S04 |
| 3 | `orchestrator/gates/research-completeness-gate.yaml` | NOVO | ORC-S07 |
| 4 | `orchestrator/templates/research-state-tmpl.yaml` | NOVO | Transversal |
| 5 | `orchestrator/agents/researcher-v2.md` | EDITADO | ORC-S05 |
| 6 | `orchestrator/agents/seo-auditor.md` | EDITADO | ORC-S05 |
| 7 | `orchestrator/agents/competitive-analyst.md` | EDITADO | ORC-S05 |
| 8 | `orchestrator/agents/brand-architect.md` | EDITADO | ORC-S05 |
| 9 | `orchestrator/agents/research-synthesizer.md` | EDITADO | ORC-S05 |
| 10 | `orchestrator/agents/micro-validator.md` | EDITADO | ORC-S05 |
| 11 | `orchestrator/squads/research/squad.yaml` | NOVO | ORC-S01 |
| 12 | `orchestrator/proposals/ORC-S06-context-tiers.md` | NOVO | ORC-S06 |
| 13 | `orchestrator/proposals/ORC-S02-analysis-lenses.md` | NOVO | ORC-S02 |
| 14 | `orchestrator/sessions/S01-ORC-RESEARCH-20260305.md` | NOVO | Documentacao |

## Ordem de execucao

```
1. LER todos os ficheiros da Parte 2 (NAO PULAR)
2. ORC-S03: Criar research-pipeline.yaml
3. ORC-S04: Criar research-backlog-tmpl.md
4. ORC-S07: Criar research-completeness-gate.yaml
5. Transversal: Criar research-state-tmpl.yaml
6. ORC-S05: Editar 6 agentes (contratos + completion output)
7. ORC-S01: Criar squad.yaml
8. ORC-S06: Documentar proposta context tiers
9. ORC-S02: Documentar proposta analysis lenses
10. Criar session log: S01-ORC-RESEARCH-20260305.md
```

---

*Handoff gerado pela sessao AIOS Core — 05/03/2026.*
*Baseado em analise do repositorio github.com/SynkraAI/aios-core.*
