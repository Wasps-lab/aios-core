# HANDOFF — S01-ORC-RESEARCH (Optimização do Research Pipeline)

**Projecto:** Wasps Intelligence — Orchestrator
**Data de criação:** 05/03/2026
**Série:** ORC (Optimização Orchestrator)
**Sessão anterior:** Sessão AIOS Core (análise do repo GitHub + extracção de padrões)
**Autor:** Claude (via Alessandro)

---

## QUEM ÉS TU

És uma nova sessão do Claude Code. Vais implementar melhorias concretas no
pipeline de research do orchestrator Wasps Intelligence, baseado em padrões
extraídos do AIOS Core (framework open-source de orquestração de agentes IA).

**O teu working directory é:** `C:\Users\aless\Wasps\`

**FOCO EXCLUSIVO:** Research pipeline (F1 + F1.5 + Gate). NÃO tocar em
fases posteriores (Offer, Copy, Tech, Launch, etc.).

---

## CONTEXTO — O QUE A SESSÃO ANTERIOR DESCOBRIU

A sessão anterior analisou o repositório AIOS Core (github.com/SynkraAI/aios-core)
e extraiu 7 sugestões de melhoria para o research, ordenadas por prioridade:

### Sugestões Extraídas (por prioridade)

#### PRIORIDADE ALTA

**ORC-S03 — Research Pipeline YAML formal**
- Criar `orchestrator/workflows/research-pipeline.yaml`
- Definir todas as fases (F1, F1.5, Gate) como workflow YAML com:
  - `sequence` de steps com agent, inputs, outputs
  - `execution_modes` (yolo, interactive, preflight)
  - `complexity_tiers` (SIMPLE, STANDARD, COMPLEX)
  - `transitions` com confidence scores
  - `parallel: true` para F1.5
  - `skip_if` para passos condicionais
  - `updates_yaml` para declarar que secções do YAML cada agente toca
- Inspirado no `spec-pipeline.yaml` e `workflow-patterns.yaml` do AIOS Core
- Exemplo de estrutura:

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
      description: "Projecto novo, pouco histórico, nicho claro"
      phases: [F1, F1.5, GATE]
      skip: [P00, P02B]
      estimated_sessions: 2-3
    STANDARD:
      description: "Projecto com algum histórico e assets"
      phases: [F1, F1.5, GATE]
      skip: []
      estimated_sessions: 4-5
    COMPLEX:
      description: "Projecto maduro com muito histórico (BMH)"
      phases: [F1, F1.5, F1.5-DEEP, GATE]
      skip: []
      extra: [historical-analysis, rebrand-assessment]
      estimated_sessions: 6-8

  sequence:
    - step: P00-auditoria-assets
      phase: F1
      agent: researcher-v2
      description: "Inventário de assets existentes"
      inputs: [project-briefing, existing-materials]
      outputs: [F1-P00-auditoria-assets.md]
      updates_yaml: [assets]
      skip_if: "no_existing_assets"

    - step: P05-espionagem-competitiva
      phase: F1
      agent: researcher-v2
      description: "Análise competitiva inicial"
      inputs: [project-briefing, competitor-urls]
      outputs: [F1-P05-espionagem-competitiva.md]
      updates_yaml: [competitors]

    # ... (demais steps)

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

    - step: GATE-1-micro-validation
      phase: GATE
      agent: micro-validator
      inputs: [F1.5-SYN-scorecard.md, research-database.yaml]
      outputs: [GATE-1-result.md]
      gate_decision: [GO, PIVOT, KILL]

  transitions:
    F1_complete:
      trigger: "All P00-P06 outputs exist + YAML updated"
      confidence: 0.90
      next: F1.5-parallel
    F1.5_complete:
      trigger: "Scorecard completude >= 80%"
      confidence: 0.85
      next: GATE-1-micro-validation
```

