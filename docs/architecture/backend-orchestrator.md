# Arquitetura do Backend NutriAI — Orquestrador com Subagents

## Visão Geral

O backend é um agente conversacional Mastra organizado em **3 pastas principais** com responsabilidades claras:

```
mastra/
├── agents/      # LLMs com prompt especializado — raciocinam e decidem
├── workflows/   # Sequências determinísticas — executam em ordem garantida
└── tools/       # Operações atômicas — uma chamada de API por tool
```

**Regra fundamental:** O `orchestratorAgent` nunca chama uma tool atômica diretamente. Sempre despacha para um **workflow-tool** (determinístico) ou **subagent-tool** (LLM especializado). Isso garante que:
- Sequências de operações têm ordem garantida
- O orquestrador mantém a conversa; subagents e workflows executam
- Cada camada tem uma única responsabilidade

---

## Arquitetura de Fluxo

```
POST /chat
  │
  ▼
[JWT validation + profile loading + daily progress injection]
  │
  ▼
orchestratorAgent.stream(messages, { context })
  │
  ├── detecta intent da mensagem
  │
  ├── intent resolve com dado → workflow-tool (determinístico)
  │     └── ex: "como foi minha semana?" → weeklyStatsWorkflowTool
  │
  ├── intent requer raciocínio → subagent-tool (LLM especializado)
  │     └── ex: "busca uma receita de frango low carb" → nutritionInfoAgentTool
  │
  └── intent composto → compound workflow-tool (múltiplos steps)
        └── ex: "cria plano alimentar e adiciona meta de exercício" → createPlanWithActivityWorkflowTool
```

---

## Taxonomia de Intents

### Categoria 1 — Leitura de Dados

O workflow executa e retorna os dados disponíveis.

| Intent | Trigger (exemplos) | Dispatch |
|---|---|---|
| `daily_summary` | "como está meu dia?", "quanto comi hoje?" | `dailySummaryWorkflow` |
| `weekly_stats` | "resumo da semana", "minha evolução" | `weeklyStatsWorkflow` |
| `search_food` | "calorias do arroz", "valor nutricional do frango" | `NutritionInfoAgent` |
| `recommendation` | "o que devo comer?", "me sugere algo" | `NutritionInfoAgent` |
| `recipe` | "receita de omelete", "como fazer frango grelhado" | `NutritionInfoAgent` |

---

### Categoria 2 — Escrita Transacional

Operações que modificam dados. O workflow extrai os parâmetros necessários da mensagem, executa e retorna o estado resultante.

| Intent | Trigger (exemplos) | Dispatch | Steps |
|---|---|---|---|
| `log_meal` | "comi arroz e frango", "registra meu almoço" | `logMealWorkflow` | search → log |
| `add_activity` | "fiz 30min de caminhada", "treinei musculação" | `addActivityWorkflow` | registra atividade |
| `add_goal` | "quero chegar a 70kg", "meta de 120g de proteína" | `addGoalWorkflow` | salva meta |
| `update_profile` | "mudei meu peso para 78kg", "meu objetivo é emagrecer" | `ProfileAgent` | atualiza campo → recalcula macros se necessário |

---

### Categoria 3 — Operações Complexas

Workflows que dependem de dados do perfil do usuário (disponíveis via `asyncContext`). Se o perfil estiver incompleto, o workflow falha retornando todos os campos faltando de uma vez.

#### `create_meal_plan` — Criar Plano Alimentar

```
Trigger: "cria um plano alimentar", "quero uma dieta", "monta meu plano"

Workflow: createMealPlanWorkflow
  Step 1 — calculate-macros
    Input:  perfil do usuário (via asyncContext)
    Output: daily_calories, protein_g, carbs_g, fat_g, explanation
    Erro:   perfil incompleto → retorna lista de campos faltando

  Step 2 — create-plan
    Input:  macros do step 1 + plan_name
    Output: plan_id, plan_name, daily_calories, summary
```

#### `log_meal` — Registrar Refeição

```
Trigger: "comi X", "registra meu almoço", "tomei café"

Workflow: logMealWorkflow
  Step 1 — search-foods
    Input:  lista de nomes extraída da mensagem
    Output: { food_name → { food_id, calories, protein, carbs, fat } }
    Execução: Promise.all para múltiplos alimentos em paralelo

  Step 2 — log-meal
    Input:  meal_type + foods com IDs e quantidades do step 1
    Output: meal_id, total_calories, total_macros
```

---

### Categoria 4 — Intents Compostos

Workflows que executam múltiplas operações de domínios diferentes em uma única chamada determinística.

#### `create_plan_with_activity` — Plano Alimentar + Meta de Exercício

