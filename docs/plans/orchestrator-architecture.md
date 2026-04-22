# Plano: Arquitetura Orquestradora para o Backend NutriAI

## Context

O `/chat` hoje chama `nutritionAnalystAgent` diretamente, que usa `toolInjectorProcessor` para classificar intent e injetar tools por step. O problema: a ORDEM de chamada de tools continua não-determinística (o LLM decide), multi-intent falha, e o agente único mistura responsabilidades de conversa e orquestração.

**Objetivo:** Trocar o ponto de entrada do `/chat` por um `orchestratorAgent` que despacha para fluxos determinísticos. Duas fases — a primeira entregável rapidamente, a segunda adiciona subagents especializados.

---

## Fase 1 — Orquestrador Único (simples)

Substitui `nutritionAnalystAgent + toolInjectorProcessor` por um `orchestratorAgent` que tem todas as tools + workflows encapsulados. Sem subagents ainda.

### Arquivos a criar

**`workflows/log-meal.ts`** — workflow determinístico 2 steps:
- Step 1 `search-foods`: recebe lista de nomes, chama `searchFoodsByEmbedding` para cada item em paralelo via `Promise.all`, retorna mapa `nome → food_id + dados nutricionais`
- Step 2 `log-meal`: recebe `meal_type` + itens do step anterior, chama `logMeal` da catalog-client
- Input schema: `{ meal_type: enum, items: Array<{food_name, quantity_g}>, notes? }`
- Usa `extractAuthContext({ requestContext })` + `asyncContext.getStore()` — mesmo padrão de `create-meal-plan.ts`

**`tools/log-meal-workflow.ts`** — tool wrapper usando `withAuth()`:
- Executa `search-foods` e `log-meal` sequencialmente dentro da própria `execute` (não delega ao workflow object para evitar problema de `requestContext`)
- Chama funções da catalog-client diretamente: `searchFoodsByEmbedding` → `logMeal`
- Isso garante a ordem search → log sem depender do LLM para sequenciamento

**`tools/create-meal-plan-workflow.ts`** — tool wrapper do `createMealPlanWorkflow` existente:
- Executa `calculateMacros` → `createMealPlan` sequencialmente dentro da `execute`
- Substitui o `createMealPlanTool` atual (que só chamava o step 2, sem calcular macros)
- Torna o agente incapaz de criar plano sem calcular macros antes

**`prompts/orchestrator/base.md`** — instruções focadas em orquestração:
- Regras explícitas de sequência (ex: para criar plano, chamar `create_meal_plan` que já garante ordem)
- Regra de multi-intent: processar cada intent em sequência, consolidar resposta
- Regras invioláveis do prompt atual (sem diagnóstico médico, não perguntar dados que já estão no contexto)
- Seção sobre como usar cada tool/quando

**`agents/orchestrator.ts`** — agente principal:
- Sem `inputProcessors` (remove o `toolInjectorProcessor`)
- Todas as tools disponíveis sempre (sem injeção seletiva)
- Memória via `createNutritionMemory()` — mesma configuração do atual
- Tools: todas as existentes exceto `logMealTool` e `createMealPlanTool` individuais (substituídas pelos wrappers de workflow)

### Arquivos a modificar

**`utils/context-loader.ts`** — adicionar:
```typescript
export function loadOrchestratorInstructions(): string {
  return loadPrompt('orchestrator', 'base');
}
```

**`index.ts`** — mudanças mínimas:
1. Adicionar import do `orchestratorAgent`
2. Adicionar `orchestratorAgent` em `agents: { nutritionAnalystAgent, orchestratorAgent }`
3. Adicionar `logMealWorkflow` em `workflows: { createMealPlanWorkflow, logMealWorkflow }`
4. No handler `/chat`, trocar `mastra.getAgent("nutritionAnalystAgent")` → `mastra.getAgent("orchestrator")`
5. **`nutritionAnalystAgent` permanece intacto** — `/eval/run` usa-o diretamente via import

### O que NÃO muda
- `agents/nutrition-analyst.ts` — não modificado, `/eval/run` continua funcionando
- `lib/async-context.ts`, `lib/user-chat-queue.ts`, `lib/jwt-auth.ts` — sem alteração
- `config/memory.ts`, `config/storage.ts`, `config/env.ts` — sem alteração
- Injeção de perfil + progresso diário no `/chat` — sem alteração
- Padrão `withAuth()` em todas as tools — sem alteração