**ORC-S04 — Research Backlog por projecto**
- Criar template `orchestrator/templates/research-backlog-tmpl.md`
- Instanciar como `{projecto}/research/RESEARCH-BACKLOG.md`
- 3 categorias: GAPS (pesquisa incompleta), HYPOTHESES (precisam validação), ENHANCEMENTS (melhorias)
- IDs contextuais: `[RB-{PROJ}-G01]`, `[RB-{PROJ}-H01]`, etc.
- Lifecycle: IDEA → TODO → IN PROGRESS → DONE → ARCHIVED
- Alimentado pelos agentes durante o research (cada agente adiciona o que encontra em falta)
- Este backlog é o destino concreto do feedback loop (F7 → F1)

**ORC-S07 — Quality Gate formal entre F1.5 e F2**
- Criar `orchestrator/gates/research-completeness-gate.yaml`
- Checklist objectiva com 5+ critérios:
  - Scorecard completude >= 75%
  - Pelo menos 2 personas validadas com evidência
  - >= 3 competitors analisados
  - >= 10 keywords SEO com volume
  - Zero afirmações sem tag [DADO]/[INFERENCIA]/[HIPOTESE]
- Gate decision: GO / CONCERNS / FAIL
- Bloqueia avanço para F2 se critérios BLOCK não passarem
- Exemplo:

```yaml
gate:
  id: research-completeness-gate
  name: "Research Completeness Gate"
  trigger: "F1.5 complete"

  checks:
    - id: RG-01
      name: "Scorecard completude"
      condition: "scorecard.overall >= 75%"
      severity: BLOCK
      message: "Research incompleto. Scorecard em {score}%. Minimo: 75%."

    - id: RG-02
      name: "Personas validadas"
      condition: "personas.count >= 2 AND personas.evidence_based == true"
      severity: BLOCK

    - id: RG-03
      name: "Pelo menos 3 competitors analisados"
      condition: "competitors.analyzed >= 3"
      severity: WARN

    - id: RG-04
      name: "SEO keywords com volume validado"
      condition: "seo.validated_keywords >= 10"
      severity: WARN

    - id: RG-05
      name: "Sem hipoteses nao marcadas"
      condition: "hypotheses.untagged == 0"
      severity: BLOCK

  decision:
    all_BLOCK_pass: GO
    any_BLOCK_fail: CONCERNS
    multiple_BLOCK_fail: FAIL
```

#### PRIORIDADE MEDIA

**ORC-S05 — Transition Triggers nos agentes**
- Adicionar seccao "Completion Output" ao final de CADA agente de research
- Formato padronizado:

```markdown
## Completion Output

Ao terminar, SEMPRE emitir:
---
### TRANSITION SUGGESTION
- **Status:** COMPLETE | PARTIAL | BLOCKED
- **Confidence:** 85%
- **Next recommended:** @seo-auditor (F1.5)
- **Reason:** "Dados de audiencia completos. SEO pode agora validar angulos."
- **Blockers:** Nenhum
- **Gaps identified:** [lista de gaps para Research Backlog]
---
```

**ORC-S01 — Reorganizar como Research Squad**
- Mover agentes de research para `orchestrator/squads/research/agents/`
- Criar `squad.yaml` com metadata
- Separar tasks, templates e data dentro do squad
- Permite instanciar o research squad por projecto com config inheritance

#### PRIORIDADE BAIXA (AGORA)

**ORC-S06 — Context Tiers no YAML central**
- Definir 3 tiers de dados: ESSENTIAL (~100 linhas), STANDARD (~300), DEEP (~700)
- Cada agente declara o tier que precisa
- Evita consumo desnecessario de context window
- Implementar quando tiver 4+ projectos activos

**ORC-S02 — Lentes de analise no Research Synthesizer**
- Adicionar 3 "lentes" ao synthesizer: customer-obsessed, contrarian, opportunity-hunter
- Forca analise multi-perspectiva em vez de voz unica neutra

### Padroes Tecnicos Adicionais a Adoptar

**Contratos formais de agentes (inputs/outputs YAML):**
- Cada agente deve ter seccao YAML com inputs/outputs declarados formalmente
- Tipo, origem, obrigatorio, validacao
- Permite detectar dependencias quebradas

