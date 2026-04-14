# Fórmula de Arquitetura: Skills + Agents

Referência para projetar sistemas de automação bem definidos com Claude Code.

---

## A regra fundamental

```
SKILL  = orquestra + decide + tem contexto da conversa
AGENT  = executa + é isolado + faz uma coisa só
```

Nunca misture: um arquivo não deve ao mesmo tempo decidir o que fazer E executar comandos de sistema.

---

## As quatro camadas

```
┌─────────────────────────────────────┐
│  CAMADA 1 — Usuário                 │
│  /slash-commands (Skills)           │
│  Disparadas pelo usuário via /nome  │
└──────────────┬──────────────────────┘
               │ spawna
┌──────────────▼──────────────────────┐
│  CAMADA 2 — Orquestração (Skills)   │
│  Lê planos, define ordem, paraleliza│
│  Não escreve código, não roda CLI   │
└──────────────┬──────────────────────┘
               │ spawna
┌──────────────▼──────────────────────┐
│  CAMADA 3 — Execução (Agents)       │
│  Implementa, escreve, roda comandos │
│  Contexto isolado, uma tarefa só    │
└──────────────┬──────────────────────┘
               │ consulta
┌──────────────▼──────────────────────┐
│  CAMADA 4 — Especialistas (Agents)  │
│  Auditam, validam, revisam          │
│  Só leitura, sem efeitos colaterais │
└─────────────────────────────────────┘
```

**Regra de direção:** camadas só invocam para baixo, nunca para cima.

---

## Árvore de decisão: skill ou agent?

```
O usuário dispara diretamente via /comando?
  SIM → Skill (com user-invocable: true)
  NÃO ↓

Precisa do histórico/contexto da conversa atual?
  SIM → Skill (sem user-invocable ou keyword-triggered)
  NÃO ↓

Faz exatamente uma coisa atômica e retorna resultado?
  SIM → Agent
  NÃO ↓

Roda em paralelo com outros workers do mesmo tipo?
  SIM → Agent
  NÃO ↓

Decide o que fazer com base em condições variáveis?
  SIM → Skill
  NÃO → Agent
```

---

## Frontmatter de Skill

```yaml
---
name: nome-do-workflow          # identificador único, kebab-case
description: >                  # quando triggar automaticamente (keywords)
  Descrição longa com palavras-chave que ativam a skill.
  Quanto mais específico, melhor o match automático.
user-invocable: true            # omitir se for keyword-triggered apenas
argument-hint: [o-que-passar]   # aparece no autocomplete do /comando
allowed-tools: Read, Write, Glob, Grep, Agent  # Agent obrigatório se spawna subagents
---
```

### Campos obrigatórios
- `name` — identificador, deve ser único entre skills e agents
- `description` — usado pelo Claude para match automático; inclua keywords relevantes

### Campos opcionais mas importantes
- `user-invocable: true` — sem isso, a skill só aciona por keyword, nunca por /comando
- `argument-hint` — texto de ajuda que aparece no autocomplete
- `allowed-tools` — sem isso, usa tudo disponível (ruim: viola princípio do menor privilégio)

---

## Frontmatter de Agent

```yaml
---
name: nome-do-especialista      # kebab-case, descreve o papel
description: quando usar + o que o agent faz  # usado no match automático pelo Agent tool
model: haiku | sonnet | opus    # escolha pelo nível de raciocínio necessário
allowed-tools: Read, Bash(cmd:*)  # restrinja ao mínimo necessário
---
```

### Escolha de modelo

| Modelo | Usar quando |
|---|---|
| `haiku` | Leitura, busca, auditoria simples, operações rápidas |
| `sonnet` | Escrita de arquivos, execução de comandos, raciocínio médio |
| `opus` | Decisões arquiteturais complexas, raciocínio profundo, código crítico |

### Restrição de Bash
Sempre restrinja comandos Bash ao mínimo:
```yaml
# Ruim — acesso total ao shell
allowed-tools: Read, Write, Bash

# Bom — só os comandos necessários
allowed-tools: Read, Write, Bash(npx prisma:*), Bash(pnpm type-check:*)
```

---

## Estrutura de diretórios

```
.claude/
├── agents/
│   ├── nome-executor.md        # worker que faz a tarefa
│   └── nome-especialista.md    # consultor que valida/audita
└── skills/
    └── nome-workflow/
        ├── SKILL.md             # obrigatório — o workflow em si
        └── references/          # opcional — documentação de suporte
            ├── patterns.md
            └── examples.md
```

### Quando criar `references/`
- A skill precisa de padrões codificados (ex: mapeamentos de tipos, convenções)
- O workflow tem muitos casos especiais para caber no SKILL.md
- Outros agents ou skills precisam dessa documentação também

---

## Paralelismo

Agents rodam em paralelo quando chamados no mesmo turno:

```
# Turno único → execução paralela
Agent(subagent_type: "auditor-A", prompt: "Audit module X")
Agent(subagent_type: "auditor-A", prompt: "Audit module Y")
Agent(subagent_type: "auditor-A", prompt: "Audit module Z")
```

Use paralelismo quando:
- As tasks são independentes (sem dependência de dados entre si)
- O mesmo tipo de agent processa partes diferentes do mesmo conjunto

Use sequencial quando:
- Task B precisa do output de Task A
- A ordem importa (ex: migrate antes de generate)

---

## Anti-patterns a evitar

| Anti-pattern | Problema | Solução |
|---|---|---|
| Agent que faz tudo | Sem foco, difícil de debugar | Um agent por responsabilidade |
| Skill que executa CLI | Mistura decisão com efeito colateral | Delegar execução a um agent |
| Agent sem `allowed-tools` | Acesso irrestrito ao sistema | Sempre declarar o mínimo necessário |
| Skill sem `allowed-tools` | Idem | Sempre declarar, incluir `Agent` se spawna |
| Agent com `model: opus` para tarefas simples | Custo desnecessário | Use `haiku` ou `sonnet` |
| Camada 3 invocando camada 1 | Ciclo de dependência | Fluxo só desce, nunca sobe |
| `description` vaga | Match automático falha | Inclua keywords específicas |

---

## Checklist antes de criar

**Para a skill:**
- [ ] O workflow está mapeado passo a passo?
- [ ] Sei quais agents ela vai invocar?
- [ ] As condicionais (se X então Y) estão documentadas?
- [ ] Os `allowed-tools` incluem `Agent` se spawna subagents?

**Para cada agent:**
- [ ] O agent faz exatamente uma coisa?
- [ ] O `allowed-tools` está no mínimo necessário?
- [ ] O modelo escolhido é proporcional à complexidade?
- [ ] A `description` é específica o suficiente para match automático?
- [ ] O formato de resposta está documentado no body?

**Para o conjunto:**
- [ ] Nenhuma camada invoca para cima?
- [ ] Os agents podem rodar em paralelo quando independentes?
- [ ] Há um agent de validação após execução?