```
Trigger: "quero criar um plano alimentar e começar a me exercitar"
         "cria minha dieta e adiciona uma meta de treino"
         "plano alimentar + atividades físicas"

Workflow: createPlanWithActivityWorkflow
  Step 1 — calculate-macros
    Input:  perfil via asyncContext
    Output: daily_calories, macros, tdee

  Step 2 — create-plan
    Input:  macros do step 1
    Output: plan_id, plan_name

  Step 3 — add-activity-goal
    Input:  tipo de atividade extraído da mensagem (default: "treino")
            frequência extraída da mensagem (default: 3x/semana)
            estimativa de calorias queimadas baseada no TDEE do step 1
    Output: goal_id, activity_type, frequency
```

#### `log_meal_and_activity` — Registrar Refeição + Atividade

```
Trigger: "almocei e depois fui caminhar 30 minutos"
         "comi frango, registra e anota que treinei"

Workflow: logMealAndActivityWorkflow
  Step 1 — search-foods
  Step 2 — log-meal
  Step 3 — log-activity
```

---

## Filosofia de Design de Workflows

### Profile-Aware

Workflows leem o perfil diretamente do `asyncContext` — não dependem do LLM para repassar dados que já estão disponíveis.

```typescript
// Em qualquer step de workflow:
const userProfile = asyncContext.getStore()?.userProfile;
// peso, altura, idade, gênero, objetivo, nível de atividade já disponíveis
```

Se o perfil estiver **incompleto**, o workflow falha rápido com todos os campos faltando de uma vez — não valida campo a campo.

```typescript
const missing: string[] = [];
if (!weight_kg) missing.push("peso");
if (!activity_level) missing.push("nível de atividade");
if (missing.length > 0) throw new Error(`Campos faltando: ${missing.join(", ")}`);
```

### Compound Workflows para Intents Compostos

Quando um intent naturalmente envolve operações de domínios diferentes, o design preferido é um único workflow composto em vez de despachar para múltiplas ferramentas separadas.

```
Preferível:
  orchestrator → createPlanWithActivityWorkflowTool
                   step 1: calculate-macros
                   step 2: create-plan
                   step 3: add-activity-goal

Em vez de:
  orchestrator → createMealPlanWorkflowTool
  orchestrator → addGoalWorkflowTool
```

Isso mantém a execução determinística e os dados fluem entre steps (ex: TDEE calculado no step 1 é usado para estimar calorias no step 3).

---

## Estrutura de Arquivos Final

```
mastra/
├── agents/
│   ├── orchestrator.ts              # Entrypoint do /chat, tem memória
│   ├── meal-logging-agent.ts        # Especialista em log de refeições e imagens
│   ├── nutrition-info-agent.ts      # Busca, cálculos, receitas, recomendações
│   ├── meal-planning-agent.ts       # CRUD de planos alimentares
│   ├── profile-agent.ts             # Atualização de perfil e macros
│   ├── nutrition-analyst.ts         # Legado — mantido para /eval/run
│   └── eval-agent.ts                # Legado — mantido para /eval/run
│
├── workflows/
│   │
│   ├── # ── Simples (1 step) ──────────────────────────────
│   ├── get-daily-summary.ts
│   ├── get-weekly-stats.ts
│   ├── add-goal.ts
│   ├── add-activity.ts
│   │
│   ├── # ── Sequenciais (2 steps) ─────────────────────────
│   ├── log-meal.ts                  # search-foods → log-meal
│   ├── create-meal-plan.ts          # calculate-macros → create-plan [EXISTE]
│   │
│   └── # ── Compostos (3+ steps, multi-intent) ────────────
│       ├── create-plan-with-activity.ts  # macros → plan → activity-goal
│       └── log-meal-and-activity.ts      # search → log-meal → log-activity
│
└── tools/
    │
    ├── # ── Atômicas (building blocks) ────────────────────
    ├── search-food-catalog.ts
    ├── calculate-nutrition.ts
    ├── find-similar-foods.ts
    ├── recommendation.ts
    ├── log-meal.ts
    ├── confirm-and-log-image-meal.ts
    ├── get-daily-summary.ts
    ├── get-weekly-stats.ts
    ├── create-meal-plan.ts
    ├── list-meal-plans.ts
    ├── get-meal-plan.ts
    ├── update-meal-plan.ts
    ├── delete-meal-plan.ts
    ├── export-meal-plan-pdf.ts
    ├── calculate-macros.ts
    ├── update-user-profile.ts
    ├── search-recipes.ts
    ├── get-recipe.ts
    ├── suggest-recipe.ts
    ├── add-goal.ts
    ├── add-activity.ts
    │
    ├── # ── Wrappers de Workflow (usados pelo orchestrator) ─
    ├── workflow-tools.ts            # getDailySummaryWorkflowTool,
    │                                #   weeklyStatsWorkflowTool,
    │                                #   logMealWorkflowTool,
    │                                #   createMealPlanWorkflowTool,
    │                                #   createPlanWithActivityWorkflowTool,
    │                                #   logMealAndActivityWorkflowTool,
    │                                #   addGoalWorkflowTool,
    │                                #   addActivityWorkflowTool
    │
    └── # ── Wrappers de Subagent (usados pelo orchestrator) ─
        └── agent-tools.ts           # mealLoggingAgentTool,
                                     #   nutritionInfoAgentTool,
                                     #   mealPlanningAgentTool,
                                     #   profileAgentTool
```