### Sequência de implementação (Fase 1)
1. `workflows/log-meal.ts`
2. `tools/log-meal-workflow.ts`
3. `tools/create-meal-plan-workflow.ts`
4. `prompts/orchestrator/base.md`
5. `utils/context-loader.ts` (adicionar função)
6. `agents/orchestrator.ts`
7. `index.ts` (trocar agente no `/chat`)

---

## Fase 2 — Subagents Especializados (completo)

Adiciona agentes stateless por domínio. O orquestrador passa a despachar para eles como tools.

O sistema se organiza em **3 pastas principais**:

- **`tools/`** — operações atômicas (uma chamada de API), building blocks usados por agents e workflows
- **`workflows/`** — sequências determinísticas (1+ steps), sempre chamadas via tool wrapper pelo orquestrador
- **`agents/`** — LLMs com prompt especializado, chamados via tool wrapper pelo orquestrador

O orquestrador **nunca chama uma tool atômica diretamente** — despacha sempre para um workflow-tool ou subagent-tool.

### Arquitetura alvo

```
/chat → orchestratorAgent (com memória)
          ├── [workflow-tool] log_meal           → logMealWorkflow (search → log)
          ├── [workflow-tool] create_meal_plan   → createMealPlanWorkflow (macros → create)
          ├── [workflow-tool] get_daily_summary  → dailySummaryWorkflow (1 step)
          ├── [workflow-tool] get_weekly_stats   → weeklyStatsWorkflow (1 step)
          ├── [workflow-tool] add_goal           → addGoalWorkflow (1 step)
          ├── [workflow-tool] add_activity       → addActivityWorkflow (1 step)
          ├── [subagent-tool] meal_logging       → MealLoggingAgent (stateless)
          ├── [subagent-tool] nutrition_info     → NutritionInfoAgent (stateless)
          ├── [subagent-tool] meal_planning      → MealPlanningAgent (stateless)
          └── [subagent-tool] profile            → ProfileAgent (stateless)
```

### Novos workflows de 1 step (operações simples)

Operações que eram tools diretas no orquestrador viram workflows para manter consistência arquitetural e ganhar tracing no playground:

**`workflows/get-daily-summary.ts`** — 1 step: chama `getDailySummary` da catalog-client

**`workflows/get-weekly-stats.ts`** — 1 step: chama `getWeeklyStats` da catalog-client

**`workflows/add-goal.ts`** — 1 step: chama API de metas

**`workflows/add-activity.ts`** — 1 step: chama API de atividades

Cada um usa `extractAuthContext` + `asyncContext` para autenticação, como os workflows existentes.

### Novos subagents

**`agents/meal-logging-agent.ts`**
- Tools: `logMealWorkflowTool`, `confirmAndLogImageMealTool`, `searchFoodCatalogTool`
- Sem `memory`
- Prompt: identificar alimentos/quantidades, confirmar, registrar

**`agents/nutrition-info-agent.ts`**
- Tools: `searchFoodCatalogTool`, `calculateNutritionTool`, `findSimilarFoodsTool`, `recommendationTool`, `searchRecipesTool`, `getRecipeTool`, `suggestRecipeTool`
- Sem `memory`
- Prompt: busca, cálculos, recomendações, receitas

**`agents/meal-planning-agent.ts`**
- Tools: `createMealPlanWorkflowTool`, `listMealPlansTool`, `getMealPlanTool`, `updateMealPlanTool`, `deleteMealPlanTool`, `exportMealPlanPdfTool`
- Sem `memory`
- Prompt: criar, listar, editar, exportar planos

**`agents/profile-agent.ts`**
- Tools: `updateUserProfileTool`, `calculateMacrosTool`
- Sem `memory`
- Prompt: atualizar dados, calcular metas

**`tools/agent-tools.ts`** — wrappers subagent-as-tool via `withAuth()`:
```typescript
// Padrão para cada subagent:
export const mealLoggingAgentTool = withAuth({
  id: "meal_logging",
  description: "Especialista em registrar refeições. Passe a mensagem completa do usuário.",
  inputSchema: z.object({ message: z.string() }),
  execute: async ({ message }) => {
    const result = await mealLoggingAgent.generate(message)
    return { result: result.text }
  }
})
```

