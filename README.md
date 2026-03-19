# Configuração do Claude Code para o Projeto REDD

Este diretório contém toda a configuração para o Claude Code entender e trabalhar eficientemente com o projeto REDD.

## Estrutura

```
.claude/
├── CLAUDE.md              # Ponto de entrada principal
├── README.md              # Este arquivo
├── settings.local.json    # Permissões de skills
├── agents/                # Agents especializados
├── rules/                 # Regras de contexto
└── skills/                # Skills de geração de código
```

## Agents Disponíveis

| Agent                       | Uso                  | Quando Usar                                                 |
| --------------------------- | -------------------- | ----------------------------------------------------------- |
| `redd-backend-generator`    | Gera código backend  | Criar/modificar Models, Repositories, Controllers, Services |
| `redd-frontend-generator`   | Gera código frontend | Criar/modificar Pages, Providers, Components, Hooks         |
| `postgresql-query-analyzer` | Analisa queries SQL  | Otimizar queries, debugar N+1, segurança                    |
| `redd-feature-analyzer`     | Analisa features     | Entender como uma feature funciona, documentar              |
| `codigo-limpo-pt-br`        | Revisão de código    | Aplicar Clean Code, SOLID, boas práticas                    |

### Como usar um Agent

No Claude Code, simplesmente peça algo que corresponda à descrição do agent:

```
"Crie um CRUD completo para Fornecedores"
→ Usa automaticamente o redd-backend-generator

"Como funciona o módulo de Pedidos?"
→ Usa automaticamente o redd-feature-analyzer

"Revise este código para mim"
→ Usa automaticamente o codigo-limpo-pt-br
```

## Rules Disponíveis

### Sempre Aplicadas (`alwaysApply: true`)

| Rule                   | Descrição                               |
| ---------------------- | --------------------------------------- |
| `project-context.mdc`  | Contexto geral do projeto               |
| `global-standards.mdc` | Padrões globais (nomenclatura, commits) |
| `code-generation.mdc`  | Regras de geração de código             |

### Por Contexto

**Backend:**

- `backend-architecture.mdc` - Arquitetura em camadas
- `backend-models.mdc` - Padrões de Models
- `backend-repositories.mdc` - Padrões de Repositories
- `backend-services.mdc` - Padrões de Services
- `backend-controllers.mdc` - Padrões de Controllers
- `backend-validators.mdc` - Padrões de Validators
- `backend-routes.mdc` - Padrões de Routes

**Frontend:**

- `frontend-architecture.mdc` - Arquitetura de componentes
- `frontend-components.mdc` - Padrões de Components
- `frontend-hooks.mdc` - Padrões de Hooks
- `frontend-forms.mdc` - Padrões de Formulários
- `frontend-services.mdc` - Padrões de API Resources

## Skills Disponíveis

| Skill                        | Descrição                           |
| ---------------------------- | ----------------------------------- |
| `redd-backend-pattern`       | Templates completos de backend      |
| `redd-frontend-pattern`      | Templates completos de frontend     |
| `redd-integration-guide`     | Guia de integração backend↔frontend |
| `react-performance-patterns` | Otimização de React                 |
| `sql-optimization-patterns`  | Otimização de queries SQL           |

## Como Adicionar Novo Agent

1. Crie um arquivo em `.claude/agents/nome-do-agent.md`
2. Use o formato:

```markdown
---
name: nome-do-agent
description: "Descrição com exemplos de quando usar\\n\\n<example>\\nuser: \"pergunta\"\\nassistant: \"resposta\"\\n</example>"
model: sonnet
color: blue
---

# Título do Agent

Conteúdo com instruções, templates, checklists...
```

## Como Adicionar Nova Rule

1. Crie um arquivo em `.claude/rules/nome-da-rule.mdc`
2. Use o formato:

```markdown
---
description: Descrição da rule
alwaysApply: true # ou false para aplicar apenas quando relevante
---

# Título da Rule

Conteúdo com padrões, exemplos, checklists...
```

## Como Adicionar Nova Skill

1. Crie uma pasta em `.claude/skills/nome-da-skill/`
2. Crie o arquivo `SKILL.md` dentro dela
3. Opcionalmente crie `examples.md` com exemplos

## Integração com .cursor/rules

As rules em `.cursor/rules/` são **as mesmas** que em `.claude/rules/`. Isso permite que tanto o Cursor quanto o Claude Code usem as mesmas regras.

## Documentação Técnica

Para contexto adicional, consulte:

- `docs/technical/ARCHITECTURE.md` - Arquitetura detalhada
- `docs/technical/BACKEND.md` - Guia do backend
- `docs/technical/FRONTEND.md` - Guia do frontend
- `docs/features/` - Documentação de features específicas