---

## Tools do OrchestratorAgent

O orquestrador expõe **12 tools** — 8 workflow-tools e 4 subagent-tools. Nenhuma tool atômica.

```typescript
// agents/orchestrator.ts
tools: {
  // ── Workflow-tools (determinístico) ──────────────────────
  get_daily_summary:          getDailySummaryWorkflowTool,
  get_weekly_stats:           getWeeklyStatsWorkflowTool,
  log_meal:                   logMealWorkflowTool,
  add_goal:                   addGoalWorkflowTool,
  add_activity:               addActivityWorkflowTool,
  create_meal_plan:           createMealPlanWorkflowTool,
  create_plan_with_activity:  createPlanWithActivityWorkflowTool,
  log_meal_and_activity:      logMealAndActivityWorkflowTool,

  // ── Subagent-tools (LLM especializado) ───────────────────
  meal_logging:               mealLoggingAgentTool,
  nutrition_info:             nutritionInfoAgentTool,
  meal_planning:              mealPlanningAgentTool,
  profile:                    profileAgentTool,
}
```

---

## Regras de Roteamento do Orquestrador

```markdown
## Quando usar workflow-tool
- Resumo do dia → get_daily_summary
- Estatísticas da semana → get_weekly_stats
- Registrar refeição → log_meal
- Criar plano alimentar (sem atividade) → create_meal_plan
- Criar plano alimentar + atividade/exercício/treino → create_plan_with_activity
- Registrar refeição + atividade na mesma mensagem → log_meal_and_activity
- Adicionar meta → add_goal
- Registrar atividade física → add_activity

## Quando usar subagent-tool
- Busca de alimento, calorias, macros, valor nutricional → nutrition_info
- Receitas, ingredientes, modo de preparo → nutrition_info
- Recomendações e sugestões de alimentos → nutrition_info
- Análise de imagem de refeição → meal_logging
- Listar, visualizar, editar, exportar planos existentes → meal_planning
- Atualizar dados do perfil (peso, altura, objetivo, atividade) → profile

## Intents Compostos
Se a mensagem contém dois intents com compound workflow disponível → usar o compound workflow.
Se não há compound workflow → despachar para cada ferramenta em sequência.
```

---

## Propagação de Contexto

```
asyncContext.run({ userId, jwtToken, userProfile })  ← /chat handler
  │
  ├── orchestratorAgent.stream()
  │     └── asyncContext.getStore() ✓
  │
  ├── workflowTool.execute()
  │     └── asyncContext.getStore() ✓
  │         └── workflow step.execute()
  │               └── asyncContext.getStore() ✓
  │
  └── subagentTool.execute()
        └── asyncContext.getStore() ✓
            └── subAgent.generate()
                  └── tool.execute()
                        └── asyncContext.getStore() ✓
```

O `asyncContext.run()` envolve toda a call stack. Todos os níveis têm acesso a `userId`, `jwtToken` e `userProfile` sem nenhuma propagação manual adicional.

---

## Tabela de Responsabilidades

| Camada | Tem Memória | Chama LLM | Garante Ordem | Exemplo |
|---|---|---|---|---|
| `orchestratorAgent` | Sim | Sim (routing) | Não | Decide qual tool chamar |
| `subagents` | Não | Sim (especialista) | Não | `NutritionInfoAgent` busca e explica |
| `workflows compostos` | Não | Não | Sim (3+ steps) | `createPlanWithActivity` |
| `workflows simples` | Não | Não | Sim (1-2 steps) | `logMealWorkflow` |
| `tools atômicas` | Não | Não | N/A | `searchFoodCatalogTool` |

---

## Guia de Quando Criar Cada Tipo

| Situação | Criar |
|---|---|
| Nova operação de API (1 chamada) | Tool atômica em `tools/` |
| Sequência de 2+ operações com ordem obrigatória | Workflow em `workflows/` |
| Intenção que envolve raciocínio, linguagem natural, múltiplos caminhos | Subagent em `agents/` |
| Intent que combina operações de domínios diferentes | Compound workflow em `workflows/` |
| O orquestrador precisa chamar algo | Wrapper em `tools/workflow-tools.ts` ou `tools/agent-tools.ts` |