**Estado de execucao persistente:**
- Criar `{projecto}/research/.research-state.yaml`
- Rastreia: phase actual, steps completados, gaps encontrados, backlog items
- Resolve o problema de "agentes nao sabem o estado uns dos outros"
- Exemplo:

```yaml
pipeline_instance: "bmh-research-20260305"
status: active
current_phase: F1.5
completed_steps:
  - P00: { status: done, date: 2026-02-28, agent: researcher-v2 }
  - P05: { status: done, date: 2026-02-28, agent: researcher-v2 }
pending_steps:
  - GATE-1: { status: pending, agent: micro-validator }
gaps_found: 3
backlog_items: 5
```

---

## O QUE JA EXISTE (LER PRIMEIRO)

### Ficheiros Obrigatorios (ler por esta ordem):

1. `orchestrator/agents/MELHORIA-PROCESSO-PESQUISA.md` — documento master de melhorias anteriores
2. `orchestrator/agents/researcher-v2.md` — agente principal
3. `orchestrator/agents/research-database-schema.md` — schema do YAML
4. `orchestrator/agents/seo-auditor.md` — agente SEO
5. `orchestrator/agents/research-synthesizer.md` — sintetizador
6. `orchestrator/agents/competitive-analyst.md` — competitivo
7. `orchestrator/agents/brand-architect.md` — marca
8. `orchestrator/agents/micro-validator.md` — validacao com trafego

### Ficheiros de Referencia (ler se necessario):

9. `adc/research/research-database.yaml` — YAML real produzido (~700 linhas)
10. `adc/DECISOES-ADC.md` — decisoes reais tomadas no projecto ADC

---

## O QUE FAZER NESTA SESSAO

### Fase 1: Ler e Compreender (NAO PULAR)

Ler todos os ficheiros listados acima. Compreender:
- Como cada agente funciona actualmente
- Como comunicam (ficheiros + YAML)
- Que gaps existem

### Fase 2: Implementar Melhorias de Alta Prioridade

Implementar **pela seguinte ordem**:

1. **ORC-S03** — Criar `orchestrator/workflows/research-pipeline.yaml`
   - Documentar todo o pipeline formal
   - Incluir complexity_tiers, execution_modes, transitions

2. **ORC-S04** — Criar `orchestrator/templates/research-backlog-tmpl.md`
   - Template reutilizavel para qualquer projecto
   - Com categorias GAPS, HYPOTHESES, ENHANCEMENTS
   - IDs contextuais

3. **ORC-S07** — Criar `orchestrator/gates/research-completeness-gate.yaml`
   - Criterios objectivos de passagem
   - GO/CONCERNS/FAIL

### Fase 3: Implementar Melhorias de Media Prioridade

4. **ORC-S05** — Adicionar "Completion Output" padronizado a cada agente de research
   - Editar: researcher-v2.md, seo-auditor.md, competitive-analyst.md,
     brand-architect.md, research-synthesizer.md, micro-validator.md
   - Adicionar seccao de transition suggestion ao final de cada um

### Fase 4: Propor (nao implementar) Baixa Prioridade

5. **ORC-S01, ORC-S06, ORC-S02** — Apenas documentar a proposta num ficheiro
   `orchestrator/proposals/ORC-S01-S02-S06-proposal.md`
   - Descrever o que seria feito
   - Estimar esforco
   - NAO implementar sem aprovacao

### Fase 5: Documentar Sessao

6. Criar `orchestrator/sessions/S01-ORC-RESEARCH-20260305.md`
   - O que foi feito
   - Decisoes tomadas
   - Ficheiros criados/modificados
   - Proximos passos

---

## REGRAS

- **NAO alterar ficheiros dentro de `adc/`, `bmh/`, `eft/`, `fmr/`** — o orchestrator e generico
- **NAO reescrever agentes inteiros** — adicionar, nao substituir
- **NAO fazer push para GitHub/VPS** sem instrucao
- **PROPOR antes de implementar** se algo nao estiver claro
- **Manter simplicidade** — o runtime e Claude Code + humano, nao ha infra de execucao
- **Filosofia incremental** — cada sessao melhora um pouco, nao reescreve tudo
- **Focar APENAS em research** — ignorar completamente fases posteriores