**`tools/workflow-tools.ts`** — wrappers workflow-as-tool para os novos workflows simples:
```typescript
// Padrão para cada workflow de 1 step:
export const getDailySummaryWorkflowTool = withAuth({
  id: "get_daily_summary",
  description: "Retorna o resumo de calorias e macros consumidos hoje.",
  inputSchema: z.object({ date: z.string().optional() }),
  execute: async ({ date }, { authToken, userId }) => {
    // chama getDailySummary da catalog-client diretamente
    return await getDailySummary(userId, date ?? today(), undefined, authToken)
  }
})
```

> **Nota sobre propagação de contexto:** O `asyncContext.run()` do `/chat` envolve toda a call stack. O `asyncContext.getStore()` estará disponível dentro das tools, workflows e subagents sem nenhuma mudança adicional.

### Arquivos a modificar (Fase 2)

**`agents/orchestrator.ts`** — todas as tools são wrappers de workflow ou subagent, nunca tools atômicas:
```typescript
tools: {
  // subagent-tools (LLM especializado)
  meal_logging: mealLoggingAgentTool,
  nutrition_info: nutritionInfoAgentTool,
  meal_planning: mealPlanningAgentTool,
  profile: profileAgentTool,
  // workflow-tools (determinístico, 1 step)
  get_daily_summary: getDailySummaryWorkflowTool,
  get_weekly_stats: getWeeklyStatsWorkflowTool,
  add_goal: addGoalWorkflowTool,
  add_activity: addActivityWorkflowTool,
}
```

**`prompts/orchestrator/base.md`** — regras de roteamento:
```
- Registrar refeição (manual ou imagem) → meal_logging
- Busca, cálculos, receitas, recomendações → nutrition_info
- Plano alimentar → meal_planning
- Perfil, macros → profile
- Resumo diário → get_daily_summary
- Estatísticas da semana → get_weekly_stats
- Adicionar meta → add_goal
- Registrar atividade → add_activity
- Multi-intent: chamar em sequência, consolidar resposta
```

**`index.ts`** — registrar subagents em `agents: { ... }`

### Sequência de implementação (Fase 2)
1. `workflows/get-daily-summary.ts`
2. `workflows/get-weekly-stats.ts`
3. `workflows/add-goal.ts`
4. `workflows/add-activity.ts`
5. `agents/meal-logging-agent.ts`
6. `agents/nutrition-info-agent.ts`
7. `agents/meal-planning-agent.ts`
8. `agents/profile-agent.ts`
9. `tools/agent-tools.ts` (subagent wrappers)
10. `tools/workflow-tools.ts` (workflow wrappers para os 4 novos)
11. `agents/orchestrator.ts` (atualizar tools)
12. `prompts/orchestrator/base.md` (adicionar roteamento)
13. `index.ts` (registrar subagents + novos workflows)

---

## Riscos e Mitigações

| Risco | Mitigação |
|---|---|
| Ciclo de dependência ao importar `mastra` singleton dentro de tools | Não referenciar o singleton — chamar catalog-client diretamente nas tool wrappers |
| `asTool()` não disponível no Mastra 1.4.0 | Usar wrappers manuais com `withAuth` + `agent.generate()` |
| Context window maior com todas as tools no orquestrador | Monitorar via observability; reduzir descrições se necessário |
| `/eval/run` quebrar | `agents/nutrition-analyst.ts` não é modificado; import direto continua válido |

---

## Verificação

**Fase 1 — Casos de teste para `/chat`:**
- `"quanto é 100g de frango?"` → deve chamar `search_food_catalog`
- `"comi 200g de arroz no almoço"` → deve chamar `log_meal` (search interno → log garantidos em ordem)
- `"registra o almoço e me mostra o resumo"` → deve chamar `log_meal` + `get_daily_summary` em sequência
- `"cria um plano alimentar"` → deve chamar `create_meal_plan` (macros → criação, em ordem)
- `/eval/run` com `agent_mode: "production"` → deve continuar respondendo via `nutritionAnalystAgent`

**Fase 2 — Verificação adicional:**
- Confirmar que `asyncContext.getStore()` tem dados dentro da `execute` do subagent tool
- Testar multi-intent com subagents: resposta deve consolidar resultados dos dois agentes