---

## DECISOES PENDENTES

| # | Decisao | Tipo | Status |
|---|---------|------|--------|
| ORC-D01 | Manter comunicacao via ficheiros (nao alterar para message queue) | Arquitectura | **DECIDIDO: MANTER** — ficheiros + YAML sao suficientes para o modelo actual |
| ORC-D02 | YAML centralizado: manter como ficheiro unico, adicionar context tiers | Data layer | **PARCIAL** — manter agora, tier quando tiver 4 projectos |
| ORC-D03 | Contratos de agentes: formalizar inputs/outputs em YAML | Governanca | **TODO nesta sessao** (ORC-S05) |
| ORC-D04 | Feedback loop: implementar via Research Backlog | Fluxo | **TODO nesta sessao** (ORC-S04) |
| ORC-D05 | Naming convention: manter `F1-P01-nome.md` e `F1.5-SYN-nome.md` | Organizacao | **DECIDIDO: MANTER** — esta funcional e claro |

---

## OUTPUT ESPERADO

1. `orchestrator/workflows/research-pipeline.yaml` — pipeline formal
2. `orchestrator/templates/research-backlog-tmpl.md` — template de backlog
3. `orchestrator/gates/research-completeness-gate.yaml` — quality gate
4. 6 agentes editados com "Completion Output" padronizado
5. `orchestrator/proposals/ORC-low-priority-proposals.md` — propostas futuras
6. `orchestrator/sessions/S01-ORC-RESEARCH-20260305.md` — documentacao da sessao

---

## CONTEXTO AIOS CORE — RESUMO DOS PADROES EXTRAIDOS

Os padroes abaixo foram extraidos directamente do codigo do AIOS Core
(github.com/SynkraAI/aios-core, ultimo commit: 03/03/2026).

### Workflow Pattern (spec-pipeline.yaml)
- Workflows YAML com `phases`, `sequence`, `execution_modes`
- 3 complexity tiers: SIMPLE (3 steps), STANDARD (6 steps), COMPLEX (8 steps com loop)
- Pre-flight checks antes de executar
- Cada step tem: agent, inputs, outputs, elicit, on_success, on_failure
- Gate behavior configuravel (strictGate: true = halt on BLOCKED)

### Agent Contract (task-v3-schema.json)
- Inputs tipados: campo, tipo, origem, obrigatorio, validacao
- Outputs tipados: campo, tipo, destino, persistido
- Pre-conditions (blocking) e post-conditions (validation)
- Execution modes: yolo (0-1 prompts), interactive (5-10), preflight (10-15)

### State Persistence (workflow-state-schema.yaml)
- Estado em `.aios/{instance-id}-state.yaml`
- Campos: workflow_id, status (active/paused/completed/aborted), current_phase, current_step_index
- Steps array com execution status per step
- Artifacts registry
- Decisions log

### Workflow Transitions (workflow-patterns.yaml)
- Transitions com confidence score (0.80-0.95)
- Greeting message + next_steps (commands sugeridos com prioridade)
- Trigger patterns: "{command} completed"

### Quality Gates (bob-surface-criteria.yaml)
- Criterios codificados com id, condition, action, message, severity, bypass
- Evaluation order definido (short-circuit)
- Bypass configuravel por criterio (YOLO mode vs critical)

### SYNAPSE Context Brackets
- FRESH (60-100%): lean injection
- MODERATE (40-60%): standard injection
- DEPLETED (25-40%): all + memory hints
- CRITICAL (0-25%): all + handoff warning
- Cada agente declara que camadas precisa

### Backlog Management
- 3 tipos: Follow-up (F), Tech Debt (T), Enhancement (E)
- IDs contextuais: [STORY-013-F1]
- Lifecycle: IDEA → TODO → IN PROGRESS → DONE → ARCHIVED
- Auto-archive apos 30 dias de DONE

---

*Handoff criado por Sessao AIOS Core — 05/03/2026.*
*Baseado em analise do repositorio github.com/SynkraAI/aios-core (ultima actualizacao: 04/03/2026).*
